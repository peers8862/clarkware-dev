# Clarkware Architecture v1

## Purpose

This document translates the Clarkware concept into an initial technical architecture for version 1. It is intended to be specific enough to guide early product design, technical diligence, and first implementation decisions without pretending that every engineering choice is already settled.

This architecture is built around a simple principle: Clarkware should preserve and relay trustworthy operational memory inside real industrial environments.

## Version 1 Goal

Clarkware version 1 should prove that CLARK can deploy workstation-level Integrated Production Environments inside a real facility, connect them to meaningful production activity, capture human and AI actions in a reviewable record, and produce facility-level visibility from that underlying record.

The first architecture should therefore optimize for:

- workstation usability
- evidence capture
- auditable messaging and coordination
- node-resident deployment
- controlled remote participation
- extensibility for later integrations

It should not optimize first for broad ERP replacement, full plant automation, or complex multi-site orchestration.

## Architectural Summary

Clarkware version 1 should be structured as six cooperating layers:

1. IPE client layer
2. Local integration and event capture layer
3. Messaging and collaboration layer
4. Operational record and query layer
5. Reporting and workflow services layer
6. Node infrastructure and trust layer

Each layer should be replaceable or extensible without forcing the whole product family into one rigid application.

## Layer 1: IPE Client Layer

### Role

The IPE client is the primary workstation-level environment where operators, technicians, quality personnel, supervisors, and experts interact with Clarkware.

### Implementation Direction

Theia should be the main shell for version 1 IPEs where mixed tooling and extension-based workspaces are needed. Theia is a strong fit because it already supports modular panels, commands, views, logs, editors, extensions, and multi-runtime deployment patterns.

Version 1 IPE clients should be able to run in one or both of these forms:

- browser-based client served from a node-resident Clarkware server
- packaged desktop client for stations that need stronger local-first behavior or tighter integration with local tools

### Core IPE Surfaces

Each IPE should be able to host these first-class surfaces:

- job context view
- workstation context view
- note composer and note history
- conversation and presence pane
- event stream pane
- artifact and attachment pane
- machine or tool integration pane where applicable
- AI assistant pane with reviewable outputs
- task and checklist pane

### UI Principles

The UI should prefer:

- fast access to current job context
- persistent visibility of workstation identity
- one-click note capture
- explicit distinction between observed facts, proposed actions, and AI-generated content
- strong keyboard support where practical
- low-friction image, file, and test-output attachment

## Layer 2: Local Integration And Event Capture Layer

### Role

This layer connects the workstation environment to the local operational reality: tools, instruments, files, sessions, and operator actions.

### Version 1 Responsibilities

- collect workstation session events
- normalize local tool and test events into Clarkware event objects
- bind events to workstation, job, and actor context where possible
- support attachment capture from local file systems, screenshots, exports, and instrument outputs
- buffer events during temporary network disruption

### Integration Model

Version 1 should prefer adapters over deep monolithic integrations. An adapter should do four things:

- identify the source system or device
- capture raw events or outputs
- map them into canonical Clarkware objects
- preserve raw evidence when useful for later audit or debugging

### Niagara-First Integration Priorities

The first Niagara Assembly integration set should focus on testing and note-linked evidence capture. Initial targets should include:

- bench test instruments that export results as files, CSV, JSON, XML, or serial output
- calibration records and measurement exports
- image capture tied to workstation and job context
- firmware build or test artifacts relevant to a production or diagnostics workflow
- operator note capture linked to attached evidence and current task state

Version 1 should not require full direct command-and-control integration with every device. It is enough to prove trustworthy event capture and contextual linkage from a small number of high-value tool paths.

## Layer 3: Messaging And Collaboration Layer

### Role

This layer supports the conversations, alerts, and presence information that allow distributed industrial work to move quickly without losing accountability.

### Core Direction

An XMPP-based or strongly XMPP-aligned server architecture should be treated as a first-order requirement, not an optional add-on. Clarkware's collaboration model depends on durable identity-aware communication that can include humans, AI agents, and automation services.

### Why XMPP Fits

XMPP is a strong fit because it supports:

- identity-aware messaging
- presence
- room-based and direct communication patterns
- extensibility through standardized and custom extensions
- federation options for later multi-node topologies
- inspectable standards-based behavior

### Version 1 Messaging Objects

The messaging layer should support at least these entities:

- direct conversation
- workspace channel
- job-linked thread
- issue-linked thread
- AI agent thread or AI-participant identity
- system alert message
- workflow notification

### Message Design Requirements

Messages should be able to carry:

