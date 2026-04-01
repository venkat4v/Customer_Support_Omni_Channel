# API Specification: AI Customer Support Copilot

## Overview

This document defines the API surface for the AI Customer Support Copilot. The backend is FastAPI (Python). **API consumers:** (1) **Admin** — web (Next.js); (2) **User interface** — web (Next.js); (3) **Native mobile app** — iOS/Android. All three use the same versioned APIs where applicable; admin-only endpoints are protected by RBAC. Third-party integrations (webhooks, SDK) also consume these APIs. See docs/PRD.md and .cursor/rules/12-interfaces-admin-user-mobile.mdc.

---

## Base and Conventions

- **Base URL:** `https://<api-host>/api/v1` (or per environment).
- **Versioning:** URL path `/api/v1`; new breaking changes under `/api/v2`.
- **Auth:** API key (header `X-API-Key` or `Authorization: Bearer <token>`) or JWT (e.g. Supabase); OAuth for third-party integrations where required. **Mobile:** Same auth as web; optional `X-Client: mobile` or `X-App-Version` for analytics; support device/push token registration for notifications.
- **Responses:** JSON; consistent envelope for list (e.g. `data`, `next_cursor`, `total`) and errors (`code`, `message`, optional `details`).
- **Status codes:** 200 OK, 201 Created, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests, 500 Internal Server Error.
- **Rate limiting:** Applied to authenticated endpoints; 429 with `Retry-After` when exceeded.
- **Idempotency:** Support `Idempotency-Key` header for POST/PUT where duplicate submission is a risk (e.g. ticket creation, webhooks).

---

## 1. Support API (Tickets and Conversations)

### Tickets

| Method | Path | Description |
|--------|------|-------------|
| GET | `/tickets` | List tickets (filters: status, channel, agent, tenant; pagination) |
| GET | `/tickets/{id}` | Get ticket by ID (with messages and summary) |
| POST | `/tickets` | Create ticket (idempotency key supported) |
| PATCH | `/tickets/{id}` | Update ticket (status, assignee, priority, tags) |
| GET | `/tickets/{id}/messages` | List messages for ticket (paginated) |
| POST | `/tickets/{id}/messages` | Add message to ticket |
| GET | `/tickets/{id}/summary` | Get AI-generated ticket summary |
| POST | `/tickets/{id}/route` | Trigger or update routing |

### Conversations

| Method | Path | Description |
|--------|------|-------------|
| GET | `/conversations` | List conversations (filters, pagination) |
| GET | `/conversations/{id}` | Get conversation (with thread and channel metadata) |
| POST | `/conversations` | Create conversation (channel, customer ref) |
| PATCH | `/conversations/{id}` | Update conversation (e.g. assign, close) |
| GET | `/conversations/{id}/messages` | List messages (unified thread) |
| POST | `/conversations/{id}/messages` | Add message (channel specified in body) |

### Customer Context

| Method | Path | Description |
|--------|------|-------------|
| GET | `/customers/{id}` | Customer profile and history |
| GET | `/customers/{id}/tickets` | Previous tickets |
| GET | `/customers/{id}/context` | Aggregated context for personalization (tickets, account, optional purchase) |

---

## 2. Chat API (AI Chatbot and Agent Copilot)

### Public / Widget Chat

| Method | Path | Description |
|--------|------|-------------|
| WebSocket | `/ws/chat/{conversation_id}` | Live chat; send user message, receive streamed AI response and citations |
| POST | `/chat/completions` | Synchronous chat turn (optional fallback); returns message and citations |
| GET | `/chat/conversations/{id}/history` | Conversation history for context |

### Agent Copilot

| Method | Path | Description |
|--------|------|-------------|
| POST | `/copilot/suggest-reply` | Get suggested reply for a ticket/conversation (body: ticket_id or conversation_id, optional context) |
| POST | `/copilot/summarize` | Generate or refresh ticket/conversation summary |
| POST | `/copilot/retrieve-kb` | Retrieve KB snippets for a query (for agent UI) |
| POST | `/copilot/troubleshooting-steps` | Get recommended troubleshooting steps for a topic or ticket |

All responses must be structured (e.g. `suggested_reply`, `summary`, `snippets`, `steps`) and optionally include confidence or source attribution.

---

## 3. Voice API

