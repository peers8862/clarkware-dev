**CLARK**

Clarkware IPE — Architecture Specification

*Integrating Eclipse Theia and IPC-CFX v2.0 into the Integrated Production Environment*

Prepared for internal use. March 2025.

This document defines the architecture for Clarkware's Integrated Production Environment (IPE) — the workstation-level software system that sits at the heart of every Clark Assembly Centre node. It establishes how Eclipse Theia serves as the foundational framework, how IPC-CFX v2.0 provides the factory communication standard, how the firmware development environment is integrated, and how the IPE's design decisions flow from Clark's business model. It also addresses the naming question between Integrated Production Environment and Industrial Process Environment.

**1. The IPE Name — Integrated vs. Industrial**

The choice between Integrated Production Environment and Industrial Process Environment is a product positioning decision as much as a naming one. Both share the IPE initialism. The distinction matters because the name will be used in customer conversations, licensing agreements, training documentation, and ultimately in Clarkware's own UI.

|                       |                                                                                                                                                                                                                                                                        |                                                                                                                                            |
|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| **Dimension**         | **Integrated Production Environment**                                                                                                                                                                                                                                  | **Industrial Process Environment**                                                                                                         |
| What it describes     | A workstation where production activities happen — assembly, firmware loading, inspection, test, evidence capture. The word 'Integrated' signals that previously separate tasks (IDE, traveller, inspection log, firmware tool) are brought into a single environment. | An environment for managing and documenting industrial processes. Emphasizes process adherence and documentation. More ERP/QMS in flavour. |
| Who it resonates with | Engineers and technicians — people who know what an IDE is and understand the analogy immediately. The parallel to Integrated Development Environment is deliberate and useful.                                                                                        | Quality managers and operations directors. Less intuitive for the technicians who will actually use the workstation every day.             |
| Brand alignment       | Mirrors the IDE concept that Clark's founding team already works in. Signals that the IPE is the production-floor equivalent of the development workstation — a deliberate design philosophy.                                                                          | Generic enough to be applied to many industrial software categories. Harder to differentiate.                                              |
| IP and trademark      | 'Integrated Production Environment' as a product name for electronics manufacturing software is not a term of art — it is available as a brand. 'IPE' as an abbreviation is also available in this context.                                                            | 'Industrial Process Environment' is closer to existing industrial software terminology and therefore harder to brand distinctively.        |
| Recommendation        | Preferred                                                                                                                                                                                                                                                              | Not recommended                                                                                                                            |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>Recommendation — Integrated Production Environment</strong></p>
<p>Clark should adopt Integrated Production Environment as the full term and IPE as the abbreviation. The IDE analogy is intentional and powerful: just as a development IDE integrates code editor, compiler, debugger, and version control into one environment, the Clarkware IPE integrates job execution, workmanship inspection, firmware loading, evidence capture, and network coordination into one workstation environment. This framing is immediately legible to Clark's customers — engineers and hardware teams who already live in IDEs.</p></td>
</tr>
</tbody>
</table>

**2. What the IPE Is — A Conceptual Definition**

The Clarkware IPE is the software environment that an operator, technician, or engineer sees and interacts with at their workstation on a Clark Assembly Centre floor. It is the point at which the physical work of electronics assembly intersects with Clark's digital platform: jobs arrive, work is guided and performed, evidence is captured, firmware is loaded, inspection criteria are applied, and all activity is recorded and communicated to the Clark network and to the customer.

The IPE is not a quality management system. It is not an ERP. It is not a traditional MES in the sense of a centralized scheduling engine. It is a workstation-level environment — the digital equivalent of the technician's bench — that makes every action at that bench visible, auditable, and network-coordinated.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>The Design Philosophy in One Sentence</strong></p>
<p>The IPE brings the discipline of software development tooling — code intelligence, version control, debugger integration, real-time collaboration, AI assistance — to the electronics assembly workstation, and connects that workstation as a native participant in the IPC Connected Factory Exchange network.</p></td>
</tr>
</tbody>
</table>

**2.1 What the IPE Replaces**

At Clark's target nodes today, the workstation equivalent is a collection of disconnected tools and manual processes:

-   A paper job traveller that follows the assembly through the floor — gets lost, has no copy, cannot be queried.

-   A printed IPC workmanship reference binder — static, not contextual, not tied to the specific job class.

-   A separate programming utility on a different PC for firmware loading — version not logged, no tie to the job record.

-   A manual inspection log or a shared Excel sheet for defect recording.

-   Email or phone calls to coordinate with the customer or a downstream node.

-   A separate calibration schedule on a whiteboard or calendar.

