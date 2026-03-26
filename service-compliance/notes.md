# Notes — Service Layer Compliance

**Status: 🔴 Violations — service class missing**

## Service class needed
`apps/api/src/services/note-service.ts` — DOES NOT EXIST

## Violations

### `apps/api/src/routes/v1/notes.ts`
Direct SQL in route handlers:
- `GET /v1/notes` — queries `notes_current` view (probably with job/issue filters)
- `POST /v1/notes` — inserts into `notes` with revision chain logic
- Any revision/supersede path

## What NoteService should own
- Data: `notes` table (append-only, revision chain via `revision_chain_id`)
- Domain events: `note.created` (when a note is first posted)
- No CFX messages needed
- Methods: `listNotes(actor, filters)`, `createNote(actor, input)`, `getRevisionChain(chainId)`

## Migration notes
The `notes` table is append-only — "edits" create a new record with `supersedes_note_id`.
NoteService must enforce this invariant (no UPDATE on notes) rather than leaving it implicit
in the route. The `notes_current` view (latest per chain) should be used for list queries.
AI-drafted notes have `review_state = 'pending_review'` — the AI review flow in `ai.ts`
updates this; that update should move to `NoteService.acceptAINote()` / `rejectAINote()`.
