# Architecture: AI Customer Support Copilot

## Purpose

This document describes the high-level architecture for the AI Customer Support Copilot. It aligns with the tech stack (Next.js, FastAPI, Supabase, Redis, OpenAI, Twilio, email provider, AWS) and the feature set in docs/PRD.md. Implementation must place components in the correct layers described here.

---

## High-Level View

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Clients: Admin (web), User UI (web), Native mobile app (iOS/Android),       │
│           Slack/Teams, WhatsApp/SMS, Email                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  API Gateway / Load Balancer (AWS)                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
         ┌────────────────────────────┼────────────────────────────┐
         ▼                            ▼                            ▼
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│  Next.js App    │         │  FastAPI Backend │         │  Webhooks /      │
│  Admin + User   │         │  (REST + WS)    │         │  Inbound         │
│  UI (web)       │         │  ← also used by  │         │  (Twilio, Email, │
│                 │         │  native mobile  │         │  Slack)          │
└────────┬────────┘         └────────┬────────┘         └────────┬────────┘
         │                            │                            │
         │                            │    ┌───────────────────────┘
         │                            ▼    ▼
         │                   ┌─────────────────────────────────────────────┐
         │                   │  Services Layer                               │
         │                   │  (Conversations, Ticketing, RAG, Agents,      │
         │                   │   Channels, NLP, Voice/Video, Automation)     │
         │                   └─────────────────────────────────────────────┘
         │                                    │
         │                                    ▼
         │                   ┌─────────────────────────────────────────────┐
         │                   │  Repositories / Integrations                 │
         │                   │  (Supabase, Redis, Twilio, Email, CRM, etc.)  │
         │                   └─────────────────────────────────────────────┘
         │                                    │
         ▼                                    ▼
┌─────────────────┐         ┌─────────────────────────────────────────────┐
│  Supabase       │         │  External: OpenAI, Twilio, Email, AWS (S3,   │
│  (PostgreSQL +  │         │  SQS/SNS), optional Kafka                    │
│   Auth, RLS)    │         └─────────────────────────────────────────────┘
└─────────────────┘
```

---

## Component Roles

### Interfaces (three)

- **Admin (web):** Served by Next.js; internal users only; AI config, KB, agents, monitoring, API keys. Routes/layout under `/admin/*` with RBAC.
- **User interface (web):** Served by Next.js; end customers and agents; chat, tickets, history, help, voice/video. Same app as Admin, role-based access.
- **Native mobile app:** Separate codebase (iOS/Android); end customers (and optionally agents); chat, tickets, push, voice/video; consumes same FastAPI backend. No admin flows in mobile. See docs/DEPLOYMENT.md for distribution.

### Frontend – Web (Next.js + TypeScript)

- **Responsibilities:** Web UI for **Admin** and **User** interfaces: chat, agent dashboard, admin panel, analytics; real-time updates via WebSocket or SSE; optional voice/video UI.
- **Communication:** REST to FastAPI for CRUD and config; WebSocket for live chat and streaming. Native mobile app uses the same APIs.
- **Layers:** `app/`, `components/`, `features/`, `hooks/`, `lib/`, `services/`, `types/`.

### Backend (Python + FastAPI)

- **Responsibilities:** REST and WebSocket APIs; business logic in services; persistence via repositories; channel adapters (Twilio, email, Slack/Teams); RAG ingestion and retrieval; agent orchestration; ticketing and automation; NLP and voice/video pipelines.
- **Layers:**
  - **Routes:** HTTP and WebSocket entrypoints; thin, validation and dependency injection only.
  - **Services:** Conversations, tickets, RAG, agents, channels, NLP, voice/video, automation.
  - **Repositories:** Supabase (PostgreSQL) access; all DB access through this layer.
  - **Integrations:** Twilio, email provider, Slack/Teams, CRM, Zendesk, Jira; behind adapter interfaces.
  - **Core:** Config, security, logging, middleware.

### Data Stores

- **Supabase (PostgreSQL):** Source of truth for conversations, messages, tickets, knowledge base metadata and chunks (or vector store reference), users, agents, org/tenant, audit logs. Use migrations and RLS where needed.
- **Redis:** Cache (e.g. KB metadata, session), rate limiting, queues for async jobs (e.g. ingestion, notifications, summarization).
- **Vector store:** Either Supabase (pgvector) or dedicated store for RAG embeddings; referenced from backend services only.
- **Object storage (e.g. S3):** Recordings, file uploads, exported documents; metadata in Supabase.

### External Services

- **OpenAI:** Chat completions, embeddings, speech-to-text, text-to-speech.
- **Twilio:** WhatsApp, SMS; webhooks for inbound messages and status.
- **Email provider:** Outbound mail and, if supported, inbound ticket ingestion (e.g. SendGrid, AWS SES).
- **Slack / Teams:** Notifications, agent workflows, optional customer-facing channels.
- **AWS:** Deployment, S3, SQS/SNS (or Kafka) for event streaming, secrets, monitoring.

---

## Cross-Cutting Concerns

- **AuthN/AuthZ:** Supabase Auth or JWT; RBAC for agents, admins, API keys; least privilege.
- **Observability:** Structured logging (request/conversation IDs), metrics (latency, AI usage, errors), dashboards and alerts; see .cursor/rules/45-observability.mdc.
- **Security:** No secrets in code; PII detection/masking; audit logs; secure webhooks; see .cursor/rules/70-security.mdc.
- **Multi-tenancy:** Tenant/organization isolation in schema and RLS; tenant-scoped APIs and caches.

---

## Key Flows (Conceptual)

1. **Omni-channel message in:** Webhook (Twilio/email/Slack) → validate → create/update conversation and message → persist → optional event to queue for NLP/ticketing → notify real-time clients.
2. **Chat (website):** Client ↔ WebSocket ↔ FastAPI → conversation service → RAG/agent → OpenAI → response streamed back; messages stored via repository.
3. **Ticketing:** Conversation or email triggers ticket creation → categorization/NLP → routing → SLA tracking; automation jobs for breach, auto-close, escalation.
4. **Voice:** Inbound call → STT (OpenAI) → text pipeline (RAG/agent) → TTS (OpenAI) → response; recording and summary stored asynchronously.
5. **RAG:** Ingestion job (doc/PDF/URL) → parse → chunk → embed (OpenAI) → store in vector DB; retrieval by conversation/query → context assembly → LLM → cited answer.

---

## Real-Time and Async

- **WebSocket:** Live chat, streaming AI responses, real-time dashboard updates.
- **Event streaming:** Kafka or AWS SQS/SNS for ingestion, notifications, analytics, automation (decoupled, scalable).
- **Workers/Jobs:** Async tasks for embedding, summarization, SLA checks, sync to CRM; use Redis queue or SQS.

---

## References

- **PRD:** docs/PRD.md  
- **API:** docs/API_SPEC.md  
- **Data model:** docs/DB_SCHEMA.md  
- **Deployment:** docs/DEPLOYMENT.md  
- **Rules:** .cursor/rules/*.mdc
