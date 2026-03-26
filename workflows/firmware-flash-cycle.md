# Workflow: Firmware Flash Cycle

**Describes:** The complete sequence of events when an operator flashes firmware to a target unit at a Clark Assembly Centre workstation, from job step activation through CFX message publication.

---

## Preconditions

- An active job is open in the IPE with a `program` step as the current step
- The target unit serial number has been scanned
- A JTAG/SWD probe is physically connected to the workstation USB
- The firmware binary (ELF file) is available at the path specified in the job record or uploaded to MinIO

---

## Step-by-Step Flow

### 1. Step Activation
- Operator advances the job to the `program` step
- `JobService` emits a `job.step.activated` domain event
- `clark-firmware-extension` listens for this event and activates the Firmware Panel
- The Firmware Panel widget comes into focus in the IPE shell

### 2. Probe Detection
- `DeviceDriverManager.enumerateProbes()` scans USB devices
- If a probe is detected: Firmware Panel shows probe type, serial number, connection status
- If no probe: Firmware Panel shows "No probe detected — connect a JTAG/SWD probe"

### 3. GDB Server Launch
- Operator (or auto-trigger) connects to the detected probe
- `DeviceDriverManager.connect(probeSerial, target)`:
  - If J-Link probe: spawns `JLinkGDBServerCL` on port 3333
  - Otherwise: spawns `openocd` with the target config file
- GDB server stdout/stderr is piped to the Firmware Panel terminal widget

### 4. Binary Selection
- The job record specifies a firmware binary reference (MinIO object key or workspace file path)
- `FirmwareService` resolves the ELF file path
- SHA-256 hash of the ELF is computed and displayed in the panel
- If the hash matches an approved build hash in the firmware ledger: green ✓ indicator
- If the hash is not in the approved list: yellow ⚠ warning (operator can override with reason)

### 5. Flash Execution
- Operator clicks "Flash" (or approves the queued auto-flash)
- `DeviceDriverManager.flash(session, elfPath)`:
  - Issues `load_image <elfPath>` to OpenOCD (or equivalent J-Link command)
  - Monitors flash progress — progress bar in Firmware Panel
  - On completion: issues `verify_image <elfPath>` for CRC verification

### 6. Verification
- CRC verify result: Pass or Fail
- If Pass: flash record is committed
- If Fail: operator is notified, flash record is committed with `FlashResult: Failed`

### 7. Audit Record Creation
- `FirmwareService` writes a flash record to the `machine_sessions` or `firmware_ledger` table:
  ```json
  {
    "unitSerial": "SN-00842",
    "elfHash": "sha256:a3f8c2d1...",
    "versionString": "v1.4.2-prod",
    "targetMCU": "STM32F407VGT6",
    "programmerType": "SEGGER J-Link",
    "programmerSerial": "JL-800123456",
    "flashResult": "Success",
    "crcVerified": true,
    "operatorId": "op-007",
    "jobId": "WO-20250325-0019",
    "stepId": "step-04-program",
    "timestamp": "2026-03-26T14:32:01Z"
  }
  ```

### 8. CFX Message Publication
- `CFXPublisher.publish('com.clark.ipe/FirmwareProvisioned', payload)` is called
- Message is written to `cfx_outbox` with status `pending`
- Background flush loop delivers it to the RabbitMQ broker
- Network Status Bar confirms delivery

### 9. Domain Event
- `job.firmware.flashed` domain event is appended to the event store
- WebSocket broadcasts the event to all subscribers of this job's stream
- Job context panel updates to reflect the completed step

### 10. Step Completion Gate
- If CRC verification passed and audit record committed: the `program` step is marked complete
- Job advances to the next step
- Firmware Panel deactivates

---

## Error Paths

| Error | Response |
|-------|---------|
| No probe detected | Show "No probe" state, prevent flash |
| GDB server failed to start | Show error in terminal, log to AuditLogger |
| Flash failed (write error) | Mark flash record as Failed, do not advance step |
| CRC verify failed | Mark as Failed, prompt operator for re-flash or manual disposition |
| CFX message delivery failed | Write to outbox — delivery happens asynchronously, does not block step |
| ELF hash not in approved list | Show warning, require operator acknowledgment before proceeding |

---

## Key Invariants

- The flash record is **always** written, regardless of success or failure
- The CFX message is **always** queued, regardless of broker connectivity
- Step completion requires CRC verification Pass — a failed flash does not advance the job
- The operator ID is always recorded — no anonymous flash operations
