# RFQX Build Plan

## AI RFQ & Quoting Intelligence for Garment Manufacturers

**Product:** RFQX (part of fabricXai platform)
**Owner:** fabricXai / SocioFi Technology Ltd.
**Date:** March 2026

---

## 1. Project Overview

RFQX is a multi-tenant SaaS platform that automates the garment manufacturing RFQ-to-quote pipeline. It ingests RFQs from email, buyer portals, WhatsApp, and file uploads, uses AI (Claude) to extract structured specs from unstructured documents, calculates costs against a tenant-specific cost library, generates buyer-formatted quotes, and tracks quote lifecycle through to win/loss.

**QUINN** is the conversational AI copilot (chat panel on every page) that orchestrates all agent actions via natural language.

### Core Pipeline

1. **Capture** - Monitor email/portal/WhatsApp for incoming RFQs
2. **Extract** - AI parses documents into structured garment specs with per-field confidence scoring
3. **Cost** - Apply tenant cost library to build line-item cost breakdown (11 cost categories)
4. **Quote** - Generate buyer-formatted PDF/Excel quote with margin control and incoterm adjustments (FOB/CIF/EXW/DDP)
5. **Send & Track** - Deliver quote, detect opens, auto follow-up on escalation schedule (4h -> 48h -> 72h, max 3)
6. **Learn** - Win rate, buyer patterns, pricing trends analytics

### Key Metrics

- Quote turnaround: 4+ hours -> under 30 minutes
- Target: 50 paying factories (Bangladesh), $24,950 MRR by Month 12
- Tiers: Starter ($299/mo, 20 RFQs, 2 users), Growth ($599/mo, 75 RFQs, 10 users), Scale ($999/mo, unlimited)

### Target Users

| Role | Primary Use |
|---|---|
| Factory Owner / Director | Dashboard overview, win rate, revenue pipeline |
| Quotation Manager / Costing Head | Reviewing and approving AI-generated quotes |
| Merchandiser | Managing RFQ inbox, coordinating with buyers |
| Commercial Executive | Sending quotes, tracking buyer responses |
| Production Manager | Reviewing capacity before quote confirmation |

---

## 2. Tech Stack

| Layer | Technology | Version/Notes |
|---|---|---|
| **Frontend** | Next.js (App Router) | 14.x, TypeScript 5.x |
| **CSS** | Tailwind CSS | 3.x |
| **UI Components** | shadcn/ui | latest |
| **Backend** | Next.js API Routes + Node.js Workers | Node 20 LTS |
| **Database** | Supabase (PostgreSQL 15 + pgvector) | RLS enabled, UUIDs |
| **Auth** | Supabase Auth | Email/password, Google OAuth, Magic Link |
| **File Storage** | Supabase Storage | S3-backed, tenant-scoped buckets |
| **Cache** | Redis (Upstash or Railway) | 7.x |
| **Queue** | BullMQ over Redis | 5.x, 7 named queues |
| **AI / LLM** | Anthropic Claude claude-opus-4-5 | Varied temperatures per agent |
| **OCR** | Google Cloud Vision API + Tesseract.js fallback | |
| **PDF Generation** | @react-pdf/renderer + Puppeteer | Server-side |
| **Email Integration** | Nylas API (Gmail/Outlook unified) or imapflow | |
| **Transactional Email** | Resend | Invites, notifications, password reset |
| **Real-time** | Supabase Realtime (WebSocket) | Notification badges, live updates |
| **Billing** | Stripe | Subscriptions + webhooks |
| **Monitoring** | Sentry + Vercel Analytics | Error tracking + performance |
| **Testing** | Vitest (unit) + Playwright (e2e) | |
| **Logging** | pino | Structured JSON |
| **Validation** | Zod | All API request bodies |
| **State Management** | zustand | QUINN panel state |
| **Charts** | recharts | Analytics dashboards |
| **Deployment (Frontend)** | Vercel | Auto-scaling serverless |
| **Deployment (Workers)** | Railway | 3 worker processes |

---

## 3. Data Models

### 3.1 Core Entities

#### tenants
Multi-tenant factory accounts. Stores plan, billing, default costing percentages.
```
id (UUID PK), name, slug (unique), plan (starter|growth|scale),
base_currency, max_rfqs_per_month, max_users, logo_url, letterhead_url,
address, phone, website, default_margin_percentage, default_incoterm,
default_quote_validity_days, overhead_percentage, commission_percentage,
finance_cost_percentage, stripe_customer_id, subscription_status,
trial_ends_at, created_at, updated_at
```

#### users
Factory team members linked to Supabase Auth. 6 roles with RBAC.
```
id (UUID PK), tenant_id (FK->tenants), email, full_name,
role (owner|admin|quotation_manager|merchandiser|commercial_executive|viewer),
avatar_url, phone, is_active, mfa_enabled, notification_preferences (JSONB),
last_login_at, supabase_auth_id (unique), created_at, updated_at
UNIQUE(tenant_id, email)
```

#### buyers
Buyer profiles with portal credentials and template preferences.
```
id (UUID PK), tenant_id (FK->tenants), name, code, contact_name,
contact_email, contact_phone, country, tier (1-5),
preferred_incoterm, preferred_currency, payment_terms,
quote_template_id (FK->quote_templates), portal_type,
portal_credentials_encrypted, commission_percentage, notes, is_active,
created_at, updated_at
UNIQUE(tenant_id, code)
```

#### rfqs
Core RFQ entity with 12 statuses, AI-extracted data, and linkages.
```
id (UUID PK), tenant_id (FK->tenants), rfq_number (auto: RFQ-YYYYMM-XXXX),
buyer_id (FK->buyers), assigned_user_id (FK->users),
status (new|extracting|extracted|costing|review|approved|sent|followed_up|won|lost|expired|archived),
priority (1-5), source_channel (email|portal|whatsapp|upload|api),
source_reference, source_email_from, source_email_subject, original_language,
extracted_data (JSONB - ExtractedSpecs: 18 fields with per-field confidence scores),
extraction_confidence_avg, fields_flagged_for_review (TEXT[]),
manual_overrides (JSONB), product_type, total_quantity, estimated_value_usd,
deadline, delivery_date, incoterm, dedup_hash (SHA-256),
merged_with_rfq_id, attachments (TEXT[]), notes, won_lost_reason,
created_at, updated_at, extracted_at, sent_at, responded_at, closed_at
```

