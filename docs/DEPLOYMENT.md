# Deployment: AI Customer Support Copilot

## Overview

Deployment targets **AWS** for compute, storage, networking, and supporting services. The application uses **Supabase** (hosted PostgreSQL and optional auth) and **Twilio** as external services; credentials and configuration are provided via environment variables and secret managers. This document describes the deployment architecture and expectations without prescribing a single tool (e.g. Terraform vs CDK); align infra code and runbooks with this document.

---

## Tech Stack Recap

| Component | Choice |
|-----------|--------|
| **Web app** | Next.js (Node.js); **Admin** + **User interface**; static export or server run on AWS |
| **Native mobile app** | iOS & Android (e.g. React Native, Swift/Kotlin, Flutter); separate codebase; consumes same FastAPI API; distribution via App Store / Google Play |
| Backend | Python FastAPI; containerized |
| Database | Supabase (PostgreSQL) — external; connection string via env |
| Cache / Queue | Redis — ElastiCache or self-managed on EC2/ECS |
| AI | OpenAI API — key via env/secrets |
| WhatsApp / SMS | Twilio — credentials via env/secrets |
| Email | SendGrid / AWS SES — credentials via env/secrets |
| Object storage | AWS S3 (recordings, uploads, exports) |
| Event streaming | AWS SQS/SNS or optional Amazon MSK (Kafka) |
| Secrets | AWS Secrets Manager or Parameter Store |
| Observability | CloudWatch Logs/Metrics; optional X-Ray; dashboards and alarms |

---

## AWS Architecture (Logical)

### Compute

- **Web app (Next.js):** Serves **Admin** and **User interface** (same app, role-based). Option A: AWS Amplify Hosting (or similar) for static/serverless Next.js. Option B: Containers (ECS Fargate or EKS) behind ALB; Next.js in Node container.
- **Backend:** FastAPI in containers (ECS Fargate or EKS); one or more services (API, workers). Scale by task count or HPA. Used by web app and **native mobile app**.
- **Native mobile app:** Not hosted on AWS; distributed via App Store and Google Play. Consumes backend API; optional push via FCM/APNs and device token registration (see DB_SCHEMA device_tokens).
- **Workers:** Async jobs (ingestion, summarization, SLA checks, notifications) run as separate tasks or Lambda, consuming from SQS or Redis queue.

### Networking

- **VPC:** Private subnets for backend and workers; public subnets for ALB (and optional NAT for outbound).
- **ALB:** Terminate TLS; route to frontend and backend by path or host (e.g. `/api` → FastAPI, `/` → Next.js).
- **WebSocket:** ALB supports WebSocket; ensure sticky sessions or state not tied to single instance for long-lived connections.
- **Outbound:** Backend and workers need outbound access to OpenAI, Twilio, Supabase, email, S3, and optional CRM/webhook endpoints.

### Data and Caching

- **Supabase:** Hosted; connection string (with pooler if used) in secrets; no DB on AWS unless migrating off Supabase later.
- **Redis:** Amazon ElastiCache (Redis) in VPC; connection string via env/Parameter Store; use for cache, rate limit, and queues.
- **S3:** Buckets for recordings, file uploads, exports; IAM roles for app access; lifecycle/retention as per compliance.
- **SQS/SNS:** Queues for async jobs and event streaming; dead-letter queues for failed messages; least-privilege IAM.

### Secrets and Config

- **Secrets Manager (or Parameter Store):** Store Supabase URL/key, OpenAI API key, Twilio credentials, email API key, webhook signing secrets, integration credentials. Rotate per policy.
- **Environment:** Per-environment config (dev/staging/prod): API base URL, feature flags, log level. Prefer env vars or config service over hardcoding.

### DNS and TLS

- **Route 53 (or other DNS):** Point customer-facing domain to ALB or Amplify.
- **ACM:** TLS certificates for ALB; use HTTPS only in production.

---

## Containers

- **Docker:** Multi-stage builds for FastAPI and Next.js; minimal base images; no secrets in layers.
- **Images:** Store in ECR; tag by commit or version; promote same image across environments where possible.
- **Health checks:** HTTP readiness/liveness for FastAPI (e.g. `/health`); same for Next.js if run as server. Use for ECS task health and ALB target health.

---

## CI/CD

- **Pipeline:** On push/merge, run lint, unit tests, and integration tests; build Docker images; push to ECR.
- **Deploy:** Deploy to dev automatically; staging/prod via approval or manual trigger. Use blue/green or rolling updates for API and frontend to avoid downtime.
- **Migrations:** Run Supabase migrations as a step in deployment or via separate migration job; ensure backward compatibility during rollout.

---

## Environments

- **Dev:** Minimal scale; shared Supabase project (or dev DB); Twilio sandbox; low-cost Redis; optional single-instance backend.
- **Staging:** Mirrors production topology; separate Supabase project; test integrations with staging endpoints.
- **Production:** High availability (multi-AZ where applicable); auto-scaling; backup and retention for Redis and S3; Supabase backup per Supabase plan.

---

## Observability and Operations

- **Logs:** Send application logs to CloudWatch Logs; structured format (JSON); log groups per service/environment.
- **Metrics:** CloudWatch Metrics for latency, request count, errors; custom metrics for AI usage, queue depth, ticket/conversation counts.
- **Alarms:** Alarms for high error rate, latency, queue backlog, and critical dependency failures (e.g. Supabase, OpenAI); notify via SNS (email/Slack).
- **Dashboards:** CloudWatch (or third-party) dashboards for traffic, errors, AI usage, and business metrics per docs/PRD and .cursor/rules/45-observability.mdc.

---

## Security and Compliance

- **IAM:** Least-privilege roles for ECS tasks and Lambda; no long-term keys in code.
- **Network:** Restrict security groups to necessary ports; no public access to Redis or queues.
- **Secrets:** No secrets in code or in image layers; use Secrets Manager or Parameter Store with IAM.
- **PII and retention:** Align S3 and Supabase retention with GDPR and internal policy; mask logs and metrics.

---

## External Dependencies

- **Supabase:** Create project and database; run migrations; configure connection pooler and RLS; store URL and keys in AWS Secrets Manager.
- **Twilio:** Account and credentials; configure webhook URLs to point to ALB (e.g. `https://api.<domain>/api/v1/webhooks/twilio`).
- **OpenAI:** API key in secrets; monitor usage and errors.
- **Email provider:** SendGrid or SES; configure domain and keys; inbound webhook URL if used for ticket ingestion.
- **Native mobile:** Build and submit to App Store / Google Play; configure deep links and API base URL per environment; optional FCM/APNs for push. Backend must expose device token registration if push is used (see API_SPEC and DB_SCHEMA).

---

## References

- **PRD:** docs/PRD.md  
- **Architecture:** docs/ARCHITECTURE.md  
- **API:** docs/API_SPEC.md  
- **DB schema:** docs/DB_SCHEMA.md  
- **Rules:** .cursor/rules/90-devops-docker-aws.mdc, 45-observability.mdc, 70-security.mdc
