# Clarkware Data Model v1

## Purpose

This document defines the initial canonical data model for Clarkware version 1. It is intended to make the Clarkware concept and architecture concrete enough for product design, backend implementation, interface design, and technical diligence.

The data model is designed around one principle: Clarkware should preserve operational narrative continuity without giving up structured queryability.

## Modeling Principles

### 1. Context Before Abstraction

Most Clarkware records should be meaningful in place. A note, message, event, or artifact should usually be tied to a workstation, job, issue, conversation, or facility context.

### 2. Append-First History

High-accountability actions should generally produce new events rather than destructive overwrites. Current state can be derived, but history should remain reviewable.

### 3. Human, AI, And Automation Are Distinct Actor Types

All three can act in the system, but the model should distinguish them clearly.

### 4. Operational Objects Need Stable IDs

Facilities, workstations, jobs, issues, conversations, artifacts, and actors should all have stable identifiers so records can remain linked over time.

### 5. Evidence Must Be Linkable

Machine outputs, files, images, and logs should be treated as first-class evidence artifacts, not as incidental attachments with weak traceability.

## Canonical Object Set

Version 1 should include these first-class objects:

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
- MachineSession
- Artifact
- Person
- Agent
- AutomationService
- Role
- PermissionGrant

## Object Definitions

### Facility

Represents a physical operating site such as Niagara Assembly.

Core fields:

- `facility_id`
- `name`
- `code`
- `jurisdiction`
- `timezone`
- `status`
- `owner_org`
- `metadata`

Example:

```json
{
  "facility_id": "fac_niagara_001",
  "name": "Niagara Assembly",
  "code": "NIAGARA",
  "jurisdiction": "ON-CA",
  "timezone": "America/Toronto",
  "status": "active"
}
```

### Zone

Represents a sub-area within a facility such as diagnostics, test, calibration, assembly, or receiving.

Core fields:

- `zone_id`
- `facility_id`
- `name`
- `zone_type`
- `status`

### Workstation

Represents a bench, station, line position, test rig, or other local operating context where work is performed.

Core fields:

- `workstation_id`
- `facility_id`
- `zone_id`
- `name`
- `station_type`
- `status`
- `device_profile`
- `integration_profile`

Example:

```json
{
  "workstation_id": "ws_testbench_a01",
  "facility_id": "fac_niagara_001",
  "zone_id": "zone_test_lab",
  "name": "Test Bench A01",
  "station_type": "diagnostics_test",
  "status": "active",
  "integration_profile": ["scope_export", "camera_capture", "firmware_artifacts"]
}
```

### Job

Represents a unit of tracked operational work. A job may correspond to a production batch, test activity, diagnostics task, calibration task, or other bounded piece of work.

Core fields:

- `job_id`
- `facility_id`
- `workstation_id` or `primary_workstation_id`
- `job_type`
- `title`
- `status`
- `priority`
- `customer_ref`
- `product_ref`
- `opened_at`
- `closed_at`
- `current_owner_actor_id`

Example:

```json
{
  "job_id": "job_2026_niagara_tb_a01_0042",
  "facility_id": "fac_niagara_001",
  "primary_workstation_id": "ws_testbench_a01",
  "job_type": "board_diagnostics",
  "title": "Power board validation run",
  "status": "in_progress",
  "priority": "high",
  "customer_ref": "cust_internal_proto",
  "product_ref": "pwr_board_rev_c"
}
```

### Task

Represents a smaller assigned or tracked unit of work inside a job.

Core fields:

- `task_id`
- `job_id`
- `title`
- `status`
- `assigned_actor_id`
- `due_at`
- `sequence_hint`
- `task_type`

### Issue

Represents an operational problem, anomaly, exception, or decision point that requires attention.

Core fields:

- `issue_id`
- `facility_id`
- `job_id`
- `workstation_id`
- `severity`
- `status`
- `issue_type`
- `opened_by_actor_id`
- `opened_at`
- `resolved_at`

### Conversation

Represents a contextual communication space. A conversation can be direct, workspace-linked, job-linked, or issue-linked.

Core fields:

- `conversation_id`
- `conversation_type`
- `facility_id`
- `zone_id`
- `workstation_id`
- `job_id`
- `issue_id`
- `title`
- `status`

Allowed `conversation_type` values in v1:

- `direct`
- `workspace`
- `job`
- `issue`
- `system`
- `ai_assist`