The IPE replaces all of this with a single, coherent, networked workstation environment. The job traveller becomes a live digital job card. The IPC binder becomes contextual workmanship guidance surfaced at the moment of inspection. The firmware programmer becomes an integrated tool within the IPE. The inspection log becomes a structured, evidence-backed, audit-ready record. Coordination becomes a CFX message event, not a phone call.

**3. Why Eclipse Theia — The Technical Case**

Eclipse Theia is a professional-grade, open-source framework for building web and desktop IDEs and tools, implemented in TypeScript. It was originally developed by TypeFox and Ericsson, and has been under Eclipse Foundation governance since May 2018. In 2025, Theia AI won the CODiE Award for Best Open-Source Development Tool. Key contributors include EclipseSource, Red Hat, IBM, Google, and Arm Holdings.

Clark's choice of Theia as the IPE foundation is not arbitrary. Seven specific properties make Theia the correct choice for a firmware-aware, IPC-connected, multi-node production workstation environment.

**3.1 Seven Properties That Make Theia Right for the IPE**

**Property 1 — Everything is an extension. Nothing is hardcoded.**

In Theia, the extension mechanism is not a bolt-on API layer — it is the architecture. Every feature in Theia, including features built by the core team, is implemented as an extension using the same InversifyJS dependency injection container that any third-party developer uses. There is no privileged core. A Clark developer can override, replace, or augment any Theia component without forking the codebase. This means Clark can build a production workstation that looks and behaves nothing like a code editor while still building on a maintained, community-supported codebase.

**Property 2 — Frontend and backend are cleanly separated and communicate via JSON-RPC over WebSockets.**

Theia's architecture splits into a TypeScript/HTML/CSS frontend running in a browser or Electron window, and a Node.js backend. The frontend renders the UI. The backend provides services — file system access, process management, hardware device communication, network connections. For the IPE, this separation is critical: the AMQP broker connection for IPC-CFX lives in the backend service layer, hardware device drivers (JTAG programmers, test bench controllers) live in the backend, and the frontend operator interface is a clean TypeScript UI that communicates with those services via the same JSON-RPC channel Theia already uses internally.

**Property 3 — Language Server Protocol (LSP) support brings code intelligence to any language.**

Theia implements LSP as a first-class citizen, through its integration of the Monaco code editor (the same editor engine used by VS Code). Any language with an LSP server — C, C++, Rust, Assembly, Python, and more — gains full IDE capabilities (autocomplete, go-to-definition, inline errors, symbol search) inside the IPE. For Clark's firmware development environment, this means clangd (the C/C++ language server) provides full code intelligence for embedded firmware without any custom implementation. The IPE becomes a first-class firmware development workstation, not a basic text editor with a flash button.

**Property 4 — Debug Adapter Protocol (DAP) support connects any debugger.**

DAP is to debuggers what LSP is to language intelligence. Any debugger with a DAP adapter can be integrated into a Theia-based application. For embedded firmware development, OpenOCD — the open-source on-chip debugger — has a GDB server interface that can be bridged to DAP. This means the IPE can support hardware debugging of the target device being programmed: set breakpoints in firmware source code, step through execution, inspect registers and memory — all from within the IPE workstation interface, using the same JTAG/SWD probe already on the bench. No separate debug window, no separate tool chain.

**Property 5 — VS Code extension compatibility provides an enormous ecosystem.**

Theia supports VS Code extensions at the plugin API level. This gives the IPE immediate access to a vast ecosystem: Cortex-Debug (the leading embedded ARM debug extension), PlatformIO IDE (for Arduino and embedded platform management), ARM assembly syntax highlighting, RTOS awareness plugins, and hundreds of language-specific tools. Clark's developers do not need to build from scratch what the embedded development community has already built. The IPE inherits this ecosystem on day one.

**Property 6 — Theia AI provides a native AI agent framework.**

Theia AI, released in 2024 and winning the CODiE Award in 2025, is a framework built into the Theia platform for building AI-powered tools. It provides LLM communication management (supporting OpenAI, self-hosted models, and local LLMs), AI agent creation and orchestration, prompt management and runtime adjustment, and tool-calling so agents can interact with the application's own services and data. Clark's business plan specifically calls for human-and-AI agent audit trails. Theia AI is the infrastructure that makes this possible without building from scratch. An IPE AI agent can assist with defect classification, suggest corrective actions, generate inspection reports, and flag anomalies in real time — all within the same workstation environment, using the same extension architecture.

**Property 7 — Open governance, no vendor lock-in, commercially friendly license.**