- sender identity
- sender type: human, AI, or automation
- timestamp
- facility and workstation context when relevant
- linked job or issue context
- attachments or artifact references
- message classification such as note, question, alert, recommendation, handoff, or resolution
- review state where an AI-generated message requires explicit acknowledgment

### Presence Model

Presence should not only mean online or offline. Clarkware should support operational presence states such as:

- available
- heads down
- on station
- in review
- awaiting response
- unavailable
- automation active

These states should remain simple in version 1 and not become heavy workflow bureaucracy.

## Layer 4: Operational Record And Query Layer

### Role

This layer is the core memory of Clarkware. It stores the canonical record of what happened.

### Design Principle

The system should preserve both narrative continuity and structured queryability. This means Clarkware needs both linked operational objects and durable event history.

### Canonical First-Class Objects

Clarkware version 1 should define at least the following first-class objects:

- Facility
- Zone
- Workstation
- Job
- Task
- Issue
- Conversation
- Message
- Note
- Event
- Tool
- Machine Session
- Artifact
- Person
- Agent
- Role
- Permission Grant

### Core Relationship Rules

The most important relationships are:

- every workstation belongs to a facility
- every event should belong to a workstation, job, issue, or conversation context where possible
- every note should identify author, time, and context
- every AI action should identify the responsible agent identity and linked review context
- every artifact should be linkable to source event, job, and workstation
- every conversation should be linkable to a job, issue, workspace, or facility scope

### Event Model

The event model should be append-first. In practice this means new events are written as new records, while higher-level objects may maintain current state derived from those events.

Example event types:

- workstation.session.started
- workstation.session.ended
- job.started
- job.paused
- task.completed
- note.created
- note.revised
- artifact.attached
- test.run.imported
- calibration.result.imported
- message.sent
- ai.summary.generated
- ai.recommendation.accepted
- ai.recommendation.rejected
- issue.opened
- issue.resolved

### Storage Direction

Version 1 should use a pragmatic mixed storage model:

- relational storage for canonical objects and permissions
- append-oriented event storage for event history and replayable audit trails
- object storage for files, images, exports, logs, and evidence artifacts
- search indexing for notes, messages, and key event fields

The exact database choices can vary, but the model should preserve structured integrity, durable history, and fast retrieval.

## Layer 5: Reporting And Workflow Services Layer

### Role

This layer turns raw operational records into useful views, summaries, escalations, and reports.

### Version 1 Responsibilities

- build role-based dashboards from canonical data
- generate issue and bottleneck summaries
- support shift, workstation, job, and facility reporting
- surface test and quality exceptions
- provide recent-history and handoff views
- generate AI-assisted summaries that remain reviewable and attributable

### Reporting Principle

Version 1 reports should be derived from recorded events, notes, messages, and artifacts. Clarkware should avoid dashboards that look polished but are disconnected from the underlying operational record.

### Workflow Principle

Clarkware should support workflow evolution rather than rigid deterministic sequencing. Version 1 workflows should therefore be light-touch:

- assign
- notify
- acknowledge
- escalate
- resolve
- hand off

This is enough to support real operations without forcing every site into an over-designed workflow engine.

## Layer 6: Node Infrastructure And Trust Layer

### Role

This layer makes Clarkware deployable in real facilities with privacy, inspection, ownership, and resilience properties that support CLARK's business model.

### Deployment Model

Each major node should run Clarkware server infrastructure under CLARK's operating model, with data governance and access rules appropriate to the facility and customer context.

A typical node deployment should include:

- Clarkware application services
- XMPP or XMPP-aligned messaging services
- operational database services
- search and indexing services
- object storage for artifacts and backups
- integration adapter services
- identity and access control services
- observability and audit services

### Trust Requirements

Version 1 infrastructure should support:

- owner-accessible exports
- clear backup and restore procedures
- inspectable service architecture
- environment-level audit logging
- role-based remote access controls
- documented data residency and retention rules

### Remote Access Model

Remote access should be treated as controlled participation, not unrestricted platform access. A remote expert should be able to view or contribute to the contexts they are authorized for without gaining unnecessary visibility into unrelated facility activity.

## Identity And Permission Model

### Identity Types

Clarkware should recognize three broad actor types:

- human user
- AI agent
- automation service

Each actor should have a distinct identity and attributable actions.

### Human Roles

Version 1 should support roles such as:

- owner
- facility administrator
- supervisor
- operator
- quality reviewer
- remote expert
- observer or auditor

### Permission Model

Permissions should be granted at multiple scopes:

- facility scope
- zone or department scope
- workstation scope
- job scope
- issue scope
- conversation scope

### Permission Categories