#### quotes
Quote entity with full cost breakdown, versioning, and buyer communication tracking.
```
id (UUID PK), tenant_id (FK->tenants), rfq_id (FK->rfqs),
quote_number (auto: QT-YYYYMM-XXXX), version, status (draft|pending_review|approved|sent|revised|accepted|rejected|expired),
cost_breakdown (JSONB - CostBreakdown: 11 cost categories with per_unit/total/formula),
cost_snapshot_id (FK->cost_library_snapshots), currency,
exchange_rate_to_usd, exchange_rate_locked, total_quantity,
cost_per_unit, total_cost, margin_percentage, margin_per_unit,
fob_per_unit, fob_total, incoterm, freight_per_unit, landed_price_per_unit,
buyer_template_id, pdf_url, excel_url, validity_days, valid_until,
sent_via, sent_at, opened_at, buyer_counter_offer (JSONB),
scenarios (JSONB - up to 3), approved_by (FK->users), approved_at,
created_by (FK->users), created_at, updated_at
```

#### cost_library_items
Tenant-specific cost entries across all 11 categories with category-specific fields.
```
id (UUID PK), tenant_id (FK->tenants),
category (fabric|cmt|trims|embellishment|washing|testing|packing|overhead|freight|commission|finance),
name, code, unit, cost, currency, supplier_name, supplier_code,
-- Fabric: fabric_composition, fabric_gsm, fabric_construction, fabric_width_cm,
           consumption_per_unit_meters, consumption_per_unit_kg
-- CMT: product_type, complexity_tier, sam_minutes, moq_min, moq_max
-- Washing: wash_type
-- Freight: origin_port, destination_port
effective_from, effective_to, is_active, notes, created_at, updated_at
```

### 3.2 Supporting Entities

| Table | Purpose |
|---|---|
| `cost_library_snapshots` | Point-in-time cost library copies for quote traceability |
| `quote_templates` | Buyer-specific quote format configs (PDF/Excel layout, sections, field order, styling) |
| `inbox_connections` | Gmail/Outlook/IMAP/portal OAuth tokens (AES-256-GCM encrypted) |
| `inbox_sync_state` | Last processed email UID/history ID per connection |
| `rfq_attachments` | File metadata + OCR text per uploaded file |
| `exchange_rates` | Currency conversion rates (manual or API-sourced) for USD, BDT, EUR, GBP, VND, INR |
| `notifications` | In-app + email notification records per user (13 notification types) |
| `audit_log` | Full action trail: entity, changes JSON (old/new), user, IP, timestamp |
| `quinn_conversations` | Chat history per user session with context refs (page, rfq_id, quote_id) |
| `auto_routing_rules` | Rules for auto-assigning RFQs by buyer/product/priority/channel (Growth+) |
| `follow_up_templates` | Email follow-up templates with 12 variable substitutions |

### 3.3 Entity Relationships

```
tenants 1--N users, buyers, rfqs, quotes, cost_library_items, inbox_connections, ...
buyers 1--N rfqs, 1--1 quote_templates (preferred)
rfqs 1--N quotes, 1--N rfq_attachments, N--1 users (assigned)
quotes N--1 rfqs, N--1 users (created_by, approved_by), N--1 quote_templates
quotes N--1 cost_library_snapshots
inbox_connections 1--1 inbox_sync_state
```

### 3.4 Row Level Security

All 17 tables enforce tenant isolation via RLS:
```sql
CREATE POLICY tenant_isolation ON <table>
  USING (tenant_id = (SELECT tenant_id FROM users WHERE supabase_auth_id = auth.uid()));
```

---

## 4. Agent Architecture

### 4.1 Orchestration Model

Code-orchestrated (deterministic TypeScript router), NOT autonomous agent loops.

```
User / QUINN Chat
       |
  [Orchestrator] -- deterministic router (TypeScript)
       |
  +----+--------+----------+-----------+
  v    v        v          v           v
Inbox  Extractor CostEngine QuoteWriter Tracker
Agent  Agent     Agent      Agent      Agent
```

### 4.2 Sub-Agent Definitions

| Agent | Trigger | Tools | Skills | Memory |
|---|---|---|---|---|
| **InboxWatcher** | Cron every 60s (BullMQ) | `fetch_emails`, `check_portal`, `parse_whatsapp_webhook` | Email classification, language detection, dedup hash | Last processed email UID per inbox (`inbox_sync_state` table) |
| **ExtractorAgent** | New RFQ (status=extracting) | `ocr_document`, `parse_pdf`, `parse_excel`, `parse_email_body` | Garment spec extraction (active), Footwear, Home Textile | Correction history per tenant (few-shot augmentation after >10 overrides in 30 days) |
| **CostEngine** | Extraction approved / auto (>=85% confidence) | `lookup_cost_library`, `get_exchange_rate`, `calculate_cmt`, `calculate_fabric_cost` | FOB/CIF/EXW/DDP calculation, margin calc, reverse calc | Last used cost snapshot per tenant (`cost_library_snapshots` table) |
| **QuoteWriter** | User approves cost breakdown | `get_buyer_template`, `generate_pdf`, `generate_excel`, `get_factory_letterhead` | Template formatting, multi-language quote generation | Buyer template preferences (`buyers` table) |
| **TrackingAgent** | Quote sent, then every 30min | `check_email_opens`, `check_portal_status`, `send_follow_up`, `update_quote_status` | Follow-up email generation, win/loss analysis | Follow-up count + last timestamp per quote |

### 4.3 Agent Prompts

