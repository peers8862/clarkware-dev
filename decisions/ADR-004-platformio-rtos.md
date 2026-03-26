# ADR-004: PlatformIO as the Embedded Build System and RTOS Awareness Strategy

**Status:** Accepted
**Date:** 2026-03-26

---

## Context

The IPE Firmware Panel needs a build system for embedded firmware. Clark's customers use a wide range of MCU families (STM32, NRF52, ESP32, RP2040, AVR/Arduino) and frameworks (Arduino, STM32Cube, ESP-IDF, Mbed, Zephyr). A build system that handles multiple targets uniformly is preferable to one that requires per-target configuration.

Additionally, many embedded projects use a Real-Time Operating System (RTOS). Debug visibility into RTOS thread state is important for diagnostics and commissioning at the Clark assembly node.

Options evaluated: CMake + toolchain files, PlatformIO, Zephyr's west tool, Mbed CLI.

---

## Decision

**1. PlatformIO as the primary embedded build system for the IPE Firmware Panel.**

PlatformIO's `platformio.ini` provides a single configuration file that handles: board selection, framework selection, toolchain download, library dependencies, upload protocol, and debug configuration. The PlatformIO VS Code plugin (`platformio-ide`) is available as a Theia VS Code plugin, giving operators a familiar interface.

The `@clark/platformio-service` Theia backend service wraps the PlatformIO Core CLI (`pio`). It exposes `build`, `upload`, `test`, and `clean` commands to the Firmware Panel. After each build, it exports `compile_commands.json` to the workspace root for clangd code intelligence.

**2. CMake is supported as a secondary build system** for customers who have existing CMake-based projects. The `DeviceDriverManager` detects `CMakeLists.txt` and activates the CMake build path. CMake's `compile_commands.json` export is used for clangd.

**3. RTOS awareness: FreeRTOS for Phase 1; Zephyr in Phase 2.**

FreeRTOS is the most widely deployed RTOS in the embedded electronics assembly space (STM32, NRF52, RP2040 all support it). FreeRTOS awareness is enabled in OpenOCD (`$_TARGETNAME configure -rtos FreeRTOS`) and in the `cortex-debug` DAP config (`"rtos": "FreeRTOS"`). This gives the debug session a live thread list view.

Zephyr RTOS has OpenOCD and `cortex-debug` awareness support but is more complex to configure. Defer to Phase 2.

Bare-metal (no RTOS) is always supported — RTOS awareness is disabled when `rtos` is set to `none` in the Firmware Panel configuration.

**4. Arduino is supported via PlatformIO's Arduino framework.** PlatformIO can build Arduino sketches for AVR, STM32, ESP32, NRF52 targets. This is the primary path for Clark's Arduino-compatible work. The standard Arduino IDE is not integrated — PlatformIO replaces it.

---

## Consequences

**Easier:**
- PlatformIO handles toolchain downloads automatically — no manual GCC toolchain installation
- `platformio-ide` plugin gives operators a UI they may already know
- Single `platformio.ini` handles the entire build/upload/debug configuration per project
- Arduino support comes for free through PlatformIO

**Harder:**
- PlatformIO requires internet access for initial toolchain and library downloads (or a local mirror for air-gapped sites)
- PlatformIO's custom upload hook (`upload_command`) requires testing to ensure it correctly integrates with `DeviceDriverManager`
- Customers with CMake projects need a separate code path
- RTOS awareness requires the firmware to export specific symbols — needs documentation for customers

**Open question:**
- Should PlatformIO run on the host or in Docker? Host is simpler for USB access; Docker isolates the toolchain but requires bind-mounting the project directory. **Decision: host for Phase 1.**
