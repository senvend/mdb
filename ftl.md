# File Transport Layer (FTL) with SENVEND

This document describes how EVA-DTS files are transferred between the vending machine, refered to as **VMC** from now on, and the SENVEND terminal, using the **F**ile **T**ransport **L**ayer (**FTL**) defined in section 2.6 of the [MDB specification][1].

## Overview

FTL is not a peripheral of its own. It is a set of commands and responses that is supported by several MDB peripherals. The SENVEND terminal supports FTL via two peripherals:

1. The **Cashless Device** (cashless device one, see [Cashless](cashless.md))
1. The **Communications Gateway**

The FTL commands and responses, and therefore the transfer flows, are the same for both peripherals. Only the MDB address and the expansion command byte are different:

|Peripheral|MDB Address|Expansion Command Byte|Enable FTL via|
|:---|:---:|:---:|:---|
|Cashless Device|`10h`|`17h`|Option bit `b0` in `EXPANSION_ENABLE_OPTIONS` (`17h 04h`), Feature Level 3 required|
|Communications Gateway|`18h`|`1Fh`|Option bit `b0` in `EXPANSION_FEATURE_ENABLE` (`1Fh 01h`)|

The VMC must only use FTL with a peripheral that reported FTL as supported in its `PERIPHERAL_ID` response, and only after enabling the feature as shown above. You only need to support FTL via one of these peripherals. Make sure to **configure both, your VMC and the SENVEND terminal,** to use the same peripheral.

The examples below use the cashless device. For the Communications Gateway, replace address `10h` with `18h`, and command byte `17h` with `1Fh`. The POLL response codes (`1Bh` - `1Fh`) are the same for both peripherals.

## Use Cases