**InboxWatcher**: "You are an RFQ detection classifier for garment manufacturing. Given an email subject, body snippet, and attachment filenames, determine if this is an RFQ. Return JSON: {is_rfq, confidence, buyer_name, product_hint, language}. Only classify as RFQ if the message requests pricing, costing, or quotation for a garment product."

**ExtractorAgent**: "You are a garment specification extraction engine. Given raw text from an RFQ document, extract all product specifications into structured JSON. Use exact garment industry terminology. For every field, provide a confidence score (0-100). Fields: product_type, style_number, season, fabric_composition, fabric_gsm, fabric_construction, yarn_count, colorways, total_quantity, size_breakdown, delivery_date, delivery_port, incoterm, payment_terms, buyer_reference, certifications, testing_requirements, packaging_specs, print_embroidery_details. If a field is not found in the source, set it to null with confidence 0."

**CostEngine**: "You are a garment costing engine. Given extracted product specs and a cost library, calculate a complete cost breakdown. Apply these rules strictly: 1) Fabric cost = consumption * cost_per_unit. 2) CMT = rate card lookup by product_type + quantity bracket. 3) Trims = sum of individual items. 4) Washing = rate per unit by wash type. 5) Embellishment = stitch/color-based. 6) Packing = per-unit rate. 7) Overhead = % of CMT. 8) Testing = total / quantity. 9) Commission = % of subtotal. 10) Finance = % of subtotal. 11) FOB = sum of all. Show every step. All amounts in USD unless specified."

**QuoteWriter**: "You are a professional quote document writer for garment manufacturing. Given a cost breakdown and buyer template specification, generate a formatted quote document. Include factory name, buyer name, style reference, product description, quantity, size breakdown, fabric details, FOB/CIF price per unit, total order value, delivery terms, payment terms, validity period (default 15 days). Be professional and concise. Match buyer template format exactly."

**TrackingAgent**: "You are a professional follow-up assistant for garment factory sales. Generate polite, concise follow-up messages for quote responses. Reference the specific quote, buyer name, product, and quantity. Keep under 100 words. Tone: professional, warm, not pushy."

### 4.4 QUINN Copilot

- **Interface**: Chat panel (right sidebar 400px, full-screen mobile), available on every page
- **LLM Config**: Claude claude-opus-4-5, temp=0.4, max_tokens=1024
- **Context**: Last 20 messages + current page context (active RFQ/quote injected as system context)
- **Tool Routing**: Access to all sub-agent tools via MCP-style definitions
- **System Prompt**: "You are QUINN, the AI copilot for RFQX -- a quoting platform for garment manufacturers. You help factory teams manage RFQs and quotes through natural conversation. You can: view RFQ details, build quotes, assign RFQs to team members, send quotes, check analytics, and answer questions about the platform. Always be concise (under 300 words). When taking actions, confirm with the user before executing. You understand English, Bengali, Hindi, Vietnamese, and Turkish. Use garment industry terminology naturally (FOB, CMT, GSM, AQL, TNA, BOM). Format responses with markdown tables and action buttons where helpful."
- **Streaming**: SSE with event types: `text`, `tool_call`, `tool_result`, `action_button`, `done`

**10 QUINN Tools**:
| Tool | Description | Params |
|---|---|---|
| `list_rfqs` | List RFQs with filters | status?, buyer?, priority?, assigned_to?, date_range? |
| `get_rfq_details` | Full RFQ details | rfq_id |
| `build_quote` | Trigger cost calc + quote | rfq_id, margin_percentage |
| `assign_rfq` | Assign to team member | rfq_id, user_id |
| `send_quote` | Send approved quote | quote_id, channel |
| `get_analytics` | Retrieve analytics | metric, date_range?, buyer_id? |
| `update_cost_library` | Update cost entry | item_id, new_value |
| `search_quotes` | Search historical quotes | query, filters? |
| `set_reminder` | Set follow-up reminder | quote_id, remind_at |
| `get_buyer_info` | Buyer profile + history | buyer_id |

**Quick Actions**: `/quote`, `/status`, `/report`, `/assign`, `/follow-up`, `/help`

### 4.5 Agent Memory

| Agent | Memory Type | Storage |
|---|---|---|
| InboxWatcher | Last processed email UID per inbox | `inbox_sync_state` table |
| ExtractorAgent | Correction history per tenant (few-shot augmentation after >10 overrides in 30 days) | `rfq.manual_overrides` aggregated |
| CostEngine | Last used cost snapshot per tenant | `cost_library_snapshots` table |
| QuoteWriter | Buyer template preferences | `buyers` table |
| TrackingAgent | Follow-up count + last timestamp per quote | `quotes` table + notifications |
| QUINN | Conversation history (20 msgs) + page context | `quinn_conversations` table |

### 4.6 LLM Configuration Summary

| Agent | Model | Temperature | Max Output Tokens | Retry | Timeout |
|---|---|---|---|---|---|
| InboxWatcher | claude-opus-4-5 | 0.2 | 1,024 | 3x exponential (1s, 2s, 4s) | 60s |
| ExtractorAgent | claude-opus-4-5 | 0.1 | 4,096 | 2x exponential | 60s |
| CostEngine | claude-opus-4-5 | 0.0 | 2,048 | 2x exponential | 60s |
| QuoteWriter | claude-opus-4-5 | 0.3 | 2,048 | 2x exponential | 60s |
| TrackingAgent | claude-opus-4-5 | 0.4 | 512 | 2x exponential | 60s |
| QUINN | claude-opus-4-5 | 0.4 | 1,024 | 3x exponential | 60s |

---

## 5. API Endpoints

All endpoints prefixed with `/api/v1`. Auth via Supabase JWT in `Authorization: Bearer <token>`. Tenant context derived from authenticated user's `tenant_id`.

