# Cashless Payments with SENVEND

This document provides an overview of cashless payment integration using the SENVEND payment terminal with the MDB (Multi-Drop-Bus) protocol.

## Overview

The SENVEND payment terminal currently only supports the `cashless device one` as defined in the MDB specification.

## Manufacturer Code, Serial Number, Model Number and Software Version

The SENVEND terminal provides the following values during the initialization sequence to the vending machine:

- Manufacturer Code: `SEN`
- Serial Number: `xxx-xxx-xxx␠` (unique per device. Is equal to the S/N printed on the white label on the back of the terminal)
- Model Number: `UX700␠␠␠␠␠␠␠` (note the spaces to fill 12 characters)
- Software Version: Will contain the current software version formatted as described in the [MDB specification][1]

## Optional Features

If the SENVEND terminal is used in Feature Level 3 (will be used automatically if the VMC supports it), the following optional features (as defined as Optional Feature Bits in the [MDB specification][1] in the `PERIPHERAL_ID` response) are supported:

- Always Idle
- Basket/Partial Refund/Options Price
- 16-bit and 32-bit monetary format
- Remote Vend
- File Transport Layer (FTL), see [EVA-DTS File Transfer via FTL](#eva-dts-file-transfer-via-ftl)

Note: Not all features may be enabled by every SENVEND terminal. That depends on the terminal configuration. The terminal will only report the features as supported, that are actually enabled on the terminal. As described in the [MDB specification][1], the vending machine should never enable features in the `EXPANSION_ENABLE_OPTIONS` command, that were not reported as supported by the terminal in the `PERIPHERAL_ID` response.

## Remote Vend

The SENVEND terminal supports the Remote Vend feature (MDB 4.3), as described in the [MDB specification][1]. With Remote Vend the **terminal** starts a vend, instead of a customer pressing a selection button on the vending machine. This lets a vend be triggered without using the vending machine's user interface. The main use case is selling through other channels, such as Storefront. Operators can also trigger a vend manually on [my.senvend.com](https://my.senvend.com).

Your VMC must support Remote Vend as well.

### Enabling Remote Vend

Remote Vend is a Level 03 optional feature (Optional Feature Bit `b6`):

1. The terminal sets `b6` in the Optional Feature Bits of its `PERIPHERAL_ID` response if Remote Vend is enabled on the terminal.
1. The VMC enables the feature by setting `b6` in the `EXPANSION_ENABLE_OPTIONS` (`17h 04h`) command.

If the VMC does not enable `b6`, the terminal will never send a `SELECTION_REQUEST`.

### How A Remote Vend Works

A Remote Vend consists of the following steps:

1. **Selection Request** – The terminal answers a `POLL` with a `SELECTION_REQUEST` (`14h`) response. It contains, among others:
   - **Funds Available**: the maximum amount (scaled) that the terminal can credit for this vend. As defined in the [MDB specification][1], `FFFFh` (`FFFFFFFFh` in expanded currency mode) means undefined credit: the terminal pays the full price of the item. `0000h` (`00000000h`) means the VMC should dispense using cash or another credit source. The SENVEND terminal currently always sends `FFFFh` (`FFFFFFFFh`). This may change in the future, so your VMC must handle every value, as described in the next step.
   - **Item Number**: the item the VMC should dispense.
   - **Item Options**: additional options, e.g. for coffee machines. Usually `00000000h`.

   The VMC may receive a `SELECTION_REQUEST` in the _Enabled_ state or in the _Session Idle_ state. "Always Idle" does not need to be enabled for this.

1. **The VMC checks the selection** – The VMC checks whether it can dispense the requested item and whether _Funds Available_ covers the price. If the funds are not enough, the VMC may credit the full _Funds Available_ from the terminal and collect the rest in cash or from another credit source, as allowed by the [MDB specification][1]. Then it either:
   - sends a `VEND_REQUEST` (`13h 00h`) for **the same item number** and the price to be credited from the terminal (even if this price is 0), or
   - sends a `SELECTION_DENIED` (`13h 07h`) with the item number and a reason code (see below). The terminal stays in the state it was in before the `SELECTION_REQUEST`.
1. **Vend Request** – The `VEND_REQUEST` is handled exactly like a regular vend request. A `VEND_REQUEST` sent from the _Enabled_ state moves the terminal directly to the _Vend_ state, just like in "Always Idle" mode. The terminal answers with `VEND_APPROVED` (`05h`) or `VEND_DENIED` (`06h`).
1. **Dispensing** – After a `VEND_APPROVED` the VMC dispenses the product and reports `VEND_SUCCESS` (`13h 02h`) or `VEND_FAILURE` (`13h 03h`).

   After a `VEND_FAILURE` the payment is refunded by the SENVEND backend. As required by the [MDB specification][1], the VMC must `POLL` at least once before it sends `SESSION_COMPLETE`. The answer shows the state of the refund:
   - `ACK`: the refund is complete.
   - `MALFUNCTION_ERROR` (`0Ah`) with error code `1100yyyy`: the refund failed.
   - No response: the refund is still in progress. The VMC keeps polling until it gets one of the answers above, for up to the maximum non-response time.

   The SENVEND terminal currently does not report refund errors. This may change in the future, so your VMC must handle all three cases.

1. **Session Complete** – If the `SELECTION_REQUEST` was received in the _Enabled_ state, the terminal is now in the _Session Idle_ state. As after any regular vend, the VMC ends the session with `SESSION_COMPLETE` (`13h 04h`), and the terminal answers with `END_SESSION` (`07h`) and returns to the _Enabled_ state. The same applies after a `VEND_DENIED`.

#### **IMPORTANT NOTICE**

The item number in the `VEND_REQUEST` **must match** the item number in the `SELECTION_REQUEST`. The [MDB specification][1] requires this. If the item numbers differ, the SENVEND terminal rejects the `VEND_REQUEST` with `VEND_DENIED`. This prevents the wrong item from being vended by accident.

This means your VMC must use the same item numbers in `VEND_REQUEST`s for regular (customer-initiated) vends and for Remote Vends.

Note: If your machine requires these numbers to be different, feel free to contact us. We will consider adding an exception for your device, although we highly recommend following the spec.

#### Selection Denied Reasons

If the VMC cannot dispense the requested item, it should answer with `SELECTION_DENIED` and one of the reason codes defined in the [MDB specification][1].

Please use the most specific reason code available. It helps to show a meaningful error to the customer.

Reason `FFh` (machine temporarily busy) is not a final denial. As allowed by the [MDB specification][1], the terminal repeats the `SELECTION_REQUEST` after 2 seconds. It tries up to 3 times in total. If the VMC still answers with `FFh`, the Remote Vend fails. Use `FFh` only for short-lived conditions, e.g. while the delivery door is still open.

### Remote Vend Examples

#### Remote Vend (Success)

|                VMC                |            SENVEND Terminal            |               Action               |
| :-------------------------------: | :------------------------------------: | :--------------------------------: |
|              `POLL`               | `SELECTION_REQUEST` (item number = 11) |    A Remote Vend was triggered     |
| `VEND_REQUEST` (item number = 11) |                 `ACK`                  |   VMC requests the selected item   |
|              `POLL`               |            `VEND_APPROVED`             |     VMC dispenses the product      |
| `VEND_SUCCESS` (item number = 11) |                 `ACK`                  | Product was dispensed successfully |
|        `SESSION_COMPLETE`         |                 `ACK`                  |       VMC closes the session       |
|              `POLL`               |             `END_SESSION`              |   Terminal returns to _Enabled_    |

#### Remote Vend (Selection Denied)

|                          VMC                          |            SENVEND Terminal            |                   Action                   |
| :---------------------------------------------------: | :------------------------------------: | :----------------------------------------: |
|                        `POLL`                         | `SELECTION_REQUEST` (item number = 11) |        A Remote Vend was triggered         |
| `SELECTION_DENIED` (item number = 11, reason = `02h`) |                 `ACK`                  | Item is empty, terminal stays in _Enabled_ |

#### Remote Vend (Vend Failure)

|                VMC                |            SENVEND Terminal            |                         Action                          |
| :-------------------------------: | :------------------------------------: | :-----------------------------------------------------: |
|              `POLL`               | `SELECTION_REQUEST` (item number = 11) |               A Remote Vend was triggered               |
| `VEND_REQUEST` (item number = 11) |                 `ACK`                  |             VMC requests the selected item              |
|              `POLL`               |            `VEND_APPROVED`             |            VMC tries to dispense the product            |
|          `VEND_FAILURE`           |                 `ACK`                  | Product could not be dispensed, the payment is refunded |
|              `POLL`               |                 `ACK`                  |                     Refund complete                     |
|        `SESSION_COMPLETE`         |                 `ACK`                  |                 VMC closes the session                  |
|              `POLL`               |             `END_SESSION`              |              Terminal returns to _Enabled_              |

#### Remote Vend With Mismatching Item Number

|                VMC                |            SENVEND Terminal            |                       Action                       |
| :-------------------------------: | :------------------------------------: | :------------------------------------------------: |
|              `POLL`               | `SELECTION_REQUEST` (item number = 11) |            A Remote Vend was triggered             |
| `VEND_REQUEST` (item number = 12) |                 `ACK`                  | Item number does not match the `SELECTION_REQUEST` |
|              `POLL`               |             `VEND_DENIED`              |             Nothing must be dispensed              |
|        `SESSION_COMPLETE`         |                 `ACK`                  |               VMC closes the session               |
|              `POLL`               |             `END_SESSION`              |           Terminal returns to _Enabled_            |

#### Raw MDB Bytes

The examples below use the reader configuration from the [age verification examples](age_verification.md#raw-mdb-bytes-without-acks-polls-and-message-checksums): currency EUR (`1978h`), scale factor `05h`, 2 decimal places. One scaled unit is 0.05 €, so a price of 1.50 € is sent as `1Eh` (30 × 0.05 €). The values are examples. Payment media ID, payment type, and so on depend on the actual transaction.

The examples assume the terminal is Cashless Device #1 (address `10h`), so `POLL` is `12h`. `ACK` is `00h`. The last byte of every message is its checksum (`CHK`): the sum of all preceding bytes of the message, modulo 256. A single `ACK` byte carries no checksum. The terminal answers a `POLL` with `ACK` when it has nothing to report; those idle `POLL`s are left out.

**Remote Vend success (16-bit monetary format)**, item number 11 (`000Bh`), price 1.50 €:

```
// EXPANSION_ENABLE_OPTIONS (Remote Vend, b6 = 40h)
data from VMC: [17h, 04h, 00h, 00h, 00h, 40h, 5bh]
// ACK
cashless to VMC: [00h]
// READER_ENABLE
data from VMC: [14h, 01h, 15h]
// ACK
cashless to VMC: [00h]
// POLL
data from VMC: [12h, 12h]
// SELECTION_REQUEST
//   funds available = FFFFh (undefined), payment media ID = FFFFFFFFh (unknown),
//   payment type = 00h, payment data = 0000h, item number = 000Bh, item options = 00000000h
cashless to VMC: [14h, ffh, ffh, ffh, ffh, ffh, ffh, 00h, 00h, 00h, 00h, 0bh, 00h, 00h, 00h, 00h, 19h]
// ACK
data from VMC: [00h]
// VEND_REQUEST (price = 001Eh, item number = 000Bh)
data from VMC: [13h, 00h, 00h, 1eh, 00h, 0bh, 3ch]
// ACK
cashless to VMC: [00h]
// POLL
data from VMC: [12h, 12h]
// VEND_APPROVED (amount = 001Eh)
cashless to VMC: [05h, 00h, 1eh, 23h]
// ACK
data from VMC: [00h]
// VEND_SUCCESS (item number = 000Bh)
data from VMC: [13h, 02h, 00h, 0bh, 20h]
// ACK
cashless to VMC: [00h]
// SESSION_COMPLETE
data from VMC: [13h, 04h, 17h]
// ACK
cashless to VMC: [00h]
// POLL
data from VMC: [12h, 12h]
// END_SESSION
cashless to VMC: [07h, 07h]
// ACK
data from VMC: [00h]
```

**Selection denied**, item number 11 (`000Bh`) is empty:

```
// POLL
data from VMC: [12h, 12h]
// SELECTION_REQUEST (item number = 000Bh)
cashless to VMC: [14h, ffh, ffh, ffh, ffh, ffh, ffh, 00h, 00h, 00h, 00h, 0bh, 00h, 00h, 00h, 00h, 19h]
// ACK
data from VMC: [00h]
// SELECTION_DENIED (item number = 000Bh, reason = 02h: selection empty)
data from VMC: [13h, 07h, 00h, 0bh, 02h, 27h]
// ACK
cashless to VMC: [00h]
```

**Mismatching item number**, the terminal requests item 11 (`000Bh`) but the VMC requests item 12 (`000Ch`):

```
// POLL
data from VMC: [12h, 12h]
// SELECTION_REQUEST (item number = 000Bh)
cashless to VMC: [14h, ffh, ffh, ffh, ffh, ffh, ffh, 00h, 00h, 00h, 00h, 0bh, 00h, 00h, 00h, 00h, 19h]
// ACK
data from VMC: [00h]
// VEND_REQUEST (price = 001Eh, item number = 000Ch)
data from VMC: [13h, 00h, 00h, 1eh, 00h, 0ch, 3dh]
// ACK
cashless to VMC: [00h]
// POLL
data from VMC: [12h, 12h]
// VEND_DENIED
cashless to VMC: [06h, 06h]
// ACK
data from VMC: [00h]
// SESSION_COMPLETE
data from VMC: [13h, 04h, 17h]
// ACK
cashless to VMC: [00h]
// POLL
data from VMC: [12h, 12h]
// END_SESSION
cashless to VMC: [07h, 07h]
// ACK
data from VMC: [00h]
```

**Remote Vend success (32-bit monetary format / expanded currency mode)**, item number 11 (`000Bh`), price 1.50 €. Enabling `b1` alone already switches the terminal to _expanded currency mode_ (`b2` is not needed). In this mode the `SELECTION_REQUEST` also contains user language, user currency code, and card options, and the amounts in `VEND_REQUEST` and `VEND_APPROVED` are 32-bit. As required by the [MDB specification][1], the VMC sends `SETUP_MAX_MIN_PRICE` in the expanded format after enabling `b1`:

```
// EXPANSION_ENABLE_OPTIONS (32-bit monetary format b1 = 02h, Remote Vend b6 = 40h)
data from VMC: [17h, 04h, 00h, 00h, 00h, 42h, 5dh]
// ACK
cashless to VMC: [00h]
// SETUP_MAX_MIN_PRICE (max price = 000003E8h, min price = 00000001h, currency = 1978h)
data from VMC: [11h, 01h, 00h, 00h, 03h, e8h, 00h, 00h, 00h, 01h, 19h, 78h, 8fh]
// ACK
cashless to VMC: [00h]
// READER_ENABLE
data from VMC: [14h, 01h, 15h]
// ACK
cashless to VMC: [00h]
// POLL
data from VMC: [12h, 12h]
// SELECTION_REQUEST
//   funds available = FFFFFFFFh (undefined), payment media ID = FFFFFFFFh, payment type = 00h,
//   payment data = 0000h, user language = 0000h, user currency = 1978h (EUR),
//   card options = 00h, item number = 000Bh, item options = 00000000h
cashless to VMC: [14h, ffh, ffh, ffh, ffh, ffh, ffh, ffh, ffh, 00h, 00h, 00h, 00h, 00h, 19h, 78h, 00h, 00h, 0bh, 00h, 00h, 00h, 00h, a8h]
// ACK
data from VMC: [00h]
// VEND_REQUEST (price = 0000001Eh, item number = 000Bh)
data from VMC: [13h, 00h, 00h, 00h, 00h, 1eh, 00h, 0bh, 3ch]
// ACK
cashless to VMC: [00h]
// POLL
data from VMC: [12h, 12h]
// VEND_APPROVED (amount = 0000001Eh)
cashless to VMC: [05h, 00h, 00h, 00h, 1eh, 23h]
// ACK
data from VMC: [00h]
// VEND_SUCCESS (item number = 000Bh)
data from VMC: [13h, 02h, 00h, 0bh, 20h]
// ACK
cashless to VMC: [00h]
// SESSION_COMPLETE
data from VMC: [13h, 04h, 17h]
// ACK
cashless to VMC: [00h]
// POLL
data from VMC: [12h, 12h]
// END_SESSION
cashless to VMC: [07h, 07h]
// ACK
data from VMC: [00h]
```

## EVA-DTS File Transfer via FTL

The cashless device can also be used to transfer EVA-DTS files via the MDB File Transport Layer. This is described in [File Transport Layer (FTL)](ftl.md).

[1]: https://www.namanow.org/wp-content/uploads/Multi-Drop-Bus-and-Internal-Communication-Protocol.pdf