Version 1 permission categories should include:

- view
- comment
- create note
- attach artifact
- initiate conversation
- participate remotely
- review AI output
- approve disposition
- administer integrations
- export records
- manage retention and backup settings

### AI Permission Rules

AI agents should not inherit broad human permissions by default. They should operate through explicit bounded grants tied to defined contexts and action classes.

An AI agent should be able to:

- summarize activity in authorized contexts
- propose next actions
- draft notes or reports
- route messages or alerts

An AI agent should not be able to silently finalize critical operational actions without explicit human review unless a later, separate automation policy allows that in a tightly bounded case.

## Local-First And Offline Behavior

### Principle

Clarkware version 1 should assume that workstation reliability matters more than permanent connectivity.

### Minimum Local-First Guarantees

At minimum, a workstation IPE should continue to support during temporary disconnection:

- viewing the current assigned job context and recent local history
- capturing notes
- attaching locally available artifacts
- recording local events with reliable timestamps
- queueing messages and sync operations for later delivery
- preserving explicit sync status so users know what has and has not propagated

### Sync Rules

When connectivity returns, the system should:

- sync queued notes, events, and attachments
- preserve original local timestamps and creation context
- mark conflicts explicitly rather than silently overwriting
- keep append-only event history intact

### Conflict Strategy

Version 1 should prefer explicit conflict visibility over hidden automatic reconciliation in high-accountability records.

## AI Agent Architecture

### Role Of AI In Version 1

AI in Clarkware version 1 should be assistive, not sovereign.

Primary AI functions should include:

- summarization of recent activity
- issue triage suggestions
- note drafting assistance
- retrieval of related historical context
- alert routing suggestions
- handoff summary generation

### AI Execution Pattern

AI services should operate as explicitly identified participants attached to recorded contexts rather than as invisible background magic.

Every meaningful AI output should carry:

- agent identity
- generation time
- source context references where practical
- review state
- acceptance, edit, or rejection outcome where applicable

### Agent Communication Pattern

AI agents should participate through the same messaging and event framework as other actors wherever feasible. This keeps the audit model coherent.

## Suggested Service Topology

A pragmatic version 1 topology could include:

- web gateway and API service
- IPE workspace service
- XMPP messaging service
- identity and access service
- event ingestion service
- integration adapter service
- reporting and query service
- AI orchestration service
- relational database
- event store
- object storage
- search index

This can start as a modular monolith with clearly separated service boundaries, then split further only where scale or operational concerns justify it.

## Security And Compliance Posture

Version 1 should emphasize:

- strong authentication
- role-scoped authorization
- transport encryption
- audit logging for sensitive actions
- explicit remote-session controls
- documented retention behavior
- exportability for owner review
- clear separation between operational data and broader cross-node intelligence layers

Clarkware should be able to support compliance-related workflows through better records and traceability, but it should avoid marketing itself as a complete compliance platform until those claims are proven in operation.

## Niagara Assembly Pilot Blueprint

The first credible deployment should prove five things at Niagara Assembly:

1. A workstation-level IPE can be used in real daily work.
2. At least one meaningful test-tool path can feed evidence into the record.
3. Operators and remote experts can collaborate in job-linked threads.
4. AI agents can generate useful summaries or routing suggestions with full auditability.
5. Facility leadership can review a trustworthy activity narrative derived from workstation records.

### Suggested First Pilot Workstation

The first pilot station should be one where these conditions hold:

- test activity is common enough to generate useful evidence
- notes and handoffs already matter operationally
- remote expert input is plausibly valuable
- a small number of users can create dense learning quickly

A diagnostics, test, or calibration-linked station is likely a better first candidate than a broadly generalized production environment.

## Build Sequence

### Phase 1

- define canonical object and event schemas
- define role and permission model
- stand up XMPP server and identity model
- build first IPE shell with notes, job context, and messaging

### Phase 2

- build first test-tool integration adapter
- implement artifact capture and attachment flows
- implement append-first event ingestion and audit storage
- build first manager and quality reporting views

### Phase 3

- add AI summary and routing services
- refine local-first sync behavior
- add facility-level exception views and handoff summaries
- prepare node-resident export and inspection workflows

## Open Technical Questions

- Which exact protocols and file formats dominate the first Niagara test-tool integrations?
- Should the first event store be implemented inside the primary relational system or as a separate append-oriented service?
- Which identity provider model best fits node-resident deployments with controlled remote access?
- What retention classes should exist for notes, messages, machine events, and attached artifacts?
- How much federation should the initial XMPP architecture support before there are multiple active nodes?
- Which events are important enough to require immutable retention from day one?