### 5.1 Authentication

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/auth/signup` | No | Create tenant + owner user |
| POST | `/auth/login` | No | Email/password login |
| POST | `/auth/refresh` | No | Refresh JWT tokens |
| POST | `/auth/logout` | Yes | End session |
| POST | `/auth/forgot-password` | No | Send reset email |
| POST | `/auth/reset-password` | No | Reset with token |
| GET | `/auth/callback/google` | No | Google OAuth callback |
| GET | `/auth/callback/microsoft` | No | Microsoft OAuth callback |

### 5.2 RFQs

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/rfqs` | All | List RFQs (paginated, filterable by status/buyer/priority/assigned/date/search, sortable) |
| GET | `/rfqs/:id` | All | RFQ detail with extracted data, attachments, quotes |
| POST | `/rfqs` | owner, admin, qm, merch | Create RFQ (multipart file upload, plan limit enforced) |
| PATCH | `/rfqs/:id` | owner, admin, qm, merch | Update RFQ fields/status (transitions enforced) |
| POST | `/rfqs/:id/extract` | owner, admin, qm | Trigger AI extraction (returns job_id, async via BullMQ) |
| PATCH | `/rfqs/:id/extracted-data` | owner, admin, qm | Override extracted fields (logged in manual_overrides) |
| POST | `/rfqs/:id/assign` | owner, admin, qm | Assign to user |
| POST | `/rfqs/bulk-action` | owner, admin, qm | Bulk assign/archive/priority (max 50) |
| DELETE | `/rfqs/:id` | owner, admin | Soft-delete (archive) |

### 5.3 Quotes

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/quotes` | All | List quotes (paginated, filterable) |
| GET | `/quotes/:id` | All | Quote detail with cost breakdown + version history |
| POST | `/quotes` | owner, admin, qm | Build quote from RFQ (triggers CostEngine, returns 422 if extraction incomplete) |
| PATCH | `/quotes/:id` | owner, admin, qm | Update margin/incoterm/details (recalculates costs) |
| POST | `/quotes/:id/approve` | owner, admin, qm | Approve quote |
| POST | `/quotes/:id/send` | owner, admin, qm, ce | Send to buyer via email/portal |
| POST | `/quotes/:id/revise` | owner, admin, qm | Create new version (previous set to "revised") |
| POST | `/quotes/:id/generate-pdf` | owner, admin, qm | Generate PDF (signed URL, 1hr expiry) |
| POST | `/quotes/:id/scenarios` | owner, admin, qm | Compare up to 3 scenarios (Growth+) |

### 5.4 Cost Library

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/cost-library` | owner, admin, qm | List items by category (paginated, searchable) |
| GET | `/cost-library/:id` | owner, admin, qm | Item detail |
| POST | `/cost-library` | owner, admin, qm | Add item |
| PATCH | `/cost-library/:id` | owner, admin, qm | Update item |
| DELETE | `/cost-library/:id` | owner, admin | Soft-delete (deactivate) |
| POST | `/cost-library/import` | owner, admin, qm | Import from Excel/CSV with column mapping |
| POST | `/cost-library/bulk-update` | owner, admin | Bulk percentage/absolute price update (Growth+) |
| POST | `/cost-library/snapshot` | owner, admin | Create point-in-time snapshot |

### 5.5 Buyers

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/buyers` | All | List buyers with computed stats (rfq_count, win_rate) |
| GET | `/buyers/:id` | All | Buyer profile + history + stats |
| POST | `/buyers` | owner, admin, qm, merch | Create buyer |
| PATCH | `/buyers/:id` | owner, admin, qm | Update buyer |

### 5.6 Users & Team

| Method | Endpoint | Roles | Description |
|---|---|---|---|
| GET | `/users` | owner, admin | List team members with workload counts |
| POST | `/users/invite` | owner, admin | Invite user (plan user limit enforced) |
| PATCH | `/users/:id` | owner, admin | Update role/status/notification preferences |
| DELETE | `/users/:id` | owner | Deactivate user (soft delete) |

### 5.7 QUINN Copilot

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/quinn/chat` | Yes | Chat message (SSE streaming: text, tool_call, tool_result, action_button, done) |
| GET | `/quinn/conversations` | Yes | List conversations |
| GET | `/quinn/conversations/:id` | Yes | Conversation history |

### 5.8 Analytics

| Method | Endpoint | Tier | Description |
|---|---|---|---|
| GET | `/analytics/overview` | All | KPI summary (win rate, response time, volume, pipeline, forecast) |
| GET | `/analytics/win-rate` | All | Win rate breakdown by buyer/product/month + 12mo trend |
| GET | `/analytics/response-time` | All | Avg/median hours + per-user breakdown |
| GET | `/analytics/pipeline` | Growth+ | Pipeline value by status + by buyer + expected close |
| GET | `/analytics/buyer-scorecard` | Growth+ | Per-buyer performance metrics + volume trend |
| GET | `/analytics/cost-trends` | Growth+ | Cost category trends + margin erosion alert |
| GET | `/analytics/export` | All | Download report as XLSX/CSV |

### 5.9 Supporting Endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET/POST/DELETE | `/inbox-connections/*` | Manage Gmail/Outlook/IMAP connections (OAuth + credentials) |
| GET/PATCH/POST | `/notifications/*` | Notification list, mark read, mark all read |
| GET/PATCH | `/tenant` | Tenant settings (plan, defaults, branding) |
| POST | `/tenant/logo`, `/tenant/letterhead` | Upload branding assets (2MB/5MB limits) |
| GET | `/audit-log` | Audit trail (filterable, exportable CSV) |
| GET/POST | `/exchange-rates` | Currency rates (manual entry) |
| POST | `/upload` | File upload (25MB limit, type validated) |
| GET | `/upload/:id/signed-url` | Signed download URL (1hr expiry) |
| CRUD | `/routing-rules` | Auto-routing rules (Growth+) |
| GET/POST | `/follow-up-templates` | Follow-up email templates (12 variables) |
| POST | `/webhooks/whatsapp` | WhatsApp incoming webhook |
| POST | `/webhooks/email` | Nylas email webhook |
| POST | `/webhooks/stripe` | Stripe subscription webhook |