Theia is governed by the Eclipse Foundation under the Eclipse Public License 2.0 — a commercially friendly open-source license that permits proprietary extension and commercial product deployment. Clark can build Clarkware IPE as a proprietary product on top of Theia, ship it to node operators under Clark's own license, and maintain full control of the IP in the extensions Clark develops. The underlying Theia framework remains open source and community-maintained — Clark does not carry the maintenance burden of the platform, only the IPE-specific extensions. Unlike VS Code (Microsoft-controlled, cannot be white-labeled), Theia is specifically designed for this use case.

**4. IPE Architecture — Layers and Components**

**4.1 Architectural Layers**

The Clarkware IPE organizes into five layers from the hardware interface upward to the Clark network. Each layer corresponds to a set of Theia backend services and frontend extensions.

|                            |                                                                                                                                                                                                                                                                            |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Layer 5: Clark Network** | CFX broker connection, cross-node job routing, customer portal events, platform audit log. Implemented as a BackendApplicationContribution that maintains the AMQP connection and publishes CFX messages. Communicates with Clark's central platform via secure WebSocket. |

|                                    |                                                                                                                                                                                                                                                                                               |
|------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Layer 4: IPE Application Shell** | The Theia ApplicationShell with Clark-customized layout: Job Panel, Workmanship Panel, Firmware Panel, Audit Panel, Network Status Bar. Each panel is a Theia widget implemented as a WidgetFactory contribution. The ApplicationShell uses Lumino DockPanel for flexible, detachable layout. |

|                                           |                                                                                                                                                                                                                                                         |
|-------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Layer 3: Clarkware Extension Services** | InversifyJS-registered backend services: JobService, InspectionService, FirmwareService, CFXPublisher, DeviceDriverManager, AuditLogger. These services expose JSON-RPC endpoints consumed by frontend panels. All are BackendApplicationContributions. |

|                                |                                                                                                                                                                                                                                  |
|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Layer 2: Protocol Adapters** | LSP servers (clangd for C/C++), DAP adapters (OpenOCD GDB bridge), IPC-CFX AMQP broker client, hardware device drivers (JTAG, ICT, test bench). Theia's built-in LSP and DAP client infrastructure handles connection lifecycle. |

|                                 |                                                                                                                                                                                                            |
|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Layer 1: Hardware Interface** | Physical workstation hardware: JTAG/SWD probe (J-Link, CMSIS-DAP), SMT equipment (CFX-enabled), test bench instruments, barcode/QR scanner, camera for evidence capture. Accessed through Layer 2 drivers. |

**4.2 Frontend Workstation Layout — The Five Panels**

The IPE workstation presents five primary panels within the Theia ApplicationShell. The layout is operator-configurable (Lumino DockPanel allows drag-and-drop panel arrangement) but defaults to the following:

**Panel 1 — Job Panel (primary, left column)**

The active job view. Displays the current work order number, assembly part number, revision, BOM, customer name, and current step in the traveller sequence. When a job is scanned in (barcode or QR), the JobService fetches the job record from the Clark platform and populates this panel. The traveller is a sequence of steps — each step has a type (assemble, inspect, program, test, calibrate, ship) and a completion gate. The operator progresses through steps by completing the required actions and logging evidence. The Job Panel is the IPE's equivalent of the paper traveller — but live, networked, and auditable.

Theia implementation: WidgetFactory contribution. Frontend panel communicates with JobService via JSON-RPC. JobService emits CFX WorkOrderStarted and UnitStarted messages when a job is opened.

**Panel 2 — Workmanship Panel (right column, upper)**

The IPC contextual guidance panel. When the active step is an inspection step, this panel surfaces the relevant IPC-A-610 workmanship criteria for the specific assembly class (Class 1, 2, or 3 — set in the job record). Criteria are rendered as structured reference material: condition name, accept criteria, reject criteria, reference image. The operator logs each inspection point as Pass, Fail, or Process Indicator. Failures trigger a defect record in the AuditLogger. The Theia AI agent in this panel can assist with defect classification — the operator describes or photographs the condition and the agent suggests the applicable IPC criterion and disposition.

Theia implementation: Custom WidgetFactory. InspectionService manages the inspection data model. IPC criteria database is a structured JSON library loaded by InspectionService at startup. AuditLogger records every inspection event with operator ID, timestamp, and result.

**Panel 3 — Firmware Panel (right column, lower — active on program steps)**

The integrated firmware development and programming environment. When the active step is a program step, this panel activates the full embedded development toolchain: Monaco editor with clangd LSP for code intelligence (C/C++/Rust/Assembly), a terminal for build tool access, a flash interface connected through the DeviceDriverManager to the physical JTAG/SWD programmer on the bench, and a binary version ledger that records what binary was loaded to which unit serial number, via which programmer, at what time. The Firmware Panel is the production equivalent of the developer's firmware IDE — but it lives inside the same workstation environment as the inspection and job management tools, and every flash event is logged to the AuditLogger and published as a CFX message.