### Message

Represents an individual message inside a conversation.

Core fields:

- `message_id`
- `conversation_id`
- `sender_actor_id`
- `sender_type`
- `message_type`
- `body`
- `created_at`
- `review_state`
- `artifact_refs`
- `job_id`
- `issue_id`
- `workstation_id`

Allowed `message_type` values in v1:

- `note_like`
- `question`
- `alert`
- `handoff`
- `recommendation`
- `resolution`
- `system`

Allowed `review_state` values in v1:

- `not_required`
- `pending_review`
- `accepted`
- `rejected`
- `edited`

### Note

Represents a durable operational note. Notes are distinct from messages because they are part of the long-lived operating record.

Core fields:

- `note_id`
- `author_actor_id`
- `author_type`
- `facility_id`
- `workstation_id`
- `job_id`
- `issue_id`
- `note_type`
- `body`
- `visibility_scope`
- `created_at`
- `revised_at`
- `supersedes_note_id`

Allowed `note_type` values in v1:

- `observation`
- `operator_log`
- `quality_note`
- `test_note`
- `shift_handoff`
- `ai_draft`
- `resolution_note`

### Event

Represents an append-only recorded occurrence.

Core fields:

- `event_id`
- `event_type`
- `facility_id`
- `workstation_id`
- `job_id`
- `issue_id`
- `conversation_id`
- `actor_id`
- `actor_type`
- `source_type`
- `occurred_at`
- `recorded_at`
- `payload`
- `artifact_refs`

Allowed `source_type` values in v1:

- `human_ui`
- `ai_agent`
- `automation_service`
- `tool_adapter`
- `system`

Example:

```json
{
  "event_id": "evt_01jws8k5x0y3",
  "event_type": "test.run.imported",
  "facility_id": "fac_niagara_001",
  "workstation_id": "ws_testbench_a01",
  "job_id": "job_2026_niagara_tb_a01_0042",
  "actor_id": "svc_adapter_scope_01",
  "actor_type": "automation_service",
  "source_type": "tool_adapter",
  "occurred_at": "2026-03-23T13:22:10Z",
  "recorded_at": "2026-03-23T13:22:12Z",
  "payload": {
    "tool_type": "oscilloscope",
    "import_format": "csv",
    "run_label": "validation_pass_1"
  }
}
```

### Tool

Represents an instrument, machine, application, or local utility that can generate evidence or participate in workflow.

Core fields:

- `tool_id`
- `facility_id`
- `workstation_id`
- `tool_type`
- `name`
- `vendor`
- `model`
- `integration_mode`
- `status`

Allowed `integration_mode` values in v1:

- `manual_attach`
- `file_import`
- `watched_folder`
- `api_adapter`
- `serial_adapter`
- `network_adapter`

### MachineSession

Represents a bounded interaction period between a workstation user or job and a tool or machine.

Core fields:

- `machine_session_id`
- `tool_id`
- `workstation_id`
- `job_id`
- `started_at`
- `ended_at`
- `operator_actor_id`
- `session_label`

### Artifact

Represents durable evidence such as files, exports, images, logs, or generated reports.

Core fields:

- `artifact_id`
- `artifact_type`
- `storage_uri`
- `checksum`
- `created_at`
- `created_by_actor_id`
- `facility_id`
- `workstation_id`
- `job_id`
- `issue_id`
- `source_event_id`
- `metadata`

Allowed `artifact_type` values in v1:

- `image`
- `file`
- `test_output`
- `calibration_output`
- `firmware_build`
- `log_export`
- `report`

### Person

Represents a human actor.

Core fields:

- `person_id`
- `display_name`
- `employment_type`
- `primary_role`
- `facility_affiliation`
- `status`

### Agent

Represents an identifiable AI participant.

Core fields:

- `agent_id`
- `display_name`
- `agent_type`
- `operator_org`
- `status`
- `allowed_action_classes`

Allowed `agent_type` values in v1:

- `summarizer`
- `router`
- `retrieval_assistant`
- `drafting_assistant`
- `triage_assistant`

### AutomationService

Represents a non-human non-AI system identity such as an integration adapter or background worker.

Core fields:

- `service_id`
- `display_name`
- `service_type`
- `status`
- `facility_scope`

### Role

Represents a reusable privilege role.

Allowed role examples in v1:

- `owner`
- `facility_admin`
- `supervisor`
- `operator`
- `quality_reviewer`
- `remote_expert`
- `observer`
- `agent_bounded`
- `service_adapter`

