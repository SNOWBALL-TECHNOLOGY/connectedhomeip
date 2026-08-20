# CC2745 Matter MSS Changes

## Overview

This document describes the downstream changes in `ti-cc2745-mss` for the TI
CC27xx Matter lighting application on LP-EM-CC2745R10-Q1.

The branch adds a production-oriented provisioning and secure-boot flow, then
extends the TI Matter BLE service with the C3 Additional Data characteristic
required to expose a Rotating Device Identifier (RDI).

## Source Branch

The official TI baseline is:

| Item | Value |
| --- | --- |
| Repository | `TexasInstruments/matter` |
| Branch | `ti/matter-v1.4-ti` |
| Commit | `c00ca92087e26d88ee8ac4cd140818b440d30c8d` |
| Release | `matter-v1.4-ti-1.0-EA-1.0` |

`ti-cc2745-mss` is 60 commits ahead of the official TI baseline. The
MSS-specific part consists of 11 commits.

The final difference from the official TI baseline contains 49 changed files,
2,601 insertions, and 298 deletions.

## What Changed

### 1. SimpleLink SDK 9.20 Integration

The CC23xx/CC27xx SimpleLink SDK submodule was updated from
`baceb8f138b446fb5f0dd04eb01d80c537b4b880` to
`68ca021502383f367d0bf2a5517fdd0dcb0ef909`.

The build integration was updated for the SDK 9.20 source and library layout:

- use the current `icall.c` source;
- include the BLE system-status implementation;
- use the current RCL header paths;
- build and verify required GCC and TI Clang libraries;
- use TI Clang secure-driver, HSM ITS, and CC2745 PSA Crypto libraries so the
  Matter application and provisioning firmware share the same persistent-key
  storage format.

These changes are needed because the original TI branch targets an earlier SDK
layout and does not provide all libraries required by this CC2745 build flow.

### 2. Factory Data

The factory-data structure and generator now support production identity and
commissioning data, including:

- Vendor ID, Product ID, serial number, product information, hardware version,
  and manufacturing date;
- setup discriminator, passcode, and SPAKE2+ values;
- DAC, PAI, and Certification Declaration;
- RDI unique ID;
- an optional DAC private key only for explicitly enabled legacy development
  images.

`FactoryDataProvider` validates field presence, size, format, and valid ranges
before the data is used. The Certification Declaration is read from provisioned
factory data instead of a hard-coded development value.

This prevents invalid provisioning data from reaching Matter commissioning and
ensures that device identity and RDI come from one consistent data source.

### 3. DAC Private Key in PSA/HSM

Production builds omit the DAC private key from factory data and store it as a
persistent P-256 PSA key backed by the CC2745 HSM.

The default key ID is `0x00027023`. During device attestation, the Matter
message is hashed with SHA-256 and the digest is signed with
`psa_sign_hash()` using ECDSA/SHA-256.

The startup key check is best-effort and does not block platform startup when
PSA/HSM is temporarily unavailable. A real attestation operation still returns
an error if the key cannot be used.

This keeps the DAC private key out of distributable firmware and prevents it
from being exported after provisioning.

### 4. Persistent Flash Layout

The application, persistent PSA storage, Matter NVS, and factory data use
separate regions:

| Region | Address | Size |
| --- | --- | ---: |
| ROM secure-boot header | `0x00000000` | `0x80` |
| Matter application | starts at `0x00000080` | ends before `0x000DC000` |
| Retained flash area | `0x000DC000-0x000FFFFF` | `0x24000` |
| PSA ITS / KeyStore | `0x000DF000-0x000E0FFF` | `0x2000` |
| Matter NVS | `0x000E1000-0x000E6FFF` | `0x6000` |
| Factory data | `0x000E7000-0x000E7FFF` | `0x1000` |

A strong `tfm_hal_its_fs_info()` implementation fixes PSA ITS at `0x000DF000`
so the Matter application uses the same KeyStore location as the temporary
provisioning firmware.

The CCFG erase-retain setting protects the flash tail beginning at
`0x000DC000`. Application update images exclude this complete retained range,
so reflashing Matter firmware does not erase the DAC key, factory data, or
Matter NVS.

### 5. ROM Secure Boot and External Signing

The application is linked for the CC2745 ROM secure-boot format:

- reset vectors start at `0x80`;
- a `0x80`-byte header and `0x640`-byte trailer are reserved;
- the secure-boot slot size is `0xDC000`;
- CCFG and SCFG data are extracted and checked;
- factory data and retained persistent data are removed from the update image;
- signing is performed by an external KMS;
- the signature and public key are verified before the final image is emitted.