Theia implementation: Monaco editor is native to Theia. clangd integration via Theia's built-in LSP client. OpenOCD GDB server bridged via DAP contribution. Flash events emitted by DeviceDriverManager, consumed by AuditLogger and CFXPublisher.

**Panel 4 — Audit Panel (lower bar or side panel)**

The evidence capture and audit trail view. Shows a chronological log of all events for the current job: scans, inspection results, firmware loads, signatures, photos, and AI-generated notes. The operator can capture photos (via connected camera or workstation webcam) and attach them to specific job steps as evidence. AI-generated summaries of the inspection record are available via Theia AI. The Audit Panel is the source of the auditable human-and-AI coordination record that Clark's business plan calls for. For defense-adjacent customers, this panel's output is the compliance documentation.

Theia implementation: Custom widget fed by AuditLogger's event stream. Photo capture via BackendApplicationContribution accessing the camera device. Theia AI agent generates structured summaries on demand.

**Panel 5 — Network Status Bar (bottom status bar)**

A persistent status bar showing: operator ID and IPC certification status, active job ID and step, CFX broker connection status, node ID and current Clark network context, and a live indicator of whether any cross-node jobs are waiting or in transit. The Network Status Bar makes the Clark platform context always visible without requiring the operator to navigate to a separate view.

Theia implementation: StatusBarContribution. CFXPublisher service emits connection status events consumed by the status bar. Operator identity provided by a session management service.

**5. IPC-CFX Integration in the IPE**

**5.1 CFX as the IPE's Communication Spine**

Every significant event in the IPE — a job opening, a unit completing a step, an inspection being logged, a firmware binary being flashed — is a candidate CFX message event. The IPE's CFXPublisher service is a BackendApplicationContribution that maintains a persistent connection to an AMQP broker (RabbitMQ or any AMQP 1.0 compliant broker) and publishes IPC-2591 formatted messages on each event.

The IPE does not require the broker to be remote. In a standalone node with no external network, the CFX broker can run locally on the workstation itself (RabbitMQ is a lightweight service). When the node is connected to the Clark network, the broker bridges to Clark's central AMQP infrastructure. The IPE's CFX messages flow upward to the Clark platform and sideways to any other CFX-aware systems the customer is already running.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>The CFX Broker Architecture</strong></p>
<p>Each IPE workstation runs a lightweight AMQP client in its Node.js backend. In a connected node, this client connects to a shared broker at the node level (one broker per node, not per workstation). The node-level broker in turn connects to Clark's platform broker, which aggregates events across all nodes in the network. This tiered architecture means IPEs work fully offline (local broker only) and sync to the Clark network when connectivity is available — important for manufacturing environments where network reliability is not guaranteed.</p></td>
</tr>
</tbody>
</table>

**5.2 IPE Events Mapped to CFX Messages**

The following table maps every significant IPE operator action to the corresponding IPC-CFX v2.0 message type. Messages marked v2.0 use the human workstation extensions introduced in the February 2025 release.

|                                  |                                           |                 |                                                                                                         |
|----------------------------------|-------------------------------------------|-----------------|---------------------------------------------------------------------------------------------------------|
| **IPE Operator Action**          | **CFX Message Type**                      | **CFX Version** | **Payload Highlights**                                                                                  |
| Job scanned in / opened          | WorkOrderScheduled, WorkOrderStarted      | 1.0+            | Work order ID, part number, quantity, assembly class, assigned node ID                                  |
| Unit loaded onto step            | UnitStarted                               | 1.0+            | Unit serial number, work order ID, current process step, workstation ID, operator ID                    |
| Inspection step completed        | InspectionCompleted                       | 1.0+            | Pass/Fail per inspection point, IPC criterion IDs, defect codes if failed, operator certification level |
| Defect recorded                  | UnitRepaired or NonConformanceCreated     | 1.0+            | Defect type, IPC criterion ID, disposition (repair, reject, use-as-is), evidence reference              |
| Operator activity at workstation | OperatorActivity                          | v2.0            | Operator ID, activity type, step ID, duration — human workstation extension                             |
| Work instruction step completed  | WorkInstructionCompleted                  | v2.0            | Step ID, operator confirmation, elapsed time — human workstation extension                              |
| Signoff required / completed     | SignoffRequired, SignoffCompleted         | v2.0            | Step ID, required certification level, signoff operator ID, timestamp                                   |
| Firmware binary loaded           | Custom CFX Extension: FirmwareProvisioned | v2.0 extension  | Binary hash, version string, target MCU, programmer ID, JTAG probe serial, unit serial                  |
| Unit completed all steps         | UnitCompleted                             | 1.0+            | Unit serial, final status (pass/fail), total cycle time, complete step sequence log                     |
| Job closed / shipped             | WorkOrderCompleted                        | 1.0+            | Work order ID, actual vs. planned completion, shipped unit count, carrier details                       |
| Material kit consumed            | MaterialInstalled                         | 1.0+            | Component part number, lot code, quantity, workstation ID — feeds BOM traceability                      |