- **SENVEND terminal -> VMC: Setting prices.** The SENVEND terminal sends an EVA-DTS file with the new prices to the VMC (see [Setting Prices in the VMC](#setting-prices-in-the-vmc)). The VMC must support reading prices from an EVA-DTS file for this to work.
- **VMC -> SENVEND terminal: Audit data.** The VMC sends its EVA-DTS audit file (e.g. `ID1` with the machine serial number and `PA1` with the products) to the SENVEND terminal. The terminal takes the machine serial number (`ID101`) from it and uses it in the price updates.

## Setting Prices in the VMC

The prices are sent as an EVA-DTS file (File ID `01h`) with one `PC1` (product configuration) record per selection. Every product configured on the terminal with a price gets one `PC1` for each of its selections:

- `PC101`: MDB selection number of the product (the item number the VMC uses in `VEND_REQUEST` and reports in `PA101`, e.g. `11`)
- `PC102`: Price in the smallest currency unit (e.g. `150` for 1.50 EUR)

If the terminal knows the machine serial number from an audit file the VMC sent before, it is added as `ID101`. If no product has a price, no file is sent. Example file:

```
DXS*SEN0000000*VA*V0/6*1
ST*001*0001
ID1*ABC98765
PC1*11*150
PC1*12*200
G85*XXXX
SE*6*0001
DXE*1*1
```

Records are separated by `CR LF`. `G85` contains the CRC-16 of the transaction set and `SE01` the number of segments from `ST` to `SE`, as defined in the EVA-DTS standard.

The terminal offers the file with `FTL_REQ_TO_SEND` on a **POLL**, as soon as:

- **Cashless Device:** the initialization is complete, FTL is enabled and the reader is enabled (`READER_ENABLE`).
- **Communications Gateway:** FTL is enabled and the VMC sent `CONTROL_ENABLE` (or, if verbose mode is enabled, the `REPORT` with the currency ID).

If the VMC resets the terminal while a price update is in progress, the terminal offers the file again after the next initialization. If the price update is interrupted by a reset for the third time, it is dropped.

## FTL Basics

The values used in the examples below:

- Address of the VMC: `00h`
- Address of the SENVEND terminal: `10h` (cashless device) or `18h` (Communications Gateway)
- File ID: `01h` (DTS defined file, from the `DXS` record up to and including the `DXE` record)
- Control: `02h` (`b1` = End of File, i.e. this is the only FTL session for this file)
- Block numbers start at `00h` and count up. A block carries up to 31 data bytes.
- Retry Delay (in `FTL_RETRY_DENY`): `00h` means "Retry Never" and is also used to abort a running transfer. `ffh` means the request is not supported. Any other value is the number of seconds after which the sender may try again.

Responses of the terminal's own, like `FTL_REQ_TO_SEND`, are sent on a **POLL**.

**NOTE:** Every response of the SENVEND terminal (Cashless Device or Communications Gateway) to a command of the VMC may also be sent delayed: The terminal first answers the command with an **ACK** and sends the actual response as the answer to a later **POLL**. For example, an `FTL_OK_TO_SEND` of the VMC may be answered with an **ACK**, and `FTL_SEND_BLOCK` #0 follows on one of the next **POLL**s. This is allowed by the [MDB specification][1], so the VMC must support both the immediate and the delayed response. The examples below show the immediate response.

In both directions, the VMC must keep polling and servicing other peripherals during the transfer. The arguments of each command or response are shown in parentheses.

## Examples

### EVA-DTS Transfer SENVEND Terminal -> VMC

Files sent by the SENVEND terminal are laid out as follows:

- Every block carries 31 bytes. The last data block is padded with `00h` bytes.
- The transfer is terminated by a block of 31 `00h` bytes.
- The Length in `FTL_REQ_TO_SEND` is the number of data blocks **plus the terminating block**.
- The file must fit into a single FTL session (at most 254 data blocks).

The example sends the price update file from [Setting Prices in the VMC](#setting-prices-in-the-vmc) (without `ID1`), which fills exactly 3 blocks, so Length = `04h`.

|VMC|SENVEND Terminal|Action|
|:---:|:----------------:|:------------------------:|
|POLL|FTL_REQ_TO_SEND(Dest=`00h`, Src=`10h`, File ID=`01h`, Length=`04h`, Control=`02h`)|Terminal offers a file of 3 data blocks + terminating block|
|FTL_OK_TO_SEND(Dest=`10h`, Src=`00h`)|FTL_SEND_BLOCK(Dest=`00h`, Block=`00h`, Data=31 bytes)|VMC accepts the transfer, the first block is the response|
|POLL|FTL_SEND_BLOCK(Dest=`00h`, Block=`01h`, Data=31 bytes)|-|
|POLL|FTL_SEND_BLOCK(Dest=`00h`, Block=`02h`, Data=31 bytes)|Last data block (ends with the `DXE` record)|
|POLL|FTL_SEND_BLOCK(Dest=`00h`, Block=`03h`, Data=31 × `00h`)|Terminating block, transfer is complete|

Instead of a **POLL**, the VMC may also request each following block with another `FTL_OK_TO_SEND`. The terminal answers it with the next block in the same way.

If the VMC can't accept the file, it answers the `FTL_REQ_TO_SEND` with `FTL_RETRY_DENY(Dest=10h, Src=00h, Retry Delay)` instead of `FTL_OK_TO_SEND`. The terminal **ACK**s it and drops the offered file. The same applies, if the VMC sends `FTL_RETRY_DENY` during the transfer.

If the VMC sends `FTL_OK_TO_SEND` while the terminal has not offered a file (e.g. because the terminal restarted during a transfer), the terminal answers with an empty `FTL_SEND_BLOCK(Dest=00h, Block=00h, no data)`, which ends the session.

**NOTE:** The SENVEND terminal does not support `FTL_REQ_TO_RCV` from the VMC. The VMC can't request a file from the terminal, it is always offered by the terminal on its own initiative. An `FTL_REQ_TO_RCV` is answered with `FTL_RETRY_DENY(Dest=00h, Src=10h, Retry Delay=ffh)`.

### EVA-DTS Transfer VMC -> SENVEND Terminal

The VMC starts the transfer with `FTL_REQ_TO_SEND`. The terminal accepts every file. Every block sent by the VMC only gets an **ACK** from the terminal. This example transfers a file of 3 blocks.

|VMC|SENVEND Terminal|Action|
|:---:|:----------------:|:------------------------:|
|FTL_REQ_TO_SEND(Dest=`10h`, Src=`00h`, File ID=`01h`, Length=`03h`, Control=`02h`)|FTL_OK_TO_SEND(Dest=`00h`, Src=`10h`)|VMC offers a 3 block EVA-DTS file, the terminal accepts it|
|FTL_SEND_BLOCK(Dest=`10h`, Block=`00h`, Data=31 bytes)|ACK|First block (starts with the `DXS` record)|
|FTL_SEND_BLOCK(Dest=`10h`, Block=`01h`, Data=31 bytes)|ACK|-|
|FTL_SEND_BLOCK(Dest=`10h`, Block=`02h`, Data=1-31 bytes)|ACK|Last data block (ends with the `DXE` record)|
|FTL_SEND_BLOCK(Dest=`10h`, Block=`03h`, no data)|ACK|Empty block, transfer is complete|

As defined in the [MDB specification][1], a file of less than 256 blocks is terminated by an empty `FTL_SEND_BLOCK`. A terminating block of 31 `00h` bytes is accepted as well.

If a block is sent without a transfer in progress, or does not fit the transfer, the terminal aborts the transfer with `FTL_RETRY_DENY(Dest=00h, Src=10h, Retry Delay=00h)` instead of the **ACK**.

Files with File ID `00h` (Manufacturer ID information) are processed as EVA-DTS files as well, files with a manufacturer specific File ID (`f0h` - `ffh`) are ignored. A file that does not parse as an EVA-DTS file is dropped. A wrong `G85` CRC or `SE` segment count is tolerated.

#### Communications Gateway: Terminal Requests the File

When the VMC enables FTL via `EXPANSION_FEATURE_ENABLE`, the Communications Gateway asks the VMC for its EVA-DTS file with `FTL_REQ_TO_RCV` on the next **POLL**:

|VMC|SENVEND Terminal|Action|
|:---:|:----------------:|:------------------------:|
|POLL|FTL_REQ_TO_RCV(Dest=`00h`, Src=`18h`, File ID=`01h`, Max Length=`ffh`, Control=`02h`)|Terminal requests the EVA-DTS file from the VMC|
|FTL_SEND_BLOCK(Dest=`18h`, Block=`00h`, Data=31 bytes)|ACK|VMC approves by sending the first block|
|...|ACK|Repeated until all blocks are sent|
|FTL_SEND_BLOCK(Dest=`18h`, Block=`n`, no data)|ACK|Empty block, transfer is complete|

If the VMC can't provide the file, it answers with `FTL_RETRY_DENY(Dest=18h, Src=00h, Retry Delay)` instead of the first `FTL_SEND_BLOCK`.

### Raw MDB bytes

The examples below show every byte on the bus, including **POLL**s, **ACK**s and the checksum (`CHK`, the last byte of every message with data). As defined in the [MDB specification][1], the mode bit is set on the first byte (address byte) of every message from the VMC and on the last byte of every message from the peripheral (the `CHK`, or the `ACK` itself).

The terminal -> VMC examples transfer the price update file from [Setting Prices in the VMC](#setting-prices-in-the-vmc) (without `ID1`). The VMC -> terminal examples transfer this audit file of 91 bytes (3 blocks):

```
DXS*VND0000001*VA*V0/6*1
ST*001*0001
ID1*1234
PA1*11*150
G85*B5FB
SE*5*0001
DXE*1*1
```

#### Cashless Device

```
// ---- SENVEND Terminal -> VMC (price update) ----
// POLL
data from VMC: [12h, 12h]
// FTL REQ TO SEND (POLL response)
cashless to VMC: [1fh, 00h, 10h, 01h, 04h, 02h, 36h]
data from VMC: [00h]  // ACK
// FTL OK TO SEND
data from VMC: [17h, fdh, 10h, 00h, 24h]
// FTL SEND BLOCK #0: "DXS*SEN0000000*VA*V0/6*1\r\nST*00"
cashless to VMC: [1dh, 00h, 00h, 44h, 58h, 53h, 2ah, 53h, 45h, 4eh, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 2ah, 56h, 41h, 2ah, 56h, 30h, 2fh, 36h, 2ah, 31h, 0dh, 0ah, 53h, 54h, 2ah, 30h, 30h, e5h]
data from VMC: [00h]  // ACK
// POLL
data from VMC: [12h, 12h]
// FTL SEND BLOCK #1 (POLL response): "1*0001\r\nPC1*11*150\r\nPC1*12*200\r"
cashless to VMC: [1dh, 00h, 01h, 31h, 2ah, 30h, 30h, 30h, 31h, 0dh, 0ah, 50h, 43h, 31h, 2ah, 31h, 31h, 2ah, 31h, 35h, 30h, 0dh, 0ah, 50h, 43h, 31h, 2ah, 31h, 32h, 2ah, 32h, 30h, 30h, 0dh, 92h]
data from VMC: [00h]  // ACK
// POLL
data from VMC: [12h, 12h]
// FTL SEND BLOCK #2 (POLL response): "\nG85*2AB0\r\nSE*5*0001\r\nDXE*1*1\r\n"
cashless to VMC: [1dh, 00h, 02h, 0ah, 47h, 38h, 35h, 2ah, 32h, 41h, 42h, 30h, 0dh, 0ah, 53h, 45h, 2ah, 35h, 2ah, 30h, 30h, 30h, 31h, 0dh, 0ah, 44h, 58h, 45h, 2ah, 31h, 2ah, 31h, 0dh, 0ah, aah]
data from VMC: [00h]  // ACK
// POLL
data from VMC: [12h, 12h]
// FTL SEND BLOCK #3 (POLL response, terminating block of 31 zero bytes)
cashless to VMC: [1dh, 00h, 03h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 20h]
data from VMC: [00h]  // ACK

// ---- VMC -> SENVEND Terminal (audit data) ----
// FTL REQ TO SEND
data from VMC: [17h, feh, 10h, 00h, 01h, 03h, 02h, 2bh]
// FTL OK TO SEND
cashless to VMC: [1eh, 00h, 10h, 2eh]
data from VMC: [00h]  // ACK
// FTL SEND BLOCK #0: "DXS*VND0000001*VA*V0/6*1\r\nST*00"
data from VMC: [17h, fch, 10h, 00h, 44h, 58h, 53h, 2ah, 56h, 4eh, 44h, 30h, 30h, 30h, 30h, 30h, 30h, 31h, 2ah, 56h, 41h, 2ah, 56h, 30h, 2fh, 36h, 2ah, 31h, 0dh, 0ah, 53h, 54h, 2ah, 30h, 30h, eeh]
cashless to VMC: [00h]  // ACK
// FTL SEND BLOCK #1: "1*0001\r\nID1*1234\r\nPA1*11*150\r\nG"
data from VMC: [17h, fch, 10h, 01h, 31h, 2ah, 30h, 30h, 30h, 31h, 0dh, 0ah, 49h, 44h, 31h, 2ah, 31h, 32h, 33h, 34h, 0dh, 0ah, 50h, 41h, 31h, 2ah, 31h, 31h, 2ah, 31h, 35h, 30h, 0dh, 0ah, 47h, 8ch]
cashless to VMC: [00h]  // ACK
// FTL SEND BLOCK #2: "85*B5FB\r\nSE*5*0001\r\nDXE*1*1\r\n"
data from VMC: [17h, fch, 10h, 02h, 38h, 35h, 2ah, 42h, 35h, 46h, 42h, 0dh, 0ah, 53h, 45h, 2ah, 35h, 2ah, 30h, 30h, 30h, 31h, 0dh, 0ah, 44h, 58h, 45h, 2ah, 31h, 2ah, 31h, 0dh, 0ah, 79h]
cashless to VMC: [00h]  // ACK
// FTL SEND BLOCK #3 (empty, end of transfer)
data from VMC: [17h, fch, 10h, 03h, 26h]
cashless to VMC: [00h]  // ACK

// ---- Denied / aborted transfer ----
// FTL RETRY / DENY from VMC, Retry Never
data from VMC: [17h, fbh, 10h, 00h, 00h, 22h]
cashless to VMC: [00h]  // ACK
// FTL REQ TO RCV from VMC (not supported)
data from VMC: [17h, fah, 10h, 00h, 01h, ffh, 02h, 23h]
// FTL RETRY / DENY, Retry Delay = ffh (not supported)
cashless to VMC: [1ch, 00h, 10h, ffh, 2bh]
data from VMC: [00h]  // ACK
```

#### Communications Gateway

```
// ---- SENVEND Terminal -> VMC (price update) ----
// POLL
data from VMC: [1ah, 1ah]
// FTL REQ TO SEND (POLL response)
gateway to VMC: [1fh, 00h, 18h, 01h, 04h, 02h, 3eh]
data from VMC: [00h]  // ACK
// FTL OK TO SEND
data from VMC: [1fh, fdh, 18h, 00h, 34h]
// FTL SEND BLOCK #0: "DXS*SEN0000000*VA*V0/6*1\r\nST*00"
gateway to VMC: [1dh, 00h, 00h, 44h, 58h, 53h, 2ah, 53h, 45h, 4eh, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 2ah, 56h, 41h, 2ah, 56h, 30h, 2fh, 36h, 2ah, 31h, 0dh, 0ah, 53h, 54h, 2ah, 30h, 30h, e5h]
data from VMC: [00h]  // ACK
// POLL
data from VMC: [1ah, 1ah]
// FTL SEND BLOCK #1 (POLL response): "1*0001\r\nPC1*11*150\r\nPC1*12*200\r"
gateway to VMC: [1dh, 00h, 01h, 31h, 2ah, 30h, 30h, 30h, 31h, 0dh, 0ah, 50h, 43h, 31h, 2ah, 31h, 31h, 2ah, 31h, 35h, 30h, 0dh, 0ah, 50h, 43h, 31h, 2ah, 31h, 32h, 2ah, 32h, 30h, 30h, 0dh, 92h]
data from VMC: [00h]  // ACK
// POLL
data from VMC: [1ah, 1ah]
// FTL SEND BLOCK #2 (POLL response): "\nG85*2AB0\r\nSE*5*0001\r\nDXE*1*1\r\n"
gateway to VMC: [1dh, 00h, 02h, 0ah, 47h, 38h, 35h, 2ah, 32h, 41h, 42h, 30h, 0dh, 0ah, 53h, 45h, 2ah, 35h, 2ah, 30h, 30h, 30h, 31h, 0dh, 0ah, 44h, 58h, 45h, 2ah, 31h, 2ah, 31h, 0dh, 0ah, aah]
data from VMC: [00h]  // ACK
// POLL
data from VMC: [1ah, 1ah]
// FTL SEND BLOCK #3 (POLL response, terminating block of 31 zero bytes)
gateway to VMC: [1dh, 00h, 03h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 00h, 20h]
data from VMC: [00h]  // ACK

// ---- VMC -> SENVEND Terminal (audit data) ----
// FTL REQ TO SEND
data from VMC: [1fh, feh, 18h, 00h, 01h, 03h, 02h, 3bh]
// FTL OK TO SEND
gateway to VMC: [1eh, 00h, 18h, 36h]
data from VMC: [00h]  // ACK
// FTL SEND BLOCK #0: "DXS*VND0000001*VA*V0/6*1\r\nST*00"
data from VMC: [1fh, fch, 18h, 00h, 44h, 58h, 53h, 2ah, 56h, 4eh, 44h, 30h, 30h, 30h, 30h, 30h, 30h, 31h, 2ah, 56h, 41h, 2ah, 56h, 30h, 2fh, 36h, 2ah, 31h, 0dh, 0ah, 53h, 54h, 2ah, 30h, 30h, feh]
gateway to VMC: [00h]  // ACK
// FTL SEND BLOCK #1: "1*0001\r\nID1*1234\r\nPA1*11*150\r\nG"
data from VMC: [1fh, fch, 18h, 01h, 31h, 2ah, 30h, 30h, 30h, 31h, 0dh, 0ah, 49h, 44h, 31h, 2ah, 31h, 32h, 33h, 34h, 0dh, 0ah, 50h, 41h, 31h, 2ah, 31h, 31h, 2ah, 31h, 35h, 30h, 0dh, 0ah, 47h, 9ch]
gateway to VMC: [00h]  // ACK
// FTL SEND BLOCK #2: "85*B5FB\r\nSE*5*0001\r\nDXE*1*1\r\n"
data from VMC: [1fh, fch, 18h, 02h, 38h, 35h, 2ah, 42h, 35h, 46h, 42h, 0dh, 0ah, 53h, 45h, 2ah, 35h, 2ah, 30h, 30h, 30h, 31h, 0dh, 0ah, 44h, 58h, 45h, 2ah, 31h, 2ah, 31h, 0dh, 0ah, 89h]
gateway to VMC: [00h]  // ACK
// FTL SEND BLOCK #3 (empty, end of transfer)
data from VMC: [1fh, fch, 18h, 03h, 36h]
gateway to VMC: [00h]  // ACK

// ---- VMC -> SENVEND Terminal (requested by the terminal) ----
// POLL
data from VMC: [1ah, 1ah]
// FTL REQ TO RCV (POLL response)
gateway to VMC: [1bh, 00h, 18h, 01h, ffh, 02h, 35h]
data from VMC: [00h]  // ACK
// FTL SEND BLOCK #0: "DXS*VND0000001*VA*V0/6*1\r\nST*00"
data from VMC: [1fh, fch, 18h, 00h, 44h, 58h, 53h, 2ah, 56h, 4eh, 44h, 30h, 30h, 30h, 30h, 30h, 30h, 31h, 2ah, 56h, 41h, 2ah, 56h, 30h, 2fh, 36h, 2ah, 31h, 0dh, 0ah, 53h, 54h, 2ah, 30h, 30h, feh]
gateway to VMC: [00h]  // ACK
// FTL SEND BLOCK #1: "1*0001\r\nID1*1234\r\nPA1*11*150\r\nG"
data from VMC: [1fh, fch, 18h, 01h, 31h, 2ah, 30h, 30h, 30h, 31h, 0dh, 0ah, 49h, 44h, 31h, 2ah, 31h, 32h, 33h, 34h, 0dh, 0ah, 50h, 41h, 31h, 2ah, 31h, 31h, 2ah, 31h, 35h, 30h, 0dh, 0ah, 47h, 9ch]
gateway to VMC: [00h]  // ACK
// FTL SEND BLOCK #2: "85*B5FB\r\nSE*5*0001\r\nDXE*1*1\r\n"
data from VMC: [1fh, fch, 18h, 02h, 38h, 35h, 2ah, 42h, 35h, 46h, 42h, 0dh, 0ah, 53h, 45h, 2ah, 35h, 2ah, 30h, 30h, 30h, 31h, 0dh, 0ah, 44h, 58h, 45h, 2ah, 31h, 2ah, 31h, 0dh, 0ah, 89h]
gateway to VMC: [00h]  // ACK
// FTL SEND BLOCK #3 (empty, end of transfer)
data from VMC: [1fh, fch, 18h, 03h, 36h]
gateway to VMC: [00h]  // ACK
```

[1]: https://www.namanow.org/wp-content/uploads/Multi-Drop-Bus-and-Internal-Communication-Protocol.pdf
