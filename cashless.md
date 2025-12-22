# Cashless Payments with SENVEND

This document provides an overview of cashless payment integration using the SENVEND payment terminal with the MDB (Multi-Drop-Bus) protocol.

## Overview
The SENVEND payment terminal currently only supports the `cashless device one` as defined in the MDB specification.

## Manufacturer Code, Serial Number, and software Version
The SENVEND terminal provides the following values during the initialization sequence to the vending machine:
- Manufacturer Code: `SEN` 
- Serial Number: `xxx-xxx-xxx ` (unique per device. Is equal to the S/N printed on the white label on the back of the terminal)
- Model Number: `UX700       ` (note the spaces to fill 12 characters)