### 5.10 Rate Limits

| Endpoint Group | Limit | Scope |
|---|---|---|
| Auth endpoints | 10/min | Per IP |
| QUINN chat | 30/min | Per user |
| File upload | 20/min | Per user |
| All other API | 100/min | Per user |
| Scale tier API | 1000/min | Per tenant |

---

## 6. File Structure

```
rfqx/
+-- .github/
|   +-- workflows/
|       +-- ci.yml                    # Lint, typecheck, test, build
|       +-- deploy.yml                # Vercel + Railway deploy
+-- public/
|   +-- images/                       # Static assets
+-- src/
|   +-- app/                          # Next.js App Router
|   |   +-- (auth)/
|   |   |   +-- login/page.tsx
|   |   |   +-- signup/page.tsx
|   |   |   +-- forgot-password/page.tsx
|   |   |   +-- layout.tsx            # Auth layout (centered card)
|   |   +-- (dashboard)/
|   |   |   +-- dashboard/page.tsx
|   |   |   +-- inbox/
|   |   |   |   +-- page.tsx          # RFQ list/board view
|   |   |   |   +-- [rfqId]/page.tsx  # RFQ detail
|   |   |   +-- quotes/
|   |   |   |   +-- page.tsx          # Quote list
|   |   |   |   +-- [quoteId]/page.tsx    # Quote detail/builder
|   |   |   |   +-- [quoteId]/preview/page.tsx
|   |   |   +-- cost-library/page.tsx
|   |   |   +-- buyers/
|   |   |   |   +-- page.tsx
|   |   |   |   +-- [buyerId]/page.tsx
|   |   |   +-- analytics/page.tsx
|   |   |   +-- notifications/page.tsx
|   |   |   +-- settings/
|   |   |   |   +-- page.tsx          # Redirect to profile
|   |   |   |   +-- profile/page.tsx
|   |   |   |   +-- team/page.tsx
|   |   |   |   +-- integrations/page.tsx
|   |   |   |   +-- templates/page.tsx
|   |   |   |   +-- routing/page.tsx
|   |   |   |   +-- notifications/page.tsx
|   |   |   |   +-- billing/page.tsx
|   |   |   |   +-- audit-log/page.tsx
|   |   |   +-- layout.tsx            # Dashboard layout (sidebar + topbar + QUINN)
|   |   +-- onboarding/page.tsx
|   |   +-- api/
|   |   |   +-- v1/
|   |   |       +-- auth/             # signup, login, refresh, logout, forgot/reset, callbacks
|   |   |       +-- rfqs/             # CRUD, extract, assign, bulk-action
|   |   |       +-- quotes/           # CRUD, approve, send, revise, pdf, scenarios
|   |   |       +-- cost-library/     # CRUD, import, bulk-update, snapshot
|   |   |       +-- buyers/           # CRUD
|   |   |       +-- users/            # list, invite, update, delete
|   |   |       +-- quinn/            # chat (SSE), conversations
|   |   |       +-- analytics/        # overview, win-rate, response-time, pipeline, etc.
|   |   |       +-- inbox-connections/ # gmail, outlook, imap
|   |   |       +-- notifications/    # list, mark-read
|   |   |       +-- tenant/           # settings, logo, letterhead
|   |   |       +-- exchange-rates/
|   |   |       +-- upload/           # file upload + signed URLs
|   |   |       +-- audit-log/        # list, export
|   |   |       +-- routing-rules/
|   |   |       +-- follow-up-templates/
|   |   |       +-- webhooks/         # whatsapp, email, stripe
|   |   +-- layout.tsx                # Root layout
|   |   +-- globals.css
|   +-- components/
|   |   +-- ui/                       # shadcn/ui components (Button, Card, Table, Dialog, etc.)
|   |   +-- layout/                   # sidebar, topbar, mobile-nav
|   |   +-- quinn/                    # quinn-panel, chat-message, action-button, quick-actions
|   |   +-- rfq/                      # rfq-table, rfq-board, rfq-detail, extraction-review, upload-dialog, status-badge
|   |   +-- quote/                    # quote-table, cost-breakdown, margin-control, scenario-compare, quote-preview
|   |   +-- cost-library/             # cost-table, cost-form, import-dialog
|   |   +-- dashboard/                # kpi-card, urgent-rfqs, pipeline-chart, activity-feed
|   |   +-- analytics/                # win-rate-chart, response-time-chart, buyer-scorecard, cost-trend-chart
|   |   +-- buyer/                    # buyer-list-table, buyer-detail-header, buyer-stats, buyer-form
|   |   +-- common/                   # search-command, notification-center, data-table, file-upload, date-range-picker
|   |   +-- settings/                 # factory-profile-form, team-management, integration-cards, billing-section
|   +-- lib/
|   |   +-- supabase/                 # client.ts, server.ts, admin.ts, middleware.ts
|   |   +-- redis/                    # client.ts
|   |   +-- anthropic/                # client.ts (retry logic, temperature configs)
|   |   +-- queue/                    # connection.ts, queues.ts (7 named queues)
|   |   +-- encryption.ts             # AES-256-GCM for credential storage
|   |   +-- validation/               # Zod schemas: rfq, quote, cost-library, buyer, user, tenant
|   |   +-- utils/                    # rfq-number, quote-number, priority, dedup, currency, date
|   |   +-- auth/                     # rbac.ts, plan-gate.ts
|   |   +-- audit.ts                  # Audit logging service
|   |   +-- constants.ts              # Default SAMs, fabric consumption, CMT rates, plan limits
|   +-- agents/
|   |   +-- orchestrator.ts           # Deterministic router
|   |   +-- inbox-watcher/            # agent.ts, prompts.ts, tools.ts
|   |   +-- extractor/                # agent.ts, prompts.ts, tools.ts
|   |   +-- cost-engine/              # agent.ts, prompts.ts, tools.ts, formulas.ts
|   |   +-- quote-writer/             # agent.ts, prompts.ts, tools.ts
|   |   +-- tracker/                  # agent.ts, prompts.ts, tools.ts
|   |   +-- quinn/                    # agent.ts, prompts.ts, tools.ts, context.ts
|   +-- workers/
|   |   +-- inbox-worker.ts           # BullMQ consumer: inbox-sync queue
|   |   +-- extraction-worker.ts      # BullMQ consumer: rfq-extraction + cost-calculation
|   |   +-- quote-worker.ts           # BullMQ consumer: quote-generation + quote-tracking
|   |   +-- notification-worker.ts    # BullMQ consumer: notifications + follow-up
|   |   +-- scheduler.ts              # Cron: expiry checks, inbox sync, tracking polls
|   +-- hooks/                        # use-rfqs, use-quotes, use-cost-library, use-notifications, use-quinn, use-realtime
|   +-- stores/                       # quinn-store.ts (zustand)
|   +-- types/                        # database.ts, rfq.ts, quote.ts, cost-library.ts, agents.ts, quinn.ts
+-- supabase/
|   +-- migrations/
|   |   +-- 00001_create_tenants.sql
|   |   +-- 00002_create_users.sql
|   |   +-- 00003_create_buyers.sql
|   |   +-- 00004_create_rfqs.sql
|   |   +-- 00005_create_quotes.sql
|   |   +-- 00006_create_cost_library.sql
|   |   +-- 00007_create_supporting_tables.sql
|   |   +-- 00008_create_rls_policies.sql
|   |   +-- 00009_create_indexes.sql
|   +-- seed.sql                      # Dev seed data
|   +-- config.toml
+-- tests/
|   +-- unit/                         # cost-engine, priority, dedup, status-transitions, plan-gate, rfq-number, currency
|   +-- e2e/                          # auth, upload-rfq, build-quote, send-quote
+-- .env.example
+-- .env.local                        # Git-ignored
+-- .eslintrc.json
+-- .prettierrc
+-- next.config.js
+-- tailwind.config.ts
+-- tsconfig.json
+-- vitest.config.ts
+-- playwright.config.ts
+-- package.json
+-- README.md
```