Green rows use the v2.0 human workstation message extensions — the IPE's most direct point of alignment with the February 2025 CFX standard update.

**5.3 The FirmwareProvisioned Custom Message**

IPC-CFX does not currently have a native firmware provisioning message type — the standard focuses on mechanical assembly and electronic inspection processes. Clark's IPE will publish a custom extension message for firmware events. CFX explicitly supports extension messages through a vendor namespace mechanism. The FirmwareProvisioned message is a Clark proprietary extension under the namespace com.clark.ipe.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>// CFX Custom Extension — com.clark.ipe.FirmwareProvisioned</p>
<p>// Namespace: com.clark.ipe / Version: 1.0.0</p>
<p>{</p>
<p>"MessageName": "com.clark.ipe/FirmwareProvisioned",</p>
<p>"Version": "1.0.0",</p>
<p>"TimeStamp": "2025-03-25T14:32:01Z",</p>
<p>"UniqueIdentifier": "ipe-fw-20250325-143201-0042",</p>
<p>"Source": "clark-ipe-workstation-01",</p>
<p>"FirmwareProvisioningInfo": {</p>
<p>"UnitSerialNumber": "SN-00842",</p>
<p>"TargetMCU": "STM32F407VGT6",</p>
<p>"FirmwareBinaryHash": "sha256:a3f8c2d1...",</p>
<p>"FirmwareVersionString": "v1.4.2-prod",</p>
<p>"BinaryFilePath": "builds/v1.4.2-prod/firmware.elf",</p>
<p>"ProgrammerType": "SEGGER J-Link",</p>
<p>"ProgrammerSerialNumber": "JL-800123456",</p>
<p>"FlashResult": "Success",</p>
<p>"VerificationResult": "CRC-verified",</p>
<p>"OperatorID": "op-007",</p>
<p>"WorkOrderID": "WO-20250325-0019",</p>
<p>"JobStepID": "step-04-program"</p>
<p>}</p>
<p>}</p></td>
</tr>
</tbody>
</table>

This message gives any downstream system — the Clark platform, a customer's own MES, or a quality audit tool — a complete, verifiable, immutable record of exactly what firmware was loaded to exactly which unit, by whom, when, and with what result. For defense-adjacent customers, this is the firmware traceability record that their supply chain compliance teams require.

**6. The Firmware Development Environment in the IPE**

**6.1 Why Firmware Lives Inside the IPE**

In most EMS operations today, firmware loading is a disconnected activity. A technician uses a separate PC running a vendor-specific programming utility (STM32CubeProgrammer, J-Flash, MPLAB IPE, etc.) to load a binary to the target device. The binary file exists somewhere on a network share or a USB drive. The version is noted — if noted at all — in a comment on the paper traveller or in someone's memory. The connection between 'which binary went onto which board' and 'the job record for that board' is informal, fragile, and invisible to the customer.

Clark's IPE eliminates this disconnection entirely. The Firmware Panel is not a programming utility bolted onto a side monitor — it is the same workstation environment where the job is tracked and the inspection is logged. Every firmware event is a first-class IPE action, logged to the AuditLogger, published to CFX, and associated with the unit serial number.

**6.2 What the Firmware Panel Contains**

The Firmware Panel activates when the active job step is a program step. It presents four sub-components:

**Monaco Editor with Embedded Toolchain Intelligence**

