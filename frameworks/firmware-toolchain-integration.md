# Framework: Firmware Toolchain Integration

**Covers:** The complete embedded development and firmware provisioning stack for the IPE Firmware Panel — GDB, DAP, OpenOCD, J-Link GDB Server, JTAG/SWD probes, CMSIS-DAP, PlatformIO, LSP (clangd), RTOS awareness plugins, SVD/CMSIS, ELF/DWARF binary handling, and the DeviceDriverManager Theia backend service.

---

## 1. Architecture Overview

The Firmware Panel is a full embedded development environment integrated into the IPE. It operates in two modes:

| Mode | Use Case | Tools Active |
|------|----------|-------------|
| **Flash mode** | Production: flash a pre-built firmware binary to a unit | DeviceDriverManager, probe driver, flash verifier, CFXPublisher |
| **Debug mode** | Development/diagnostics: set breakpoints, step through firmware, inspect registers | All of flash mode + LSP (clangd), DAP client, OpenOCD/J-Link GDB server, SVD viewer, RTOS thread view |

In production (Phase 1), Flash mode is the primary use case. Debug mode is available for engineering and diagnostics use.

---

## 2. Hardware Layer: JTAG/SWD Probes

### Supported probe types

| Probe | Protocol | Driver | Notes |
|-------|----------|--------|-------|
| SEGGER J-Link | JTAG/SWD | J-Link GDB Server (proprietary) | Best performance. Requires SEGGER license for production use. J-Link EDU for prototyping. |
| SEGGER J-Link BASE/PLUS | JTAG/SWD | J-Link GDB Server | Production-grade, volume license available |
| ST-Link v2/v3 | JTAG/SWD | OpenOCD (`stlink` driver) | STM32-native, cheap, no license. Limited for non-ST targets. |
| CMSIS-DAP (generic) | JTAG/SWD | OpenOCD (`cmsis-dap` driver) | Open standard HID interface. Works with any MCU. DAPLink, pyOCD compatible. |
| Black Magic Probe | JTAG/SWD | Native GDB serial interface | No OpenOCD needed — exposes GDB server directly via CDC serial. Simpler setup. |

### Probe enumeration in Node.js

```typescript
// Uses the 'usb' npm package to enumerate connected probes
import * as usb from 'usb';

const JLINK_VID = 0x1366;
const STLINK_VID = 0x0483;
const CMSIS_DAP_CLASS = 0xFF; // HID class

function enumerateProbes(): ProbeInfo[] {
  return usb.getDeviceList()
    .filter(d => isKnownProbe(d.deviceDescriptor))
    .map(d => ({ vid: d.deviceDescriptor.idVendor, pid: d.deviceDescriptor.idProduct, serial: getSerial(d) }));
}
```

---

## 3. GDB Server Layer: OpenOCD vs J-Link GDB Server

### OpenOCD

- **What it is:** Open-source on-chip debugger. Runs as a local process. Exposes:
  - GDB server on port 3333
  - Telnet control interface on port 4444
  - TCL scripting interface on port 6666
- **Launch from Node.js:**
  ```typescript
  const openocd = spawn('openocd', [
    '-f', 'interface/cmsis-dap.cfg',
    '-f', `target/${targetConfig}.cfg`,
    '-c', 'transport select swd'
  ]);
  ```
- **Target config files:** `/usr/share/openocd/scripts/target/stm32f4x.cfg`, `nrf52.cfg`, `rp2040.cfg`, etc.
- **RTOS support:** OpenOCD has built-in RTOS thread awareness plugins for FreeRTOS, ThreadX, eCos, ChibiOS, Zephyr, Mbed. Activated via `-c "rtos FreeRTOS"` in the OpenOCD config.

### J-Link GDB Server (SEGGER)

- **What it is:** Proprietary GDB server from SEGGER. Available as `JLinkGDBServerCL` (command-line, embeddable).
- **Launch from Node.js:**
  ```typescript
  const jlinkServer = spawn('JLinkGDBServerCL', [
    '-device', 'STM32F407VG',
    '-if', 'SWD',
    '-speed', '4000',
    '-port', '3333',
    '-nogui'
  ]);
  ```
- **Advantages over OpenOCD for J-Link probes:** Faster flash speeds, more reliable connection, better flash algorithm coverage, supports J-Link's flash download acceleration.
- **RTOS support:** J-Link GDB Server has built-in RTOS awareness for FreeRTOS, embOS, Zephyr, ThreadX, and others.

### Decision

Use **OpenOCD** as the default (open-source, no license requirement, CMSIS-DAP/ST-Link support). Use **J-Link GDB Server** when a J-Link probe is detected (better performance, still free for non-production use). The `DeviceDriverManager` selects the GDB server based on the detected probe type.

---

## 4. DAP Layer: Theia Debug Adapter Protocol Integration

### How Theia's DAP client works

Theia implements the Debug Adapter Protocol client natively via `@theia/debug`. It communicates with a DAP adapter over a socket or stdio pipe. The DAP adapter in turn communicates with the GDB server.

