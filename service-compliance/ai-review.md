# AI Review — Service Layer Compliance

**Status: 🟡 Partial — DB write remains in route**

## Service class
`@clark/ai` package provides `buildReviewStateUpdate()` — business logic extracted.
No dedicated `AIReviewService` in `apps/api/src/services/` yet.

## Remaining violation

### `apps/api/src/routes/v1/ai.ts`
One remaining direct SQL call:
- After calling `buildReviewStateUpdate()`, the route calls `query()` directly to apply
  the UPDATE to `notes` or `messages` tables.

This UPDATE touches data owned by the Notes and Conversations bounded contexts. The correct
fix is for this write to be delegated to `NoteService.applyReviewDecision()` or
`ConversationService.applyReviewDecision()` — whichever context owns the record being
reviewed — not done directly in `ai.ts`.

## What remains
1. Add `applyReviewDecision(actor, update: ReviewStateUpdate)` to `NoteService` and
   `ConversationService`
2. Remove the `query()` import and direct SQL from `routes/v1/ai.ts`
3. Route calls `buildReviewStateUpdate()` then passes the result to the owning service

## Other AI endpoints in `ai.ts`
The generative AI calls (draft note, suggest routing, summarise job) use `@clark/ai`
internally and do not write to the DB directly — those are already compliant.
