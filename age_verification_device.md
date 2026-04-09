# Age Verification Device with SENVEND

This document describes the integration of the Age Verification Device, defined in the MDB specification, with the SENVEND payment terminal.

Keep in mind that this document only describes the integration of the "dedicated" Age Verification Device, similar to the DCM5. For information about the age verification features as part of the cashless device, refer to the [Cashless](cashless.md) and [Age Verification](age_verification.md) documentation.

Additionally, the actual process of validating a required age is described in [Age Verification](age_verification.md).

## Manufacturer Code, Serial Number, Model Number and Software Version

The SENVEND terminal provides the following values during the initialization sequence to the vending machine:

- Manufacturer Code: `SEN`
- Serial Number: `xxx-xxx-xxx␠` (unique per device. Is equal to the S/N printed on the white label on the back of the terminal)
- Model Number: `UX700␠␠␠␠␠␠␠` (note the spaces to fill 12 characters)
- Software Version: Will contain the current software version formatted as described in the [MDB specification][1]

[1]: https://www.namanow.org/wp-content/uploads/Multi-Drop-Bus-and-Internal-Communication-Protocol.pdf