### DAP adapters for embedded ARM

| Adapter | Source | Notes |
|---------|--------|-------|
| `cortex-debug` (Marus25) | VS Code extension (Theia-compatible) | **Best choice for ARM Cortex-M.** Wraps OpenOCD or J-Link GDB Server. Supports SVD register view, RTOS thread list, peripheral view, ITM tracing. |
| `cppdbg` (MS C/C++) | VS Code extension | Generic GDB/MI adapter. Works but no ARM-specific features. |
| `gdb-debug` (WebFreak001) | VS Code extension | Pure GDB adapter, no ARM extras. |

**Recommendation: `cortex-debug`** as the primary DAP adapter. It is the de facto standard for embedded ARM debugging and has the richest feature set for Clark's use case.

### Theia DAP contribution

```typescript
// In clark-firmware-extension/src/browser/debug-configuration-provider.ts
@injectable()
export class FirmwareDebugConfigurationProvider implements DebugConfigurationProvider {
  type = 'cortex-debug';

  provideDebugConfigurations(folder?: FileStat): DebugConfiguration[] {
    return [{
      type: 'cortex-debug',
      request: 'launch',
      name: 'Clark IPE: Flash and Debug',
      servertype: 'openocd',  // or 'jlink' based on probe
      device: '${config:clark.firmware.targetDevice}',
      configFiles: ['${config:clark.firmware.openocdConfig}'],
      svdFile: '${config:clark.firmware.svdFile}',
      rtos: 'FreeRTOS',
      preLaunchTask: 'clark-firmware-build'
    }];
  }
}
```

---

## 5. LSP Layer: clangd for C/C++ Code Intelligence

### How clangd works

`clangd` is a language server for C/C++. It reads `compile_commands.json` (a compilation database) to understand include paths, compiler flags, and target definitions. For cross-compilation (ARM Cortex-M), the compilation database must include:
- `-target arm-none-eabi`
- `-mcpu=cortex-m4` (or appropriate CPU)
- `-mfpu=fpv4-sp-d16` (for FPU-enabled targets)
- System include paths for the ARM toolchain sysroot
- CMSIS include paths

### Generating compile_commands.json

**Via CMake:** `cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON`

**Via PlatformIO:** PlatformIO 6.x generates a `compile_commands.json` in `.pio/build/<env>/`. The `@clark/platformio-service` should export this file to the workspace root for clangd to find.

**Via `.clangd` config file:**
```yaml
# .clangd
CompileFlags:
  Add:
    - --target=arm-none-eabi
    - -mcpu=cortex-m4
    - -mfloat-abi=hard
  Remove:
    - -mno-thumb-interwork
```

### Theia LSP contribution

Theia's `@theia/monaco` and the built-in LSP client handle clangd automatically — it is treated as a standard LSP server. Clark's firmware extension registers the clangd server with target-specific flags:

```typescript
@injectable()
export class ClangdContribution extends BaseLanguageClientContribution {
  readonly id = 'clangd';
  readonly name = 'C/C++ (clangd)';

  protected createServerOptions(): ServerOptions {
    return {
      command: 'clangd',
      args: ['--background-index', '--clang-tidy', '--header-insertion=iwyu'],
    };
  }
}
```

---

## 6. SVD/CMSIS: Peripheral Register View

### What SVD files are

CMSIS-SVD (System View Description) files are XML files that describe an MCU's peripheral register layout: base addresses, register names, bit fields, reset values, access permissions. Every ARM Cortex-M MCU vendor provides SVD files for their devices.

### Sources

- **CMSIS-Packs:** Keil MDK pack server (`https://www.keil.com/pack/`) — downloadable `.pack` files contain `.svd` files
- **Vendor repositories:** STMicroelectronics, NXP, Nordic, Microchip all publish SVD files on GitHub
- **cmsis-svd repository:** `posborne/cmsis-svd` on GitHub — aggregated SVD files for hundreds of devices

### Integration with Cortex-Debug

Cortex-Debug reads SVD files to populate the Peripheral view. The `DeviceDriverManager` should maintain a local SVD library indexed by MCU part number. When a job specifies a target MCU (e.g., `STM32F407VGT6`), the Firmware Panel automatically loads the correct SVD.

```typescript
interface SVDLibrary {
  getPath(partNumber: string): string | undefined;
  download(partNumber: string): Promise<string>; // downloads from pack server if not cached
}
```

---

## 7. ELF/DWARF: Binary Handling

### ELF structure relevant to flash

```
ELF file
├── .text    — executable code → flash
├── .rodata  — read-only data → flash
├── .data    — initialized data → RAM (copied from flash at startup)
├── .bss     — zero-initialized data → RAM
└── .debug_* — DWARF debug info (strip before release if needed)
```

### Flash workflow

1. Receive ELF file path from job record or firmware build output
2. Compute SHA-256 hash of the ELF file — record as `FirmwareBinaryHash`
3. Invoke flash tool (OpenOCD or J-Link Commander) with the ELF path
4. After flash: read back a known region and CRC-verify against expected value
5. Record result: hash, version string, programmer serial, CRC result, flash timestamp
6. Emit `com.clark.ipe/FirmwareProvisioned` CFX message