---

## 7. Build Order

### Phase 1: Foundation (Week 1-2)

| # | Task | Dependencies |
|---|---|---|
| 1.1 | Project scaffold: Next.js 14, TypeScript, Tailwind, shadcn/ui, ESLint, Prettier | None |
| 1.2 | Supabase project setup + database migrations (all 17 tables, indexes, RLS) | None |
| 1.3 | Supabase client setup (browser, server, admin) | 1.1, 1.2 |
| 1.4 | Redis + BullMQ connection setup | None |
| 1.5 | Auth system: signup, login, refresh, logout, forgot/reset password, Google OAuth | 1.3 |
| 1.6 | RBAC middleware (6 roles) + plan gating utility (starter < growth < scale) | 1.5 |
| 1.7 | Zod validation schemas for all entities | 1.1 |
| 1.8 | Core utilities: RFQ/quote number gen, dedup hash, priority scoring, currency formatting, date helpers | 1.1 |
| 1.9 | TypeScript types: database generated types, domain types (ExtractedSpecs, CostBreakdown) | 1.2 |
| 1.10 | Encryption module (AES-256-GCM for stored credentials) | 1.1 |

### Phase 2: Core Backend (Week 3-4)

| # | Task | Dependencies |
|---|---|---|
| 2.1 | Tenant API: GET/PATCH settings, logo/letterhead upload | 1.5, 1.6 |
| 2.2 | User API: list, invite, update, deactivate (plan user limit enforced) | 1.5, 1.6 |
| 2.3 | Buyer API: CRUD + computed stats (rfq_count, win_rate, avg_order_value) | 1.5, 1.6 |
| 2.4 | Cost Library API: CRUD, import (Excel/CSV + column mapping), bulk-update, snapshot | 1.5, 1.6, 1.7 |
| 2.5 | RFQ API: CRUD, assign, bulk-action, status transitions (enforced per business logic) | 1.5, 1.6, 1.7, 1.8 |
| 2.6 | File Upload API + Supabase Storage (25MB limit, type validation, malware scan) | 1.3 |
| 2.7 | Exchange Rates API | 1.5 |
| 2.8 | Notification API + creation service + Supabase Realtime subscription | 1.5 |
| 2.9 | Audit log API + audit logging service (change tracking with old/new values) | 1.5 |

### Phase 3: AI Agents (Week 5-6)

| # | Task | Dependencies |
|---|---|---|
| 3.1 | Anthropic Claude client wrapper with retry logic (3x exponential backoff on 429/500) | 1.1 |
| 3.2 | ExtractorAgent: prompts, tools (parse_pdf, parse_excel, ocr_document), output parsing, confidence scoring | 3.1, 2.5 |
| 3.3 | CostEngine Agent: formulas module (11 categories + incoterm adjustments), cost library lookup, cost breakdown builder | 3.1, 2.4 |
| 3.4 | QuoteWriter Agent: PDF generation (react-pdf/Puppeteer), buyer template system, Excel export | 3.3, 2.6 |
| 3.5 | Agent Orchestrator: deterministic router for extraction -> costing -> quote flow | 3.2, 3.3, 3.4 |
| 3.6 | Quote API: create (triggers cost engine), approve, send, revise, scenarios, generate-pdf | 3.3, 3.4 |
| 3.7 | BullMQ workers: extraction, cost calculation, quote generation | 3.5, 1.4 |
| 3.8 | Cost engine unit tests (all formulas, incoterm adjustments, margin calc, reverse calc) | 3.3 |

### Phase 4: Core UI (Week 7-9)

