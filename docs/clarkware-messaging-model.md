# Clarkware Messaging Model v1

## Purpose

This document defines the version 1 messaging model for Clarkware. It describes how conversation, presence, AI participation, and event-linked communication should work inside an XMPP-based or strongly XMPP-aligned architecture.

The messaging system is not a side feature. It is part of Clarkware's operational memory system.

## Messaging Principles

### 1. Conversation Must Stay Tied To Work

Messages should usually be linked to a job, issue, workstation, or workspace context.

### 2. Identity Must Be Explicit

Every participant must have a clear identity, including human users, AI agents, and automation services.

### 3. AI Participation Must Be Visible

AI agents should participate as named actors, not as invisible background behavior.

### 4. Messaging Must Support Auditability Without Becoming Rigid

Operational communication needs to be searchable, attributable, and preservable without becoming so formal that people avoid using it.

## Core Messaging Entities

Version 1 messaging should support:

- direct conversations
- workspace channels
- job-linked threads
- issue-linked threads
- AI assist threads
- system alert streams

## Conversation Types

### Direct Conversation

Used for person-to-person or person-to-agent exchanges.

Example ID pattern:

- `dm.person_taylor_01.person_morgan_01`

### Workspace Channel

Used for an ongoing station, bench, line, or team context.

Example ID pattern:

- `ws.ws_testbench_a01`

### Job Thread

Used for communication specific to a job.

Example ID pattern:

- `job.job_2026_niagara_tb_a01_0042`

### Issue Thread

Used for anomalies, exceptions, or escalations.

Example ID pattern:

- `issue.issue_01jws8pxd6w4`

### AI Assist Thread

Used when an AI assistant participates in a bounded support context.

Example ID pattern:

- `ai.job_2026_niagara_tb_a01_0042.summary`

## XMPP-Oriented Model

Version 1 should use XMPP concepts in a practical way:

- JIDs for actor identity
- rooms or room-like constructs for shared threads
- direct messages for bounded exchanges
- presence states for operational availability
- extension payloads or payload-linked references for job, issue, and workstation context

Illustrative JID patterns:

- `person.taylor@niagara.clark`
- `agent.summary01@niagara.clark`
- `svc.adapter.scope01@niagara.clark`
- `room.job_2026_niagara_tb_a01_0042@conference.niagara.clark`

## Message Classes

Version 1 should classify messages for downstream handling.

Allowed message classes:

- `chat`
- `question`
- `alert`
- `handoff`
- `recommendation`
- `resolution`
- `status_update`
- `system_notice`

## Standard Message Payload

Every message should be able to carry:

- `message_id`
- `conversation_id`
- `sender_actor_id`
- `sender_type`
- `message_class`
- `body`
- `created_at`
- `job_id`
- `issue_id`
- `workstation_id`
- `artifact_refs`
- `review_state`
- `reply_to_message_id`
- `metadata`

Example:

```json
{
  "message_id": "msg_01jwsa17cg3p",
  "conversation_id": "job.job_2026_niagara_tb_a01_0042",
  "sender_actor_id": "person_taylor_01",
  "sender_type": "human",
  "message_class": "status_update",
  "body": "Imported first scope run and attached trace image. Investigating drift on channel 2.",
  "created_at": "2026-03-23T13:27:18Z",
  "job_id": "job_2026_niagara_tb_a01_0042",
  "issue_id": null,
  "workstation_id": "ws_testbench_a01",
  "artifact_refs": ["art_scope_trace_001"],
  "review_state": "not_required"
}
```

## Review States

Review state matters most for AI-originated messages, but the field can exist for all messages.

Allowed values:

- `not_required`
- `pending_review`
- `accepted`
- `rejected`
- `edited`

## AI Message Behavior

AI agents should be allowed to:

- summarize recent activity in a thread
- suggest likely next actions
- draft handoff notes
- flag missing evidence
- route alerts to the right human roles

AI messages should always include:

- visible agent identity
- message class
- review state where applicable
- context linkage to the triggering job, issue, or workspace

AI agents should not silently alter historical messages. Corrections should appear as new messages or linked note revisions.

## Presence Model

Version 1 should support operational presence states.

Allowed states:

- `available`
- `heads_down`
- `on_station`
- `in_review`
- `awaiting_response`
- `unavailable`
- `automation_active`

Presence should be scoped where possible. A user may be `on_station` for one workstation context and still reachable in a limited remote-support capacity elsewhere.

## Notification Semantics

Clarkware messaging should support light workflow semantics without becoming a BPM engine.

Key notification types:

- assignment notice
- escalation notice
- waiting-on-response notice
- test exception notice
- quality review request
- remote expert invite
- AI summary available notice

## Context Binding Rules

Version 1 should apply these rules:

- every job thread must include `job_id`
- every issue thread must include `issue_id`
- every workstation channel should include `workstation_id`
- every message with an attached artifact should include `artifact_refs`
- every AI recommendation should include a `review_state`
- every system alert should identify its originating service or adapter

## Event Emission From Messaging

Important messaging actions should also emit Clarkware events.

Examples:

- `conversation.created`
- `message.sent`
- `message.edited`
- `message.reviewed`
- `alert.escalated`
- `presence.changed`
- `ai.recommendation.accepted`
- `ai.recommendation.rejected`

## Room And Access Model

Version 1 room access should be governed by the Clarkware permission model.

Examples:

- a job thread is visible only to actors with rights to that job
- an issue thread can temporarily include a remote expert through an issue-scoped grant
- a workstation channel is visible to authorized facility actors for that station
- an AI assist thread may be visible only to the core participants plus reviewers

## Retention Rules

Version 1 should retain:

- job and issue threads as part of the operational record
- system notices linked to operational decisions
- AI recommendation messages and outcomes
- handoff messages with durable relevance

Version 1 may treat low-value transient presence events with shorter retention than substantive operational messages.

## Suggested XMPP Extension Payload Areas

Clarkware will likely need custom payload support or linked metadata for:

- facility context
- workstation context
- job and issue references
- artifact references
- AI review state
- message class
- actor type

## Niagara Pilot Messaging Design

The first Niagara pilot should implement:

- one workstation channel for the pilot station
- one job-thread model for active diagnostics or test jobs
- one issue-thread escalation path
- one AI summarizer identity
- one remote expert identity path
- one manager or quality-review visibility pattern

## Open Messaging Questions

- Which XMPP server implementation best fits node-resident control and extensibility requirements?
- Which message classes require immutable retention from day one?
- How much of the review-state behavior should live in XMPP payloads versus Clarkware application metadata?
- Which presence states are actually useful to operators versus merely attractive in theory?