The Monaco editor (Theia's built-in code editor, the same engine as VS Code) is configured with:

-   **clangd LSP:** clangd language server — provides full C/C++ code intelligence: autocomplete, go-to-definition, inline diagnostics, hover type information, refactoring. Configured with the project's compile_commands.json for accurate cross-compilation awareness.

-   **rust-analyzer LSP:** ARM Embedded Rust language server (rust-analyzer) — for projects using Rust embedded frameworks (embassy, RTIC). The same LSP client infrastructure handles both.

-   **Assembly grammar:** Assembly syntax highlighting — TextMate grammar for ARM Thumb/Thumb-2 and other target architectures loaded as a VS Code extension compatible with Theia's plugin host.

-   **SVD support:** CMSIS device header support — SVD (System View Description) files for ARM Cortex-M peripherals, enabling register-level hover documentation inside the editor.

**Hardware Flash Interface**

The flash interface connects the IPE backend to the physical programmer on the bench through the DeviceDriverManager service. The programmer connection is established via:

-   **J-Link:** SEGGER J-Link — via J-Link Commander CLI or the J-Link SDK, wrapped in a BackendApplicationContribution that exposes a JSON-RPC flash endpoint to the frontend.

-   **CMSIS-DAP:** CMSIS-DAP compliant probes (DAPLink, ULINK, Black Magic Probe) — via PyOCD or OpenOCD CLI bridge.

-   **OpenOCD:** OpenOCD server — started as a child process by the DeviceDriverManager, communicating over its telnet or TCL RPC interface. The GDB server side of OpenOCD is bridged to Theia's DAP client, enabling source-level debugging in the same panel.

When the operator clicks Flash, the IPE: (1) resolves the binary path from the job record, (2) verifies the binary hash against the expected hash in the BOM, (3) invokes the flash command, (4) verifies the flash result via readback, (5) logs the result to AuditLogger, and (6) publishes the FirmwareProvisioned CFX message. The operator cannot skip any of these steps — the Job Panel does not advance to the next step until a verified flash result is recorded.

**Binary Version Ledger**

The Binary Version Ledger is a persistent, append-only log maintained by FirmwareService. Every flash event adds a row: unit serial, binary hash, version string, programmer serial, operator ID, timestamp, and result. The ledger is queryable — the customer or Clark's platform can ask 'what firmware is on unit SN-00842' and get an authoritative, cryptographically verified answer.

The binary hash (SHA-256) is computed from the actual binary file before flashing and recorded in the CFX message. This means no one can claim a different binary was loaded after the fact. The ledger is the source of truth for firmware traceability.

**DAP-Based Hardware Debugger**

When the Firmware Panel is in debug mode (available to engineers, not production operators by default), Theia's built-in DAP client connects to the OpenOCD GDB server. The operator sees standard debug controls — Run, Pause, Step Over, Step Into, Breakpoints, Call Stack, Variables, Watch — all surfaced in the familiar IDE layout, operating directly on the target hardware via JTAG or SWD. This is the same experience as VS Code's Cortex-Debug extension, implemented natively in the IPE without requiring a separate tool.

**6.3 Binary-Level and Assembly Code Support**

Clark's founding team works at the binary and assembly level. The IPE supports this natively:

-   **Disassembly:** Disassembly view — Theia's debug infrastructure includes a built-in disassembly view. When debugging, the operator can see ARM/Thumb assembly alongside C source code, with the program counter tracked in real time.

-   **Memory view:** Memory view — live memory inspection during debug sessions, showing raw hex and ASCII content of any memory region. Essential for inspecting firmware parameters, calibration constants, and device configuration regions.

-   **Register view:** Register view — full register file display for the target MCU, updated at each debug step. ARM Cortex-M core registers, FPU registers, and peripheral registers (via SVD) are all visible.

-   **Binary diff:** Binary diff — FirmwareService can compare the binary on the device (read back via programmer) against the expected binary file and report any differences at the byte level. This is a production-quality check that goes beyond simple CRC verification.

-   **ELF/DWARF:** ELF/DWARF inspection — the FirmwareService can parse ELF files and extract DWARF debug information to resolve symbol names, function addresses, and variable locations without requiring a full debug session.

**7. Theia AI — The Intelligent Coordination Layer**

Theia AI, introduced in Theia 1.54 and released to production in 2024, is a first-class framework within the Theia platform for building AI-powered tools. It is not a chatbot bolted onto a code editor — it is an extensible agent architecture that gives any Theia extension access to LLM communication, prompt management, function calling, and multi-agent orchestration.

Clark's business plan calls for human-and-AI agent audit trails — a live coordination layer where AI assists operators, flags anomalies, generates evidence summaries, and participates in the documented record of what happened on the floor. Theia AI is the infrastructure that makes this concrete without requiring Clark to build LLM integration from scratch.

**7.1 IPE Agent Architecture**

The IPE defines four specialized AI agents using the Theia AI agent framework. Agents can communicate with each other dynamically and are configurable by Clark to use any LLM — cloud-based (Anthropic Claude, OpenAI), self-hosted (Ollama, LM Studio), or Clark's own fine-tuned model in later phases.

|                             |                                                                                                                                                                                                                                                                                                                                                                                                                 |
|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Inspection Agent**        | Assists operators during IPC inspection steps. When an operator marks a condition as uncertain, they can invoke the agent with a description or photo. The agent queries the IPC criteria database (injected as context) and suggests the applicable criterion, accept/reject classification, and disposition. The agent's recommendation and the operator's final decision are both logged in the AuditLogger. |
| **Defect Analysis Agent**   | Activated when a defect is recorded. Reviews the defect record against the job's historical defect pattern (via AuditLogger query) and suggests root cause hypotheses and corrective actions. Generates a structured defect report in CFX InspectionCompleted payload format.                                                                                                                                   |
| **Firmware Advisory Agent** | Available during program steps. Can answer questions about the target firmware (if source code is accessible in the workspace), explain what a binary parameter does, or identify the implications of a specific version change. Uses DWARF debug information extracted by FirmwareService as context.                                                                                                          |
| **Audit Summary Agent**     | On job close, generates a structured audit summary: all steps completed, inspection results, defects and dispositions, firmware version loaded, operator IDs, and timestamps. The summary is appended to the job record and available for customer delivery. Uses the AuditLogger event stream as its source.                                                                                                   |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>Transparency and Control</strong></p>
<p>Theia AI is designed with full transparency — operators and auditors can see exactly what prompts were sent to the AI and what it returned. The complete AI communication history is logged in the AuditLogger alongside human operator events. This is critical for defense-adjacent work where AI involvement in quality decisions must be documented and auditable. Clark's LLM provider choice can be fully self-hosted if customer data sovereignty requirements demand it.</p></td>
</tr>
</tbody>
</table>

**8. Clarkware Extension Architecture — What Clark Builds**

Theia provides the framework. Clark builds extensions. The following defines what Clark owns and maintains versus what is inherited from the Theia platform and ecosystem.

**8.1 What Theia Provides (Clark Does Not Build)**

-   ApplicationShell — the workbench chrome, Lumino-based panel management, tab system, status bar, toolbar.

-   Monaco editor — code editor engine with syntax highlighting, multi-cursor, find/replace, minimap.

-   LSP client infrastructure — protocol handling, server lifecycle management, editor decoration for diagnostics.

-   DAP client infrastructure — debug session management, breakpoints, variable inspection, call stack.

-   Extension/plugin host — VS Code extension compatibility layer, dependency injection container.

-   Theia AI framework — LLM communication, agent lifecycle, prompt management, function calling.

-   Terminal — integrated terminal widget backed by node-pty, providing shell access to the backend environment.

-   File system abstraction — virtual file system that can be backed by local disk, remote, or custom providers.

**8.2 What Clark Builds (Clark's Proprietary IP)**

|                          |                                                                                                                                                                                                          |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Extension / Service**  | **Responsibility**                                                                                                                                                                                       |
| clark-ipe-core           | InversifyJS module binding all Clark services into the Theia DI container. The entry point for the Clarkware IPE application.                                                                            |
| clark-job-service        | Backend service: job record loading from Clark platform, traveller step management, step completion gating, job close. JSON-RPC endpoint consumed by Job Panel.                                          |
| clark-inspection-service | Backend service: IPC criteria database (structured JSON), inspection point recording, defect record creation, inspection report generation.                                                              |
| clark-firmware-service   | Backend service: DeviceDriverManager (J-Link, OpenOCD, CMSIS-DAP adapters), binary hash verification, flash command execution, readback verification, Binary Version Ledger.                             |
| clark-cfx-publisher      | Backend service: AMQP broker connection, IPC-CFX v2.0 message construction using IPC SDK, event subscription from other Clark services, message publication, connection state management.                |
| clark-audit-logger       | Backend service: append-only event log for the current job, evidence attachment management (photos, AI summaries, operator notes), AuditLog export for customer delivery.                                |
| clark-job-panel          | Frontend widget: Theia WidgetFactory contribution rendering the Job Panel. Consumes clark-job-service via JSON-RPC.                                                                                      |
| clark-workmanship-panel  | Frontend widget: Workmanship Panel with IPC criteria display, inspection recording UI, photo capture integration, Inspection Agent chat window.                                                          |
| clark-firmware-panel     | Frontend widget: Firmware Panel layout manager, flash interface UI, Binary Version Ledger view, DAP debug controls wrapper.                                                                              |
| clark-audit-panel        | Frontend widget: Audit Panel timeline view, evidence viewer, Audit Summary Agent trigger.                                                                                                                |
| clark-network-statusbar  | Frontend contribution: StatusBarContribution showing node ID, CFX connection status, operator certification status, job ID.                                                                              |
| clark-session            | Frontend/backend service: operator login, IPC certification status lookup, role-based access control (which panels are visible to which operator roles).                                                 |
| clark-ipe-agents         | Theia AI agent definitions: Inspection Agent, Defect Analysis Agent, Firmware Advisory Agent, Audit Summary Agent. Each defined as a Theia AI AgentContribution with prompt templates and tool bindings. |

**9. Implementation Sequence — From Shell to Full IPE**

The IPE is built incrementally. The following sequence is aligned with Clarkware's Phase 1-4 product roadmap and ensures each phase produces a deployable, useful workstation at Clark's nodes.

**Phase 1 — The Digital Traveller (Weeks 1–8)**

Build clark-ipe-core, clark-job-service, clark-job-panel, clark-cfx-publisher (stub), and clark-audit-logger. Deploy to one workstation at Niagara Assembly. The result: a workstation that replaces the paper traveller with a live digital job card. CFX messages are logged locally only. No firmware integration, no IPC inspection guidance yet. Operator feedback drives Panel 2 and 3 design.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>Phase 1 CFX status</strong></p>
<p>WorkOrderStarted and UnitStarted/UnitCompleted messages published. Local broker only. This is a functional CFX endpoint — the workstation is already a native CFX participant.</p></td>
</tr>
</tbody>
</table>

**Phase 2 — Network Coordination (Weeks 8–16)**

Connect clark-cfx-publisher to the Clark platform broker. Build clark-network-statusbar. Deploy clark-session with operator login and role management. Cross-node job visibility becomes live. Customer-facing job status becomes real-time. This is the feature no Unisoft or Tulip deployment can offer — and it is fully functional before any firmware or AI capability is added.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>Phase 2 milestone</strong></p>
<p>The Clark network can see every open job across all connected nodes in real time. Any operator at any node can see what step a job is on, who is working on it, and when it was last updated. This is the platform value proposition made tangible.</p></td>
</tr>
</tbody>
</table>

**Phase 3 — Workmanship Intelligence (Weeks 16–24)**

Build clark-inspection-service (IPC criteria database, inspection recording), clark-workmanship-panel, and clark-ipe-agents (Inspection Agent and Defect Analysis Agent). The Workmanship Panel surfaces IPC-A-610 criteria in context during inspection steps. AI assistance for defect classification becomes available. InspectionCompleted CFX messages carry structured quality data.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>Phase 3 milestone</strong></p>
<p>A Clark node can produce an IPC-aligned, AI-assisted, CFX-structured quality record for every assembly. This is the document a defense-adjacent customer's procurement team will ask for on first audit.</p></td>
</tr>
</tbody>
</table>

**Phase 4 — Firmware Environment (Weeks 24–40)**

Build clark-firmware-service (DeviceDriverManager, Binary Version Ledger), clark-firmware-panel, and clark-ipe-agents (Firmware Advisory Agent). Integrate LSP servers (clangd, rust-analyzer), DAP adapter (OpenOCD bridge), and flash interface. Publish FirmwareProvisioned CFX custom extension messages.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p><strong>Phase 4 milestone</strong></p>
<p>The IPE becomes the only production workstation tool in the small EMS market that provides firmware traceability from binary hash to unit serial number, with LSP-powered code intelligence and hardware debugging, all in the same environment as the job traveller and inspection log.</p></td>
</tr>
</tbody>
</table>

**10. Deployment — Desktop or Browser, Connected or Standalone**

Theia supports three deployment modes. Clark should support all three to accommodate the variety of node environments in the corridor.

|                                      |                                                                                                                                                                                                                                                                                                                                                         |
|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Electron desktop app**             | The IPE runs as a native desktop application on a Windows or Linux workstation. No browser, no network dependency for the frontend. The Node.js backend runs locally in the Electron main process. This is the default for production workstations — fast, offline-capable, full hardware device access. The most deployable option for corridor nodes. |
| **Browser-connected app**            | The IPE frontend runs in a browser; the backend runs on a local or network server. Suitable for shared workstation environments or thin-client deployments. Requires stable local network but allows multiple screens/operators to share a single backend.                                                                                              |
| **Browser-only (frontendOnly mode)** | For lightweight read-only IPE views (Audit Panel, Network Status Bar) that do not require hardware device access. Introduced in Theia 1.46. Useful for customer-facing job status dashboards or management visibility panels that can be hosted as static web pages.                                                                                    |

For Phase 1 deployment, Clark should ship the IPE as an Electron desktop application on a standard mini-PC (Intel NUC class or similar) attached to each workstation. This eliminates network dependency, provides full hardware device access, and is the most operator-familiar form factor — it looks like a computer, not a server appliance.

Document prepared for Clark internal strategy use. March 2025.

Architecture defined by Clark based on Eclipse Theia platform (Eclipse Foundation, EPL 2.0), IPC-2591 Connected Factory Exchange v2.0 (IPC, open standard), and Clark's proprietary Clarkware extension layer. Theia technical references: theia-ide.org, github.com/eclipse-theia/theia, eclipsesource.com. CFX technical references: ipc-cfx.org, IPC-2591 v2.0 specification.