| # | Task | Dependencies |
|---|---|---|
| 4.1 | Layout: sidebar, topbar, mobile nav, dark/light theme (next-themes), Cmd+K search | 1.1 |
| 4.2 | Auth pages: login, signup, forgot-password, 3-step onboarding wizard | 1.5, 4.1 |
| 4.3 | Dashboard: KPI cards (open RFQs, avg response time, win rate, pipeline value), urgent RFQs, pipeline chart, activity feed | 4.1, 2.5 |
| 4.4 | RFQ Inbox: list view (filterable/sortable table), board view (Kanban), bulk actions, upload dialog, drag-and-drop | 4.1, 2.5 |
| 4.5 | RFQ Detail: extraction review (confidence bars, inline edit), attachments viewer, status timeline, notes with @mentions | 4.4, 3.2 |
| 4.6 | Quote Builder: cost breakdown table, margin slider, incoterm toggle, currency display, scenarios side-by-side | 4.1, 3.6 |
| 4.7 | Quote list page, quote preview/PDF preview | 4.6 |
| 4.8 | Cost Library: 11 category tabs, category-specific tables/forms, add/edit dialog, Excel import with column mapping | 4.1, 2.4 |
| 4.9 | Buyers: list with stats + detail (overview, RFQ history, quote history, settings tabs) | 4.1, 2.3 |
| 4.10 | Notification center: bell dropdown + full page + Supabase Realtime live badge | 4.1, 2.8 |
| 4.11 | Reusable components: data-table (sort/filter/paginate), file-upload, date-range-picker | 4.1 |

### Phase 5: QUINN Copilot (Week 10)

| # | Task | Dependencies |
|---|---|---|
| 5.1 | QUINN chat panel component (right sidebar, 400px desktop, full-screen mobile, SSE streaming, typing indicator) | 4.1 |
| 5.2 | QUINN agent: system prompt, tool definitions (10 tools), context injection (page route + active entity) | 3.1 |
| 5.3 | QUINN API: chat endpoint (SSE), conversation CRUD | 5.2 |
| 5.4 | QUINN tool execution: list_rfqs, build_quote, assign_rfq, send_quote, get_analytics, etc. (respects RBAC) | 5.2, 2.5, 3.6 |
| 5.5 | Quick actions (/ commands with autocomplete), action buttons in responses | 5.1 |
| 5.6 | QUINN conversation persistence + zustand store | 5.3 |

### Phase 6: Integrations & Tracking (Week 11-12)

| # | Task | Dependencies |
|---|---|---|
| 6.1 | InboxWatcher Agent: email classification, language detection, dedup hash | 3.1 |
| 6.2 | Gmail OAuth + inbox monitoring (Nylas or direct IMAP) | 6.1, 1.10 |
| 6.3 | Outlook OAuth + inbox monitoring | 6.1, 1.10 |
| 6.4 | Inbox connections API + settings UI | 6.2, 6.3 |
| 6.5 | TrackingAgent: email open detection, follow-up escalation (4h -> 48h -> 72h, max 3, min 24h gap) | 3.1, 3.6 |
| 6.6 | Auto follow-up worker + follow-up templates (12 template variables) | 6.5, 2.8 |
| 6.7 | Inbox sync worker (BullMQ cron every 60s) | 6.1, 1.4 |
| 6.8 | Scheduler: auto-expiry (RFQ deadline+7d, quote valid_until), deadline alerts, tracking polls | 1.4, 2.8 |

### Phase 7: Analytics & Settings (Week 13-14)

| # | Task | Dependencies |
|---|---|---|
| 7.1 | Analytics API: overview, win-rate, response-time, pipeline, buyer-scorecard, cost-trends | 2.5, 3.6 |
| 7.2 | Analytics UI: recharts (win rate trend, response time, pipeline bar, buyer intelligence, cost trends) | 7.1, 4.1 |
| 7.3 | Analytics export (XLSX/CSV download) | 7.1 |
| 7.4 | Settings pages: profile, team, integrations, templates, routing rules, notifications, billing, audit log | 4.1, 2.1, 2.2 |
| 7.5 | Stripe integration: subscription management, plan upgrades, billing portal, webhook handler | 7.4 |
| 7.6 | Audit log page with filters, expandable JSON, CSV export | 2.9, 4.1 |
| 7.7 | Auto-routing rules API + settings UI (Growth+ gated) | 2.5 |

### Phase 8: Testing & Polish (Week 15-16)

| # | Task | Dependencies |
|---|---|---|
| 8.1 | Unit tests: cost engine formulas, status transitions, priority scoring, plan gating, dedup, currency conversion | All backend |
| 8.2 | E2E tests: auth flow, upload RFQ, build quote, send quote | All frontend |
| 8.3 | Error handling + loading skeletons across all pages | All |
| 8.4 | Rate limiting middleware (auth: 10/min/IP, QUINN: 30/min/user, general: 100/min/user) | 1.6 |
| 8.5 | Security headers (CSP, HSTS, X-Content-Type-Options, X-Frame-Options, Permissions-Policy) | 1.1 |
| 8.6 | Sentry integration (error tracking with user/tenant context) | 1.1 |
| 8.7 | Performance: Redis caching layer (tenant config 300s, cost library 60s, exchange rates 3600s, analytics 300s) | All |
| 8.8 | Seed data + onboarding flow polish + responsive testing (desktop/tablet/mobile breakpoints) | All |

---

## 8. Dependencies

### Production
```json
{
  "next": "^14.x",
  "react": "^18.x",
  "react-dom": "^18.x",
  "typescript": "^5.x",
  "@supabase/supabase-js": "^2.x",
  "@supabase/ssr": "^0.x",
  "tailwindcss": "^3.x",
  "zod": "^3.x",
  "@anthropic-ai/sdk": "latest",
  "bullmq": "^5.x",
  "ioredis": "^5.x",
  "pino": "^8.x",
  "pino-pretty": "^10.x",
  "@react-pdf/renderer": "latest",
  "puppeteer": "latest",
  "tesseract.js": "latest",
  "imapflow": "latest",
  "stripe": "latest",
  "resend": "latest",
  "next-themes": "latest",
  "zustand": "^4.x",
  "date-fns": "^3.x",
  "recharts": "^2.x",
  "lucide-react": "latest",
  "class-variance-authority": "latest",
  "clsx": "latest",
  "tailwind-merge": "latest",
  "@radix-ui/react-*": "latest",
  "cmdk": "latest",
  "xlsx": "latest",
  "uuid": "^9.x"
}
```

