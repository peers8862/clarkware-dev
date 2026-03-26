# Conversations / Messaging — Service Layer Compliance

**Status: 🔴 Violations — service class missing**

## Service class needed
`apps/api/src/services/conversation-service.ts` — DOES NOT EXIST

## Violations

### `apps/api/src/routes/v1/conversations.ts`
Direct SQL in route handlers:
- `GET /v1/conversations` — queries `conversations` + `conversation_participants`
- `POST /v1/conversations` — inserts conversation + initial participant records
- `GET /v1/conversations/:id/messages` — queries `messages` table
- `POST /v1/conversations/:id/messages` — inserts message (should also publish via XMPP)

## What ConversationService should own
- Data: `conversations`, `conversation_participants`, `messages`
- XMPP: should delegate to `@clark/messaging` for actual XMPP stanza delivery
- Domain events: `conversation.created`, `message.sent`
- Methods: `listConversations(actor, filters)`, `createConversation(actor, input)`,
  `getMessages(actor, conversationId, since?)`, `sendMessage(actor, conversationId, body)`

## Migration notes
The current Messages panel in the Theia UI shows WebSocket domain events, not real XMPP
messages. This is a P2 concern — the ConversationService refactor and XMPP wiring can be
done together. For Phase 1, the priority is just removing direct SQL from the route layer.
`@clark/messaging` backend is complete and awaiting wiring.