The compatibility handling for TI SDK 9.20 is implemented inside the repository
helper script. It does not modify the installed TI SDK.

### 6. Provider Registration and Startup Order

The CC27xx `ConfigurationManagerImpl` initializes and registers the TI
`FactoryDataProvider` as the:

- `DeviceInstanceInfoProvider`;
- `DeviceAttestationCredentialsProvider`;
- `CommissionableDataProvider`.

Registration happens before BLE advertising starts. Duplicate application-level
registration was removed from the TI light-switch, lock, pump, and pump
controller examples.

This ordering ensures that VID, PID, discriminator, credentials, and RDI are
available when BLE advertising and commissioning start.

## MSS and Matter BLE C3

The MSS-specific changes add 475 lines and remove 12 lines across 12 files.

The lighting application enables:

```gn
chip_enable_additional_data_advertising = true
chip_enable_rotating_device_id = true
custom_factory_data = true
```

The production build also enables:

```gn
ti_dac_key_use_psa_hsm = true
```

### C3 GATT Characteristic

The shared TI CHIPoBLE profile now conditionally provides a read-only C3
Additional Data characteristic.

The implementation includes:

- a separate C3 UUID and GATT attributes;
- a maximum value size of 512 bytes;
- 16-bit length handling;
- GATT offset and segmented-read support;
- a callback for generating or recovering the C3 value;
- explicit tracking of the valid C3 data length.

The CC27xx target links `src/platform/ti/chipOBleProfile.c`, which contains the
C3-capable shared TI profile.

### RDI Payload

Before BLE advertising starts, `BLEManagerImpl`:

1. reads the RDI unique ID from `DeviceInstanceInfoProvider`;
2. reads the rotating-device lifetime counter;
3. generates the Matter Additional Data payload;
4. checks that the payload fits the C3 characteristic;
5. preloads the value into the GATT profile;
6. sets the Additional Data advertisement flag only when C3 is available.

The payload is cached in a `System::PacketBufferHandle` so repeated or segmented
GATT reads return the same data. If the cache is unexpectedly empty, the value
is regenerated and stored again.

This prevents the device from advertising Additional Data support when the
corresponding C3 value cannot be read.

## Build

The production-equivalent GN configuration includes:

```gn
ti_simplelink_board = "LP_EM_CC2745R10_Q1"
chip_enable_additional_data_advertising = true
chip_enable_rotating_device_id = true
custom_factory_data = true
ti_dac_key_use_psa_hsm = true
```

For CC27xx, `custom_factory_data=true` requires either:

- `ti_dac_key_use_psa_hsm=true` for production; or
- `ti_allow_factory_data_dac_private_key=true` for an explicitly selected
  legacy development image.

This build-time check prevents accidental production images containing a
plaintext DAC private key.

## CI Output and Checks

The CC27xx workflow builds the lighting application, prepares the secure-boot
image, signs it externally, and produces the unsigned and signed artifacts.

The workflow checks:

- the expected SimpleLink SDK commit and required libraries;
- the strong `tfm_hal_its_fs_info` override;
- reset-vector and application-slot placement;
- absence of application data in the retained flash range;
- the CCFG erase-retain setting;
- presence of the Matter BLE C3 implementation in both the ELF and signed BIN;
- generation of the signed binary, protected-data-free HEX, and manifest.

## Main Files

| Area | Files |
| --- | --- |
| CI | `.github/workflows/examples-cc27xx.yaml` |
| Secure boot | `scripts/tools/ti/cc2745_secure_boot.py` |
| Image validation | `third_party/ti_simplelink_sdk/validate_cc27xx_image.py` |
| SDK build integration | `third_party/ti_simplelink_sdk/ti_simplelink_sdk.gni` |
| Factory data generation | `third_party/ti_simplelink_sdk/create_factory_data.py` |
| Factory data and DAC signing | `src/platform/ti/FactoryDataProvider.cpp` |
| Flash layout | `src/platform/ti/cc27xx/cc27xxx10_freertos*.lds` |
| SysConfig and erase retention | `examples/platform/ti/sysconfig/chip_cc27xx.syscfg` |
| Provider registration | `src/platform/ti/cc27xx/ConfigurationManagerImpl.cpp` |
| C3 and RDI generation | `src/platform/ti/cc27xx/BLEManagerImpl.cpp` |
| TI CHIPoBLE profile | `src/platform/ti/chipOBleProfile.c`, `src/platform/ti/chipOBleProfile.h` |
| Lighting configuration | `examples/lighting-app/ti/cc27xx/args.gni` |