| Method | Path | Description |
|--------|------|-------------|
| POST | `/voice/transcribe` | Submit audio; return transcript (OpenAI STT) |
| POST | `/voice/synthesize` | Submit text; return audio URL or stream (OpenAI TTS) |
| GET | `/voice/calls/{id}` | Get call metadata, transcript, summary |
| POST | `/voice/calls` | Register call (e.g. after Twilio webhook); return call id for later association |
| POST | `/voice/calls/{id}/summary` | Request or retrieve call summary (async job) |

Recording storage and playback URLs are returned as signed or scoped URLs (e.g. S3 presigned).

---

## 4. Knowledge Base and Ingestion

| Method | Path | Description |
|--------|------|-------------|
| GET | `/knowledge/documents` | List documents (with version and status) |
| POST | `/knowledge/documents` | Upload document (PDF, etc.); triggers async ingestion |
| GET | `/knowledge/documents/{id}` | Document metadata and status |
| DELETE | `/knowledge/documents/{id}` | Soft delete or remove document and chunks |
| POST | `/knowledge/ingest-url` | Submit URL for scraping and ingestion |
| GET | `/knowledge/collections` | List collections/sources (e.g. help center, FAQ) |
| POST | `/knowledge/collections` | Create collection and optional sync rule |

Ingestion is asynchronous; status and errors are available via document or job endpoints.

---

## 5. Admin and Configuration

| Method | Path | Description |
|--------|------|-------------|
| GET | `/admin/agents` | List agents (and status) |
| PATCH | `/admin/agents/{id}` | Update agent (role, skills, availability) |
| GET | `/admin/config/ai` | AI model and RAG config (read) |
| PATCH | `/admin/config/ai` | Update AI model config (model, temperature, limits) |
| GET | `/admin/api-keys` | List API keys (masked) |
| POST | `/admin/api-keys` | Create API key (return raw key once) |
| DELETE | `/admin/api-keys/{id}` | Revoke API key |
| GET | `/admin/usage` | Usage analytics (tokens, requests, by tenant) |

All admin endpoints require elevated RBAC (admin role). **Consumers:** Admin web UI only; not used by User UI or native mobile app.

---

## 6. Webhooks (Outbound)

- **Events:** e.g. `ticket.created`, `ticket.updated`, `conversation.closed`, `message.received`.
- **Delivery:** HTTP POST to subscriber URL; signature header for verification; retries with backoff.
- **Configuration:** Subscribers and event types managed via admin or config API; secrets stored securely.

---

## 7. Inbound Webhooks (Integration Points)

- **Twilio (WhatsApp/SMS):** POST to `/webhooks/twilio`; validate Twilio signature; parse body and create/update conversation and message; return TwiML if needed.
- **Email (inbound):** POST to `/webhooks/email` or provider-specific endpoint; parse and create ticket/conversation; idempotency by message-id.
- **Slack/Teams:** POST to `/webhooks/slack` and `/webhooks/teams` for events; validate signing secret; process and respond per PRD.

All webhook handlers must validate payloads, be idempotent where possible, and return 2xx quickly; heavy work offloaded to queue.

---

## 7b. Mobile-specific (Native app)

- **Device/push tokens:** POST `/users/me/device-tokens` or `/customers/{id}/device-tokens` to register FCM/APNs token for push notifications; optional `X-Client: mobile`, `X-App-Version`. Used by **native mobile app** only.
- **Deep links:** Document deep-link scheme (e.g. `yourapp://chat/{conversation_id}`) for opening conversations from push; backend may expose link generation in API response where relevant.
- All other mobile needs (chat, tickets, voice, auth) are covered by the same Support, Chat, and Voice APIs; use RBAC and standard auth.

---

## 8. Analytics and Monitoring (Optional API)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/analytics/dashboard` | Aggregates: CSAT, resolution time, first response, sentiment, AI vs human resolution |
| GET | `/analytics/agents` | Agent performance metrics |
| GET | `/analytics/calls` | Call quality and duration metrics |

Filtering by tenant, time range, and channel; used by **Admin** dashboard and reporting. **User UI** and **mobile** use support/chat/ticket endpoints only; **admin** additionally uses admin and analytics endpoints.

---

## References

- **PRD:** docs/PRD.md  
- **Architecture:** docs/ARCHITECTURE.md  
- **DB schema:** docs/DB_SCHEMA.md  
- **Deployment:** docs/DEPLOYMENT.md  
- **Rules:** .cursor/rules/10-backend-fastapi.mdc, 15-omni-channel.mdc, 95-integrations.mdc
