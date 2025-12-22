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

If the SENVEND terminal is used in Feature Level 3 (will be used automatically if the VMC supports it), the following optional features (as defined as Optional Feature Bits in the [MDB specification][1] in the `Peripheral ID` response) are supported:

- Always Idle
- Basket/Partial Refund/Options Price
- 16-bit and 32-bit monetary format
- Remote Vend

Note: Not all features may be enabled by every SENVEND terminal. That depends on the terminal configuration. The terminal will only report the features as supported, that are actually enbaled on the terminal. As described in the [MDB specification][1], the vending machine should never enable features in the `Enbale Options` command, that were not reported as supported by the terminal in the `Peripheral ID` response.

[1]: https://www.namanow.org/wp-content/uploads/Multi-Drop-Bus-and-Internal-Communication-Protocol.pdf