### Dev Dependencies
```json
{
  "@types/node": "^20.x",
  "@types/react": "^18.x",
  "vitest": "latest",
  "@playwright/test": "latest",
  "eslint": "^8.x",
  "eslint-config-next": "^14.x",
  "@typescript-eslint/eslint-plugin": "latest",
  "prettier": "^3.x",
  "prettier-plugin-tailwindcss": "latest",
  "supabase": "latest"
}
```

---

## 9. Environment Variables

### Required (all environments)
```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=                # Supabase project URL
NEXT_PUBLIC_SUPABASE_ANON_KEY=           # Supabase anon/public key
SUPABASE_SERVICE_ROLE_KEY=               # Supabase service role key (server-only)
SUPABASE_DB_URL=                         # Direct PostgreSQL connection string

# Redis
REDIS_URL=                               # Redis connection URL

# AI
ANTHROPIC_API_KEY=                       # Claude API key

# Security
ENCRYPTION_KEY=                          # 32-byte hex, AES-256 key for credential encryption

# Application
NEXT_PUBLIC_APP_URL=                     # https://app.rfqx.ai (production)
NODE_ENV=                                # development | staging | production
```

### Optional (feature-specific)
```bash
# Google OAuth (Gmail integration + Google login)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=

# Microsoft OAuth (Outlook integration)
MICROSOFT_CLIENT_ID=
MICROSOFT_CLIENT_SECRET=
MICROSOFT_REDIRECT_URI=

# OCR
GOOGLE_CLOUD_VISION_API_KEY=
GOOGLE_APPLICATION_CREDENTIALS=          # Path to service account JSON

# Billing
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# Email (Nylas - alternative to direct IMAP)
NYLAS_CLIENT_ID=
NYLAS_CLIENT_SECRET=
NYLAS_API_KEY=

# Transactional email
RESEND_API_KEY=
EMAIL_FROM_ADDRESS=                      # noreply@rfqx.ai
EMAIL_FROM_NAME=                         # RFQX

# WhatsApp (Phase 2)
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_VERIFY_TOKEN=
WHATSAPP_BUSINESS_ACCOUNT_ID=

# Monitoring
SENTRY_DSN=
NEXT_PUBLIC_SENTRY_DSN=

# Exchange rates (optional, for live rates)
EXCHANGE_RATE_API_KEY=
EXCHANGE_RATE_API_URL=
```

### Environment Defaults

| Variable | Development | Staging | Production |
|---|---|---|---|
| NODE_ENV | development | staging | production |
| NEXT_PUBLIC_APP_URL | http://localhost:3000 | https://staging.rfqx.ai | https://app.rfqx.ai |
| Log level | debug | info | warn |
| Rate limiting | disabled | relaxed | strict |
| Email sending | console only | sandbox | live |
| Stripe | test keys | test keys | live keys |

---

## 10. Queue Architecture

| Queue Name | Purpose | Concurrency | Retry |
|---|---|---|---|
| `inbox-sync` | Poll email inboxes and portals | 5 | 3 retries, exponential backoff |
| `rfq-extraction` | AI extraction on new RFQs | 3 | 2 retries |
| `cost-calculation` | Build cost breakdowns | 5 | 2 retries |
| `quote-generation` | Generate PDF/Excel quotes | 3 | 2 retries |
| `quote-tracking` | Check email opens, portal status | 5 | 3 retries |
| `notifications` | Send email/in-app notifications | 10 | 3 retries |
| `follow-up` | Send automated follow-up emails | 2 | 2 retries |

All queues backed by Redis. Dead letter queue enabled after max retries.

---

## 11. Key Business Logic Reference

- **RFQ Statuses (12)**: new -> extracting -> extracted -> costing -> review -> approved -> sent -> followed_up -> won/lost/expired/archived
- **Quote Statuses (8)**: draft -> pending_review -> approved -> sent -> revised/accepted/rejected/expired
- **Auto-costing trigger**: All extracted fields >= 85% confidence AND required fields present (product_type, fabric_composition|fabric_gsm, total_quantity)
- **Dedup**: SHA-256(normalized sender + subject + buyer + date), 7-day match window
- **Priority**: base 3 + buyer tier adj (-1 to +2) + deadline adj (0 to +2) + value adj (-1 to +1), clamped 1-5
- **Costing formula**: Fabric + CMT + Trims + Embellishment + Washing + Packing + Overhead + Testing + Commission + Finance = Cost
- **Margin formula**: `MARGIN_PER_UNIT = COST_PER_UNIT * (margin% / (100 - margin%))`
- **FOB**: `FOB_PER_UNIT = COST_PER_UNIT + MARGIN_PER_UNIT`
- **Incoterm adjustments**: FOB (base), CIF (+freight +0.5% insurance), EXW (-local transport), DDP (+freight +insurance +duty +destination)
- **Follow-up escalation**: 4h -> 48h -> 72h, max 3 auto follow-ups, min 24h between, 3rd CCs owner
- **Plan enforcement**: RFQ/user limits checked on creation, feature gates on API entry (starter < growth < scale)
- **Quote versioning**: Same quote_number, incremented version, previous set to "revised"
- **Auto-expiry**: RFQs (deadline+7d for sent, immediate for pre-sent), Quotes (valid_until date)
- **Win rate**: won / (won + lost) * 100 (expired excluded from denominator)
- **Currencies**: USD, BDT, EUR, GBP, VND, INR (4 decimal places for per-unit, 2 for display)
- **RBAC**: 6 roles (owner > admin > quotation_manager > merchandiser > commercial_executive > viewer)
- **Tenant isolation**: PostgreSQL RLS on all 17 tables via auth.uid() -> users.supabase_auth_id -> tenant_id
