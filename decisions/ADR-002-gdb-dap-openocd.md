# ADR-002: GDB Server and DAP Adapter Selection

**Status:** Accepted
**Date:** 2026-03-26

---

## Context

The IPE Firmware Panel requires hardware debugging capability. Three components must be chosen:
1. A GDB server (translates between GDB protocol and the physical JTAG/SWD probe)
2. A DAP adapter (bridges Theia's DAP client and the GDB server)
3. A probe driver approach (how the IPE detects and connects to physical probes)

Options for GDB server: OpenOCD (open-source, broad hardware support) or J-Link GDB Server (SEGGER proprietary, superior performance for J-Link probes).

Options for DAP adapter: `cortex-debug` VS Code extension, `cppdbg` (MS C/C++), or a custom implementation.

---

## Decision

**1. GDB server: OpenOCD as default; J-Link GDB Server when a J-Link probe is detected.**

The `DeviceDriverManager` inspects the detected probe's USB VID/PID. If a J-Link probe is present (VID 0x1366), it launches `JLinkGDBServerCL`. Otherwise it launches `openocd` with the appropriate interface config. This gives the best performance for J-Link (Clark's likely production probe) without forcing a proprietary dependency for development and testing with cheaper probes.

**2. DAP adapter: `cortex-debug` (Marus25) as the VS Code plugin loaded by Theia's plugin system.**

`cortex-debug` is the de facto standard for ARM Cortex-M debugging in the VS Code / Theia ecosystem. It provides: SVD-based register view, RTOS thread awareness, ITM/SWO tracing, peripheral view — all features the IPE Firmware Panel needs. It is loaded as a Theia VS Code plugin, not reimplemented.

**3. Probe detection: `usb` npm package for enumeration; `node-pty` for spawning GDB server processes.**

The `DeviceDriverManager` uses `usb.getDeviceList()` to enumerate connected HID/bulk transfer devices. Probes are identified by VID/PID from a maintained list. GDB server processes are spawned via `node-pty` which handles PTY I/O correctly for readline-based tools.

**4. JTAG vs SWD: SWD as the default transport for ARM Cortex-M targets.** SWD uses 2 pins vs JTAG's 4-5, is the standard interface on most ARM dev boards, and is supported by all probes. JTAG available as a configuration option.

---

## Consequences

**Easier:**
- OpenOCD + `cortex-debug` is a well-trodden path — large community, abundant config examples
- Probe auto-detection means operators don't need to configure the probe type manually
- `cortex-debug`'s SVD and RTOS features are available without custom implementation

**Harder:**
- J-Link GDB Server requires SEGGER's tooling to be installed on the IPE host — add to the IPE OS image/setup documentation
- OpenOCD configs are per-target — the IPE needs a curated library of target config files for the MCUs Clark's customers use (STM32, NRF52, RP2040, ESP32 via a JTAG bridge)
- `cortex-debug` plugin loading in Theia requires that Theia's VS Code plugin support is properly configured — needs testing with Theia 1.69+

**Open question:**
- Should the GDB server be run in Docker or directly on the host? Docker isolation is cleaner but USB passthrough to containers adds complexity on Linux (requires `--device` mount) and macOS (not natively supported in Docker Desktop). **Recommendation: run on host for Phase 1.**