### PermissionGrant

Represents a scoped permission assignment.

Core fields:

- `permission_grant_id`
- `actor_id`
- `actor_type`
- `role_id`
- `scope_type`
- `scope_id`
- `permission_list`
- `granted_at`
- `granted_by_actor_id`
- `expires_at`

Allowed `scope_type` values in v1:

- `facility`
- `zone`
- `workstation`
- `job`
- `issue`
- `conversation`

## Relationship Summary

Minimum required relationships:

- `Facility 1 -> many Zones`
- `Facility 1 -> many Workstations`
- `Facility 1 -> many Jobs`
- `Workstation 1 -> many Jobs`
- `Job 1 -> many Tasks`
- `Job 1 -> many Notes`
- `Job 1 -> many Events`
- `Job 1 -> many Artifacts`
- `Issue 1 -> many Notes`
- `Conversation 1 -> many Messages`
- `Tool 1 -> many MachineSessions`
- `MachineSession 1 -> many Events`
- `Actor 1 -> many Notes / Messages / Events`

## Event Taxonomy

Version 1 event families should include:

- workstation events
- job lifecycle events
- task workflow events
- note lifecycle events
- artifact lifecycle events
- conversation and message events
- issue lifecycle events
- tool integration events
- AI action events
- permission and access-control events

Suggested event names:

- `workstation.session.started`
- `workstation.session.ended`
- `job.created`
- `job.started`
- `job.paused`
- `job.closed`
- `task.assigned`
- `task.completed`
- `note.created`
- `note.revised`
- `artifact.attached`
- `conversation.created`
- `message.sent`
- `issue.opened`
- `issue.escalated`
- `issue.resolved`
- `tool.adapter.connected`
- `test.run.imported`
- `calibration.result.imported`
- `ai.summary.generated`
- `ai.recommendation.accepted`
- `ai.recommendation.rejected`
- `permission.granted`

## JSON Envelope Patterns

### Standard Event Envelope

```json
{
  "event_id": "evt_01jws8k5x0y3",
  "event_type": "note.created",
  "facility_id": "fac_niagara_001",
  "workstation_id": "ws_testbench_a01",
  "job_id": "job_2026_niagara_tb_a01_0042",
  "issue_id": null,
  "conversation_id": null,
  "actor": {
    "actor_id": "person_taylor_01",
    "actor_type": "human"
  },
  "source_type": "human_ui",
  "occurred_at": "2026-03-23T13:18:00Z",
  "recorded_at": "2026-03-23T13:18:01Z",
  "payload": {
    "note_id": "note_01jws8jv9a0e",
    "note_type": "observation"
  },
  "artifact_refs": []
}
```

### Standard Message Envelope

```json
{
  "message_id": "msg_01jws8mmd0qe",
  "conversation_id": "conv_job_0042",
  "sender_actor_id": "agent_summary_01",
  "sender_type": "ai_agent",
  "message_type": "recommendation",
  "review_state": "pending_review",
  "job_id": "job_2026_niagara_tb_a01_0042",
  "body": "Three recent anomalies resemble prior voltage drift failures. Recommend connector inspection before rerun.",
  "artifact_refs": ["art_report_0192"],
  "created_at": "2026-03-23T13:24:11Z"
}
```

## Retention And Mutability Guidance

Version 1 should treat these as effectively immutable once recorded, subject only to additive correction events:

- events
- sent messages
- attached artifacts
- AI recommendation outcome records
- permission grant history

Version 1 can allow revision with linked history for:

- notes
- task descriptions
- job metadata
- issue summaries

## Search And Query Priorities

Search should be able to filter by:

- facility
- workstation
- job
- issue
- actor
- event type
- artifact type
- date range
- tool
- message or note content

## Data Model Risks To Avoid

Version 1 should avoid:

- overly generic polymorphic models that make accountability hard to query
- unscoped notes and messages with weak context
- silent mutation of historical records
- AI content without review or attribution metadata
- attachments stored without stable artifact IDs and checksums

## Near-Term Implementation Questions

- Should `Actor` be a unified abstract table with typed specializations, or separate tables joined through a shared identity layer?
- Which event families must be immutable from day one?
- How much denormalized read-model materialization is needed for fast facility dashboards?
- Which artifact metadata fields should be standardized for test outputs in the Niagara pilot?
