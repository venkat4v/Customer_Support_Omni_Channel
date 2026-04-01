# Database Schema: AI Customer Support Copilot

## Overview

The application uses **Supabase (PostgreSQL)** as the primary data store. This document describes the logical schema for the AI Customer Support Copilot. All schema changes must be applied via migrations. Data is consumed by **Admin** (web), **User interface** (web), and **Native mobile app**; auth and RBAC scope access per interface. See docs/PRD.md and .cursor/rules/12-interfaces-admin-user-mobile.mdc.

---

## Conventions

- **Primary keys:** UUID `id` for all main entities.
- **Timestamps:** `created_at`, `updated_at` (timestamptz) on all core tables.
- **Soft delete:** Use `deleted_at` (timestamptz, nullable) where soft delete is required; filter in queries.
- **Tenancy:** Include `tenant_id` (or `organization_id`) where multi-tenant isolation is required; enforce via RLS.
- **Indexes:** Add indexes for foreign keys, common filters (status, channel, tenant, created_at), and search (e.g. full-text on content).

---

## Core Entities

### tenants (organizations)

- **Purpose:** Multi-tenant isolation; one row per customer/org.
- **Key columns:** `id`, `name`, `slug`, `settings` (jsonb), `created_at`, `updated_at`.
- **Notes:** RLS policies scope data by `tenant_id` where applicable.

### users

- **Purpose:** Identity for **admin** (web), **agents** (web), and **end customers** (web + **native mobile**); can link to Supabase Auth or external IdP.
- **Key columns:** `id`, `tenant_id`, `email`, `display_name`, `role` (enum: admin, agent, viewer, etc.), `metadata` (jsonb), `created_at`, `updated_at`.
- **Indexes:** `tenant_id`, `email`, `role`.

### agents

- **Purpose:** Support agents (humans) who handle tickets and conversations.
- **Key columns:** `id`, `tenant_id`, `user_id` (FK), `display_name`, `status` (enum: available, busy, offline), `skills` (jsonb or array), `created_at`, `updated_at`.
- **Indexes:** `tenant_id`, `status`.

### customers

- **Purpose:** End customers (or contacts) who open conversations/tickets.
- **Key columns:** `id`, `tenant_id`, `external_id` (e.g. CRM id), `email`, `phone`, `display_name`, `metadata` (jsonb), `created_at`, `updated_at`.
- **Indexes:** `tenant_id`, `external_id`, `email`.

### conversations

- **Purpose:** Unified thread that can span multiple channels (chat, email, WhatsApp, etc.).
- **Key columns:** `id`, `tenant_id`, `customer_id` (FK), `channel` (enum: chat, email, whatsapp, sms, slack, teams, voice, video), `status` (enum: open, pending, closed), `assigned_agent_id` (FK nullable), `metadata` (jsonb), `created_at`, `updated_at`, `closed_at` (nullable).
- **Indexes:** `tenant_id`, `customer_id`, `status`, `channel`, `assigned_agent_id`, `created_at`.

### messages

- **Purpose:** Single message in a conversation; can be from customer, agent, or system/AI.
- **Key columns:** `id`, `tenant_id`, `conversation_id` (FK), `sender_type` (enum: customer, agent, system, ai), `sender_id` (FK nullable; user/agent or customer), `channel` (enum, same as conversations), `content` (text), `content_type` (e.g. text, html), `metadata` (jsonb; e.g. external_id, attachments), `created_at`, `updated_at`.
- **Indexes:** `conversation_id`, `tenant_id`, `created_at`.
- **Notes:** Attachments may be stored as URLs (e.g. S3) in `metadata`.

### tickets

- **Purpose:** Support ticket; may be created from a conversation or email.
- **Key columns:** `id`, `tenant_id`, `conversation_id` (FK nullable), `customer_id` (FK), `subject`, `status` (enum: open, pending, resolved, closed), `priority` (enum: low, medium, high, urgent), `assigned_agent_id` (FK nullable), `sla_due_at` (nullable), `sla_breach_at` (nullable), `summary` (text, AI-generated), `metadata` (jsonb), `created_at`, `updated_at`, `closed_at` (nullable).
- **Indexes:** `tenant_id`, `customer_id`, `conversation_id`, `status`, `priority`, `assigned_agent_id`, `sla_due_at`, `created_at`.

### ticket_messages (optional)

- **Purpose:** If ticket messages are denormalized or linked separately from conversation messages.
- **Key columns:** `id`, `ticket_id` (FK), `message_id` (FK to messages), `created_at`.
- **Alternative:** Tickets may reference `conversation_id` only and use `messages` filtered by conversation; this table is for explicit ticket–message link if needed.

### knowledge_documents

