# ADR-005: SVD/CMSIS Peripheral View and ELF/DWARF Binary Handling

**Status:** Accepted
**Date:** 2026-03-26

---

## Context

Two related decisions are needed for the Firmware Panel's debug and flash capabilities:

1. **SVD/CMSIS:** Debug sessions on ARM Cortex-M need peripheral register visibility. SVD files describe the MCU's peripheral layout. How should the IPE manage SVD files?

2. **ELF/DWARF:** Firmware binaries arrive as ELF files. The IPE needs to: hash them (for the firmware provenance record), extract version information, and hand them to the flash tool. How should ELF/DWARF parsing work?

---

## Decision

### SVD/CMSIS

**1. Bundle a curated SVD library** for the 20-30 MCUs most likely to appear at Clark's initial customer nodes. Source: `posborne/cmsis-svd` GitHub repository (aggregated, open-source). Store in `packages/firmware-utils/svd/`.

**2. On-demand download** for MCUs not in the bundled library. The `SVDManager` service downloads from the `posborne/cmsis-svd` GitHub repository via the GitHub API when needed. Downloaded files are cached locally.

**3. SVD file selection** is automatic: the Firmware Panel reads the target MCU part number from the job record (set when the job is created). The `SVDManager.getPath(partNumber)` resolves the SVD path, which is passed to `cortex-debug` via the DAP launch config.

**Priority MCU families for the bundled library:**
- STM32F4xx, STM32F0xx, STM32L4xx, STM32H7xx (ST — most common in electronics assembly)
- NRF52840, NRF52832 (Nordic — IoT/BLE devices)
- RP2040 (Raspberry Pi — growing in hobbyist-to-industrial segments)
- SAMD21, SAMD51 (Microchip — Arduino MKR, Feather boards)
- ESP32-S3 (Espressif — note: Xtensa/RISC-V architecture, not ARM — SVD format differs)

**4. CMSIS-Pack** integration is deferred to Phase 2. CMSIS-Pack provides device support packages including SVD + startup code + linker scripts — valuable for toolchain setup but adds significant complexity. Phase 1 uses SVD files directly.

### ELF/DWARF

**1. Binary hashing:** Compute SHA-256 of the full ELF file using Node.js `crypto.createHash('sha256')` with a readable stream. No external tools required.

**2. Firmware version extraction:** Use `arm-none-eabi-readelf --string-dump=.rodata` to search for version strings in the read-only data section. Alternatively, the build system can write a version string to a known ELF symbol (e.g., `__clark_firmware_version`) that the IPE reads via `arm-none-eabi-nm`. This is the preferred approach — it is deterministic and doesn't require pattern matching.

**3. ELF parsing library:** Use `arm-none-eabi-readelf` (subprocess) for Phase 1. A pure-JS ELF parser would eliminate the toolchain dependency but adds implementation complexity. Defer pure-JS parser to Phase 2.

**4. Flash format:** Pass the ELF file directly to OpenOCD or J-Link GDB Server — both support ELF input natively. Do not convert to HEX or BIN unless required by a specific probe or target (add conversion via `arm-none-eabi-objcopy` as a fallback if needed).

**5. CRC verification:** After flash, use OpenOCD's `verify_image` command or J-Link's `verifybin` command to verify the written flash contents against the ELF. Record the verification result in the firmware ledger.

---

## Consequences

**Easier:**
- Bundled SVD library means zero network dependency for the most common MCUs
- SHA-256 hashing in Node.js requires no external tools
- ELF → flash tool is a direct path, no intermediate conversion
- `arm-none-eabi-readelf` is part of the ARM GNU toolchain already required by the build system

**Harder:**
- The SVD library must be maintained as new MCU families become common — needs a governance process
- The version extraction approach requires the firmware build to embed a version symbol — need to document this requirement for Clark's customers
- `arm-none-eabi-readelf` subprocess adds a dependency on the ARM toolchain being installed on the IPE host — document in prerequisites

**Open question:**
- For the firmware version symbol approach: should Clark define a standard ELF section and symbol for Clark-managed firmware? This would allow the IPE to consistently extract version info without customer-by-customer configuration. Potentially worth standardizing in the Clark firmware SDK.
