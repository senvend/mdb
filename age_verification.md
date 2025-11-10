# Age Verification with SENVEND

In the following guide, you will find information on what is required from a vending machine, using the MDB protocol, to use SENVEND age verification features.

## MDB Age Verification

The [MDB specification][1] includes support for two different age verification methods. One involves a *dedicated* age verification device(68h), for example the DCM5. The other verifies the age via the cashless payment device(10h), i.e. the SENVEND terminal. Both the payment device and the vending machine need to support this (which SENVEND does). In the following, these two methods are referred to as:

1. '***Dedicated Age Verification***' (Using the dedicated age verification device)
1. '***Embedded Age Verification***' (Using the cashless payment device)

In addition the vending machine will be referred to as 'VMC' (**V**ending **M**achine **C**ontroller), just as in the [MDB specification][1].

### Mdb Age Verification Commands

For the process of age verification, the MDB specification defines two relevant commands:

- **D**iagnostic **R**equest **A**ge **V**erification On/Off (**DRAVP**)
- **D**iagnostic **R**equest **A**ge **V**erification **S**tatus (**DRAVS**)

These commands are used by the VMC to address the age verification device. In short, the **DRAVP** command is used to set the required age ("Customer must be 16/18/21/...!"), and the **DRAVS** command to ask for an age verification update (see examples below). For detailed information about these commands, please refer to the [MDB specification][1].

#### **IMPORTANT NOTICE**

The [MDB specification][1] defines two ways an age verification device can respond to a **DRAVP** command:

1. The device sends the response immediately after receiving the **DRAVP** command. It does not **ACK** the **DRAVP** command.
1. The device **ACK**s the **DRAVP** command, then waits with the response until it receives a **POLL** from the VMC.

For technical reasons, **the SENVEND terminal only supports the second method**. **The VMC must support this method** for the SENVEND age verification to work. More on this in the following.

## SENVEND Age Verification Support

While only one type of response is allowed, the SENVEND terminal supports both ***Dedicated Age Verification*** and ***Embedded Age Verification*** to ensure maximum compatibility. In the ***Dedicated*** mode, the SENVEND terminal will act as if it was two devices:

1. A dedicated age verification device
1. A cashless payment device (without age-verification capabilities)

The VMC will see it as such, and send messages for both types of device on the MDB bus. The SENVEND terminal will accept and handle both types of messages. In the ***Embedded*** mode, the VMC will see the terminal for what it actually is: As a cashless payment device with age verification capabilities.

**You only need to support one of these modes.** If your VMC supports it, the ***Embedded*** mode is probably the better choice. That said, it really doesn't matter too which one you support, as long as it is implemented properly. Make sure to **configure both, your VMC and the SENVEND terminal,** to use the right mode.

### Embedded Age Verification

When using the ***Embedded*** mode, the VMC should send the **DRAVP** command during the cashless initialization sequence, as well as every time the required age changes. The SENVEND terminal will use the provided age to perform age verification for the user.

As mentioned above, the ***DRAVS*** command is used to ask for an age verification update. This means that the VMC should send the **DRAVS** command every time an age restricted product is selected by a customer. The SENVEND terminal will then prompt the user to perform an age verification check. **It will be verified whether the customer is older than the last age supplied via the **DRAVP** command**.

### Examples

The following examples show how the age verification process should look like:

#### Cashless Initialization Sequence With Age Verification

The only difference to the cashless initialization sequence without age verification, is the **DRAVP** command.

|VMC|SENVEND Terminal|
|:---:|:----------------:|
|RESET|ACK|
|POLL|JUST_RESET|
|SETUP_CONFIG|ACK|
|POLL|READER_CONFIG_DATA|
|SETUP_MAX_MIN_PRICE|ACK|
|EXPANSION_REQUEST_ID|ACK|
|POLL|PERIPHERAL_ID|
|EXPANSION_ENABLE_OPTIONS|ACK|
|DRAVP|ACK|
|POLL|DRAVP|

The **DRAVP** command sequence may be sent at any other time during initialization.

#### Embedded Age Verification(Success)

|VMC|SENVEND Terminal|
|:---:|:----------------:|
|DRAVS|ACK|
|POLL|DRAVS|
|VEND_REQUEST|ACK|
|POLL|VEND_APPROVED|
|VEND_SUCCESS|ACK|

#### Embedded Age Verification(Failure)

|VMC|SENVEND Terminal|
|:---:|:----------------:|
|DRAVS|ACK|
|POLL|DRAVS|

#### Embedded Age Verification, Changing Age

|VMC|SENVEND Terminal |
|:----------:|:-----------:|
|DRAVP |ACK |
|POLL | DRAVP |
|DRAVS |ACK |
|POLL |DRAVS |
|VEND_REQUEST|ACK |
|POLL |VEND_APPROVED|
|VEND_SUCCESS|ACK |

### Dedicated Age Verification

The process for the ***Dedicated*** age verification method is identical to the ***Embedded*** age verification method. The only difference is that the VMC needs to sent its commands **to the age verification device (address `68h`)** and ***not*** to the cashless device (address `10h`).

For the initialization sequence of the **Age verification Device**, please refer to the [MDB specification][1].

### Additional Examples

#### Raw MDB bytes (without ACKs, POLLs, and message checksums)

```
// Reset
data from VMC: [10h]
// Setup config
data from VMC: [11h, 00h, 03h, 20h, 01h, 00h]
// Config Response
cashless to VMC: [01h, 03h, 19h, 78h, 05h, 02h, c8h, 09h]
// Setup max/min price
data from VMC: [11h, 01h, ffh, ffh, 00h, 00h]
// Dravp
data from VMC: [17h, ffh, 05h, 06h, 12h, 44h, 52h, 41h, 56h, 50h]
// Dravp response
Sending cashless to VMC: [ffh, 05h, 06h, 00h, 44h, 52h, 41h, 56h, 50h]
// Expansion request id
data from VMC: [17h, 00h, 54h, 56h, 44h, 30h, 30h, 32h, 31h, 30h, 32h, 36h, 31h, 32h, 30h, 33h, 35h, 31h, 2eh, 31h, 36h, 2eh, 31h, 32h, 6ah, 66h, 32h, 32h, 33h, 16h, 12h]
// Id response
Sending cashless to VMC: [09h, 56h, 46h, 49h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 30h, 05h, 00h, 00h, 00h, 00h, 20h]
// Expansion enable options
data from VMC: [17h, 04h, 00h, 00h, 00h, 20h]
// Reader enable
data from VMC: [14h, 01h]
// DRAVS
Got data from mdb: [17h, ffh, 06h, 06h, 00h, 44h, 52h, 41h, 56h, 53h]
// DRAVS response(age approved)
Sending cashless to mdb: [ffh, 06h, 07h, 9fh, 10h, 44h, 52h, 41h, 56h, 53h]
// Vend request
Got data from mdb: [13h, 00h, 00h, 3ch, 00, 0bh]
// Vend approved
Sending cashless to mdb: [05h, 00h, 3ch]
// Vend success
Got data from mdb: [13h, 02h, 00h, 0bh]
```

[1]: https://www.namanow.org/wp-content/uploads/Multi-Drop-Bus-and-Internal-Communication-Protocol.pdf