- **Purpose:** Source document for RAG (help center, FAQ, PDF, etc.).
- **Key columns:** `id`, `tenant_id`, `title`, `source_type` (enum: upload, url, api), `source_ref` (url or path), `status` (enum: pending, processing, ready, failed), `version`, `metadata` (jsonb), `created_at`, `updated_at`, `deleted_at`.
- **Indexes:** `tenant_id`, `status`, `source_type`.

### knowledge_chunks

- **Purpose:** Chunk of text with embedding for vector search.
- **Key columns:** `id`, `tenant_id`, `document_id` (FK), `content` (text), `embedding` (vector type, e.g. pgvector), `chunk_index`, `metadata` (jsonb: page, section, title), `created_at`.
- **Indexes:** `document_id`, `tenant_id`; vector index on `embedding` for similarity search.
- **Notes:** Embedding dimension must match OpenAI embedding model (e.g. 1536); use pgvector or external vector store per architecture.

### customer_context (optional)

- **Purpose:** Cached or synced customer context (purchase history, account info) for personalization.
- **Key columns:** `id`, `tenant_id`, `customer_id` (FK), `source` (e.g. crm, erp), `data` (jsonb), `updated_at`.
- **Indexes:** `customer_id`, `tenant_id`.

---

## Supporting Tables

### audit_logs

- **Purpose:** Audit trail for sensitive actions (RBAC, compliance).
- **Key columns:** `id`, `tenant_id`, `actor_id` (FK nullable), `action`, `resource_type`, `resource_id`, `details` (jsonb), `ip`, `created_at`.
- **Indexes:** `tenant_id`, `actor_id`, `resource_type`, `created_at`.

### api_keys

- **Purpose:** API key management for developer and partner access. **Admin** (web) creates/revokes keys; keys may be used by **User UI**, **mobile**, or external integrations.
- **Key columns:** `id`, `tenant_id`, `name`, `key_hash` (hashed), `scopes` (jsonb or array), `expires_at` (nullable), `created_at`, `last_used_at` (nullable).
- **Indexes:** `tenant_id`, `key_hash` (for lookup).

### integrations_config

- **Purpose:** Per-tenant integration settings (CRM, Zendesk, Slack, webhooks). Managed via **Admin** (web).
- **Key columns:** `id`, `tenant_id`, `provider` (e.g. twilio, sendgrid, salesforce), `config` (jsonb, encrypted or ref to secrets), `enabled` (boolean), `created_at`, `updated_at`.
- **Indexes:** `tenant_id`, `provider`.

### slas (optional)

- **Purpose:** SLA definitions and breach tracking.
- **Key columns:** `id`, `tenant_id`, `name`, `conditions` (jsonb), `due_duration_minutes`, `created_at`, `updated_at`.
- **Indexes:** `tenant_id`.

### voice_calls (optional)

- **Purpose:** Voice call metadata and link to recording/transcript. Used by **User UI** (web) and **native mobile** for call history; **Admin** for monitoring.
- **Key columns:** `id`, `tenant_id`, `conversation_id` (FK nullable), `customer_id` (FK), `agent_id` (FK nullable), `external_id` (e.g. Twilio call SID), `recording_url`, `transcript` (text or FK to transcript table), `summary` (text), `duration_seconds`, `created_at`, `ended_at`.
- **Indexes:** `tenant_id`, `conversation_id`, `external_id`, `created_at`.

### device_tokens (optional, for native mobile)

- **Purpose:** Push notification tokens for **native mobile app** (FCM, APNs). Link to user or customer for targeted push.
- **Key columns:** `id`, `tenant_id`, `user_id` or `customer_id` (FK), `token` (text, hashed or encrypted), `platform` (enum: ios, android), `created_at`, `updated_at`.
- **Indexes:** `tenant_id`, `user_id`/`customer_id`, `platform`.

---

## Enums (PostgreSQL)

Define enums for:

- `channel_enum`: chat, email, whatsapp, sms, slack, teams, voice, video
- `sender_type_enum`: customer, agent, system, ai
- `ticket_status_enum`: open, pending, resolved, closed
- `priority_enum`: low, medium, high, urgent
- `document_status_enum`: pending, processing, ready, failed
- `user_role_enum`: admin, agent, viewer, api

---

## Row Level Security (RLS)

- Enable RLS on tenant-scoped tables; policy: `tenant_id = current_setting('app.current_tenant_id')::uuid` (or equivalent from JWT).
- `audit_logs` and `api_keys`: restrict by tenant and role.
- `knowledge_*`: restrict read/write by tenant; public read may be allowed for certain KB content per product needs.

---

## References

- **PRD:** docs/PRD.md  
- **Architecture:** docs/ARCHITECTURE.md  
- **API:** docs/API_SPEC.md  
- **Deployment:** docs/DEPLOYMENT.md  
- **Rules:** .cursor/rules/30-database-supabase.mdc