### Binary version extraction

DWARF info can contain build version strings. The `@clark/firmware-utils` package should provide:
```typescript
function extractFirmwareVersion(elfPath: string): { version: string; buildDate: string; gitHash: string };
```
Using `arm-none-eabi-readelf` or the `dwarfdump` utility, or the `pyelftools` Python library as a subprocess.

---

## 8. PlatformIO Integration

### What PlatformIO provides

- Unified build system for 1000+ boards (Arduino, ESP32, STM32, RP2040, NRF52, etc.)
- Toolchain management (downloads GCC toolchains, SDKs automatically)
- Library manager (`lib_deps` in `platformio.ini`)
- Multiple framework support: Arduino, ESP-IDF, Zephyr, Mbed, STM32Cube
- VS Code / Theia plugin (`platformio-ide`)
- PlatformIO Core CLI: `pio run`, `pio run -t upload`, `pio test`, `pio debug`

### Integration model in the IPE

The `@clark/platformio-service` Theia backend service:
1. Detects a `platformio.ini` in the open workspace
2. Exposes commands: `Build`, `Upload`, `Test`, `Clean` to the Firmware Panel
3. After build: exports `compile_commands.json` to workspace root (for clangd)
4. After upload: reads build output for binary path and version string
5. Passes binary path to `DeviceDriverManager` for flash + verification

### platformio.ini for Clark-managed builds

```ini
[env:clark-target]
platform = ststm32
board = nucleo_f407zg
framework = stm32cube
upload_protocol = custom
upload_command = clark-flash $SOURCE  ; calls DeviceDriverManager
debug_tool = custom
debug_server =                         ; DeviceDriverManager manages GDB server
```

---

## 9. RTOS Awareness

### What RTOS awareness provides in a debug session

- List of all RTOS threads with names, states (running/blocked/suspended), stack usage
- Ability to switch between threads in the debugger (inspect each thread's stack)
- Detection of stack overflows, deadlocks

### FreeRTOS awareness setup

**Via OpenOCD:** Add to OpenOCD config:
```tcl
$_TARGETNAME configure -rtos FreeRTOS
```
**Via Cortex-Debug:** Set `"rtos": "FreeRTOS"` in the DAP launch config.

**Via J-Link GDB Server:** RTOS awareness is enabled automatically when a supported RTOS is detected.

### Requirement on firmware build

RTOS awareness requires that FreeRTOS symbols are not stripped from the binary. Specifically: `pxCurrentTCB`, `uxTopUsedPriority`, `pxReadyTasksLists`, `xDelayedTaskList1`. These must be exported from the linker script.

---

## 10. DeviceDriverManager — Theia Backend Service Design

```typescript
export interface IDeviceDriverManager {
  /** Enumerate connected probes */
  enumerateProbes(): Promise<ProbeInfo[]>;

  /** Connect to a probe and launch the appropriate GDB server */
  connect(probeSerial: string, target: FirmwareTarget): Promise<GDBServerSession>;

  /** Flash an ELF binary and verify */
  flash(session: GDBServerSession, elfPath: string): Promise<FlashResult>;

  /** Disconnect and clean up */
  disconnect(session: GDBServerSession): Promise<void>;

  /** Get path to SVD file for a target MCU */
  getSVDPath(partNumber: string): Promise<string>;
}

interface FirmwareTarget {
  partNumber: string;          // e.g. 'STM32F407VGT6'
  openocdConfigFile: string;   // e.g. 'target/stm32f4x.cfg'
  flashAlgorithm?: string;     // override default flash algorithm
  rtos?: 'FreeRTOS' | 'Zephyr' | 'ThreadX' | 'none';
}

interface FlashResult {
  success: boolean;
  binaryHash: string;          // SHA-256 of ELF
  crcVerified: boolean;
  flashDurationMs: number;
  errorMessage?: string;
}
```

---

## 11. npm Package Dependencies Required

| Package | Purpose |
|---------|---------|
| `usb` | Probe USB enumeration |
| `node-pty` | Spawn OpenOCD/JLinkGDBServerCL with PTY (handles readline-based output) |
| `@vscode/debugadapter` | DAP protocol types for the debug adapter bridge |
| `fast-xml-parser` | Parse CMSIS-SVD XML files |
| `js-sha256` | ELF binary hashing |

---

## 12. Outstanding Questions / Decisions Required

- [ ] Should PlatformIO run inside Docker or on the host? (Docker isolates toolchains but adds latency)
- [ ] ELF version extraction: `arm-none-eabi-readelf` subprocess vs `pyelftools` Python vs a pure-JS ELF parser?
- [ ] SVD library hosting: bundle a curated set with the IPE or download on demand from CMSIS-Pack server?
- [ ] RTOS support scope for Phase 1: FreeRTOS only, or also Zephyr and bare-metal?
- [ ] J-Link license procurement: J-Link BASE at ~$400/unit — confirm with CLARK operations
