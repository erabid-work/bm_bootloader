# Configurable Bare-Metal A/B Bootloader

Minimal reusable Cortex-M bootloader framework with configurable A/B firmware banks.

## Features

- CRC32 image validation
- Configurable validation layer
- Dual application banks
- Pending/confirmed/invalid states
- Rollback architecture
- UART/USB/Ethernet transport extension points
- Flash/platform abstraction
- Cortex-M application jump
- ARM GCC toolchain
- CMake build
- MCU-specific `port/` directory

## Build

Install:

```bash
arm-none-eabi-gcc
arm-none-eabi-objcopy
cmake
make
```

Build:

```bash
cmake -S . -B build   -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake

cmake --build build -j
```

Outputs:

```text
build/baremetal_bootloader.elf
build/baremetal_bootloader.hex
build/baremetal_bootloader.bin
build/baremetal_bootloader.map
```

## Configuration

```bash
cmake -S . -B build   -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake   -DBOOTLOADER_START=0x08000000   -DBOOTLOADER_SIZE=0x00008000   -DBANK_A_START=0x08008000   -DBANK_A_SIZE=0x00038000   -DBANK_B_START=0x08040000   -DBANK_B_SIZE=0x00038000   -DMETADATA_START=0x08078000   -DMETADATA_SIZE=0x00008000   -DBOOT_TRANSPORT=UART   -DBOOT_VALIDATION=CRC32

cmake --build build -j
```

## Source tree

```text
baremetal_bootloader/
├── CMakeLists.txt
├── cmake/
│   └── arm-none-eabi.cmake
├── include/
│   ├── boot_config.h
│   └── boot_types.h
├── src/
│   ├── main.c
│   ├── bank/
│   ├── image/
│   ├── validation/
│   ├── metadata/
│   ├── transport/
│   ├── jump/
│   ├── platform/
│   └── config/
└── port/
    └── cortex_m/
        ├── startup.c
        └── linker.ld
```

## A/B update concept

```text
Bank A = CONFIRMED
Bank B = EMPTY

Download new firmware to B:

A = CONFIRMED
B = PENDING

Reboot:

Boot B

B confirms:
A = PREVIOUS
B = CONFIRMED

B fails:
B = INVALID
Boot A
```

## Separation of responsibilities

`main.c` contains boot policy only.

The feature implementations are separated:

- `validation/` — CRC/checksum/signature
- `transport/` — UART/USB/Ethernet
- `metadata/` — persistent boot state
- `bank/` — A/B policy
- `platform/` — MCU services
- `jump/` — application hand-off
- `port/` — architecture/MCU-specific code

Weak functions can be overridden by target-specific implementations.

## Important production work

The template is intentionally minimal. A production bootloader should add:

- redundant power-loss-safe metadata
- application confirmation
- watchdog-based boot-attempt handling
- flash erase/program driver
- transport framing/retry/timeout
- signed firmware authentication
- anti-rollback
- firmware version policy
- recovery mode
- MCU-specific startup/linker/vector handling
- cache/MPU handling where required

CRC32 provides integrity checking but is not cryptographic authentication.

## MCU adaptation

The supplied linker script is a generic Cortex-M4 example. Change the CPU, flash, RAM, vector table, startup, flash driver and application hand-off for the target MCU.

The CMake architecture is intended to keep those changes outside the bootloader policy.
