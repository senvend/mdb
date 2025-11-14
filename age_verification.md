# Age Verification with SENVEND

In the following guide, you will find information on what is required from a vending machine, refered to as **VMC** from now on, to use the MDB age verification features with the SENVEND terminal

## MDB Age Verification

The [MDB specification][1] details two different age verification methods. One involves a *dedicated* age verification device(MDB addr. 68h), like for example the DCM5 ID scanner. The other is for verifying the age via the cashless payment device(MDB addr. 10h), i.e. the SENVEND terminal. Both the payment device and the vending machine need to support this (which SENVEND does). In the following, these two methods are referred to as:

1. '***Dedicated Age Verification***' (Using the dedicated age verification device)
1. '***Embedded Age Verification***' (Using the cashless payment device)

The ***Embedded Age Verification*** method is enabled by default on the SENVEND terminal. If you want to use the ***Dedicated Age Verification*** method instead, you must enable the **MDB Dediziertes Gerät** option in the terminal settings on [my.senvend.com](https://my.senvend.com).

### Mdb Age Verification Commands

For the process of age verification, the MDB specification defines two relevant commands:

- **D**iagnostic **R**equest **A**ge **V**erification On/Off (**DRAVP**)
- **D**iagnostic **R**equest **A**ge **V**erification **S**tatus (**DRAVS**)

These commands are used by the VMC to address the age verification device. In short, the **DRAVP** command is used to set the required age ("Customer must be 16/18/21/...!"), and the **DRAVS** command to ask for an age verification update (see examples below). For detailed information about these commands, please refer to the [MDB specification][1].

#### **IMPORTANT NOTICE**

The [MDB specification][1] defines two ways an age verification device can respond to a **DRAVP** command:

1. The age verification device sends the response immediately after receiving the **DRAVP** command. It does not **ACK** the **DRAVP** command.
1. The age verification device **ACK**s the **DRAVP** command and then waits for a **POLL** from the VMC. No response is sent before the **POLL**. The response might not be sent on the first **POLL**, depending on the time that passes between the **DRAVP** and the **POLL** command.

For technical reasons, **the SENVEND terminal only supports the second method**. **The VMC must support this method** for the SENVEND age verification to work. More on this in the following.

## SENVEND Age Verification Support

While only one type of response to the **DRAVP** command is allowed, the SENVEND terminal supports both ***Dedicated Age Verification*** and ***Embedded Age Verification***. In the ***Dedicated*** mode, the SENVEND terminal will act as if it was two devices:

1. A dedicated age verification device
1. A cashless payment device (without age-verification capabilities)

The VMC will see it as such, and send messages for both types of device on the MDB bus. The SENVEND terminal will accept and handle both types of messages.

In the ***Embedded*** mode, the VMC will see the terminal for what it actually is: As a cashless payment device with age verification capabilities. The VMC will address the cashless payment device only. Functionally, there is no difference between these two modes. Both are supported by SENVEND solely for compatibility reasons.


**You only need to support one of these modes.** If your VMC supports it, the ***Embedded*** mode is probably the better choice. That said, it really doesn't matter too much which one you support, as long as it is implemented properly. Make sure to **configure both, your VMC and the SENVEND terminal,** to use the right mode.

### Embedded Age Verification

When using the ***Embedded*** mode, the VMC should send the **DRAVP** command during the cashless initialization sequence, as well as every time the required age changes. The SENVEND terminal will use the provided age to perform age verification for the user.

As mentioned above, the ***DRAVS*** command is used to ask for an age verification update. This means that the VMC should send the **DRAVS** command every time an age restricted product is selected by a customer. The SENVEND terminal will then prompt the user to perform an age verification check. More precisely, **it will verify whether the customer is older than the age specified in the last **DRAVP** command**.

An Example of the age verification prompt can be found in the [screenshot section](#age-verification-prompt) below.

## Additional Functionality

### Aborting A Requested Age Verification

When the VMC has sent a **DRAVS** command to request an age verification, the SENVEND terminal will wait and show a prompt to the user, until the age verification is completed (either successfully or unsuccessfully). In some cases, the VMC might want to abort this process, for example when the user has changed their mind and selected a different product that does not require age verification.

The SENVEND terminal supports 5 ways to abort a requested age verification:

1. The VMC can send a **VendRequest** command, which will cause the SENVEND terminal to switch into payment mode. [Example flow](#aborting-a-requested-age-verificationvend-request)
1. The VMC can send a **DRAVP** command with age set to `00h` (Age Off). This will cause the SENVEND terminal to abort any ongoing age verification and switch back to its idle state. [Example flow](#aborting-a-requested-age-verificationage-off)
1. The VMC can send a **DRAVP** command with age set to `ffh` (Age SupportedWillBeSwitchedOn). This will also cause the SENVEND terminal to abort the ongoing process and switch back to its idle state. [Example flow](#aborting-a-requested-age-verificationage-supportedwillbeswitchedon)
1. If the VMC resets the SENVEND terminal, by sending a **RESET** command, any ongoing age verification process will be aborted.
1. The user can press the back button ([a little arrow at the top left of the screen][1]) on the age verification prompt screen to cancel the age verification process. This will switch the terminal back to its idle state. This will cause the SENVEND terminal to send a **DRAVS** response to the VMC, indicating that the user is not allowed to buy the age restricted product. [Example flow](#aborting-a-requested-age-verificationuser-presses-back-button)

**NOTE:** If an age verification process was aborted by options 2 or 3, it is required that the VMC sends an additional **DRAVP** command with the new age, before sending another **DRAVS** command. Sending **DRAVP** with age set to either `00h` or `ffh` will reset the previously provided age limit. Additionally note, that the 5th option (RESET) is only a backup, in case something goes wrong. You **should not** use a **RESET** command by design to abort an age verification process.

## Examples

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

|VMC|SENVEND Terminal|Action|
|:---:|:----------------:|:------------------------:|
|DRAVS|ACK|User is prompted for age verification with age limit set by DRAVP|
|POLL|DRAVS|-|
|VEND_REQUEST|ACK|Terminal switches to payment mode|
|POLL|VEND_APPROVED|-|
|VEND_SUCCESS|ACK|-|


#### Embedded Age Verification(Failure)

|VMC|SENVEND Terminal|Action|
|:---:|:----------------:|:------------------------:|
|DRAVS|ACK|User is prompted for age verification with age limit set by DRAVP|
|POLL|DRAVS|Terminal switches back to idle state|


#### Embedded Age Verification, Changing Age

|VMC|SENVEND Terminal |Action|
|:----------:|:-----------:|:------------------------:|
|DRAVP |ACK |-|
|POLL | DRAVP |-|
|DRAVS |ACK |User is prompted for age verification with age limit set by DRAVP|
|POLL |DRAVS |-|
|VEND_REQUEST|ACK |Terminal switches to payment mode|
|POLL |VEND_APPROVED|-|
|VEND_SUCCESS|ACK |-|


### Dedicated Age Verification

The process for the ***Dedicated*** age verification method is identical to the ***Embedded*** age verification method. The only difference is that the VMC needs to sent its commands **to the age verification device (address `68h`)** and ***not*** to the cashless device (address `10h`).

For the initialization sequence of the **Age verification Device**, please refer to the [MDB specification][1].

### Aborting A Requested Age Verification(Vend Request)

|VMC|SENVEND Terminal|Action|
|:----------:|:-----------:|:------------------------:|
|DRAVP|ACK|-|
|POLL|DRAVP|-|
|DRAVS|ACK|User is prompted for age verification with age limit set by DRAVP|
|VEND_REQUEST|ACK|Age verification is aborted, terminal switches to payment mode|
|POLL |VEND_APPROVED|-|
|VEND_SUCCESS|ACK|-|


### Aborting A Requested Age Verification(Age Off)

|VMC|SENVEND Terminal|Action|
|:----------:|:-----------:|:------------------------:|
|DRAVP|ACK|-|
|POLL|DRAVP|-|
|DRAVS|ACK|User is prompted for age verification with age limit set by DRAVP|
|DRAVP(Age=`00h`)|ACK|Age verification is aborted, terminal switches back to idle state|
|POLL|DRAVP|-|


### Aborting A Requested Age Verification(Age SupportedWillBeSwitchedOn)

|VMC|SENVEND Terminal|Action|
|:----------:|:-----------:|:------------------------:|
|DRAVP|ACK|-|
|POLL|DRAVP|-|
|DRAVS|ACK|User is prompted for age verification with age limit set by DRAVP|
|DRAVP(Age=`ffh`)|ACK|Age verification is aborted, terminal switches back to idle state|
|POLL|DRAVP|-|


### Aborting A Requested Age Verification(User Presses Back Button)

|VMC|SENVEND Terminal|Action|
|:----------:|:-----------:|:------------------------:|
|DRAVP|ACK|-|
|POLL|DRAVP|-|
|DRAVS|ACK|User is prompted for age verification with age limit set by DRAVP|
|-|-|User presses back button|
|POLL|DRAVS|Age not approved because of abort|


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
Sending cashless to VMC: [ffh, 06h, 07h, 9fh, 10h, 44h, 52h, 41h, 56h, 53h]
// Vend request
Got data from mdb: [13h, 00h, 00h, 3ch, 00, 0bh]
// Vend approved
Sending cashless to VMC: [05h, 00h, 3ch]
// Vend success
Got data from mdb: [13h, 02h, 00h, 0bh]
```

## Screenshots

Here are some screenshots of how screens on the terminal might look like:

### Age Verification Prompt

![2]

[1]: https://www.namanow.org/wp-content/uploads/Multi-Drop-Bus-and-Internal-Communication-Protocol.pdf
[2]: assets/age_select_screen.png
