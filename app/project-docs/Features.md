# RFQX Feature Breakdown

**Product:** RFQX - AI RFQ & Quoting Intelligence for Garment Manufacturers
**Stack:** Next.js 14, TypeScript, Tailwind CSS, shadcn/ui, Supabase (PostgreSQL + Auth + Storage + Realtime), Redis/BullMQ, Claude API
**Source:** BUILD_PLAN.md, PRD.md, TECH_SPEC.md, UI_SPEC.md, API_SPEC.md, BUSINESS_LOGIC.md, CONSTRAINTS.md

---

## Feature 1: Foundation (large)

**Description:** Database schema (all 17 tables), migrations, RLS policies, indexes, TypeScript types, Supabase client setup (browser/server/admin), Redis + BullMQ connection, authentication system (signup/login/logout/password reset/Google OAuth), RBAC middleware (6 roles), plan gating utility, encryption module (AES-256-GCM), Zod validation schemas, core utilities (RFQ/quote number generation, dedup hash, priority scoring, currency formatting, date helpers), Anthropic Claude client wrapper with retry logic, audit logging service, and constants (default SAMs, fabric consumption, CMT rates, plan limits). No frontend.

**Database:**
- Tables to create: `tenants`, `users`, `buyers`, `rfqs`, `quotes`, `cost_library_items`, `cost_library_snapshots`, `quote_templates`, `inbox_connections`, `inbox_sync_state`, `rfq_attachments`, `exchange_rates`, `notifications`, `audit_log`, `quinn_conversations`, `auto_routing_rules`, `follow_up_templates`
- RLS policies for all 17 tables (tenant isolation via `auth.uid()` -> `users.supabase_auth_id` -> `tenant_id`)
- Indexes on all foreign keys, filter/sort columns, composite indexes per TECH_SPEC
- Migration files:
  - `00001_create_tenants.sql`
  - `00002_create_users.sql`
  - `00003_create_buyers.sql`
  - `00004_create_rfqs.sql`
  - `00005_create_quotes.sql`
  - `00006_create_cost_library.sql`
  - `00007_create_supporting_tables.sql` (quote_templates, inbox_connections, inbox_sync_state, rfq_attachments, exchange_rates, notifications, audit_log, quinn_conversations, auto_routing_rules, follow_up_templates)
  - `00008_create_rls_policies.sql`
  - `00009_create_indexes.sql`

**Backend:**
- Supabase client setup: `src/lib/supabase/client.ts` (browser), `src/lib/supabase/server.ts` (server components), `src/lib/supabase/admin.ts` (service role)
- Redis client: `src/lib/redis/client.ts`
- BullMQ queue definitions: `src/lib/queue/connection.ts`, `src/lib/queue/queues.ts` (7 named queues: inbox-sync, rfq-extraction, cost-calculation, quote-generation, quote-tracking, notifications, follow-up)
- Auth API endpoints:
  - `POST /api/v1/auth/signup` - Create tenant + owner user
  - `POST /api/v1/auth/login` - Email/password login
  - `POST /api/v1/auth/refresh` - Refresh JWT
  - `POST /api/v1/auth/logout` - End session
  - `POST /api/v1/auth/forgot-password` - Send reset email
  - `POST /api/v1/auth/reset-password` - Reset with token
  - `GET /api/v1/auth/callback/google` - Google OAuth callback
- RBAC middleware: `src/lib/auth/rbac.ts` (6 roles: owner, admin, quotation_manager, merchandiser, commercial_executive, viewer)
- Plan gating: `src/lib/auth/plan-gate.ts` (starter < growth < scale feature/limit checks)
- Encryption: `src/lib/encryption.ts` (AES-256-GCM for stored credentials)
- Zod schemas: `src/lib/validation/` (rfq, quote, cost-library, buyer, user, tenant schemas per BUSINESS_LOGIC validation rules)
- Utilities: `src/lib/utils/` (rfq-number gen RFQ-YYYYMM-XXXX, quote-number gen QT-YYYYMM-XXXX, dedup SHA-256 hash, priority scoring formula, currency formatting 6 currencies, date helpers)
- Audit service: `src/lib/audit.ts` (log entity changes with old/new JSON diffs)
- Constants: `src/lib/constants.ts` (default SAMs, fabric consumption by product type, CMT rates by country, plan limits)
- TypeScript types: `src/types/` (database.ts generated, rfq.ts with ExtractedSpecs, quote.ts with CostBreakdown, cost-library.ts, agents.ts, quinn.ts)
- Anthropic client wrapper: `src/lib/anthropic/client.ts` (retry logic 3x exponential backoff on 429/500, per-agent temperature configs)

**Frontend:** None (foundation only)

**Agent:** None

**Depends on:** Nothing

**Done when:**
- [ ] All 17 tables created with correct schemas, constraints, and CHECK constraints
- [ ] RLS policies enforce tenant isolation on all tables
- [ ] All indexes created per TECH_SPEC
- [ ] Supabase client works from browser, server components, and admin context
- [ ] Redis + BullMQ connections establish successfully
- [ ] Auth signup creates tenant + owner, login returns JWT, refresh/logout/password reset work
- [ ] Google OAuth callback processes correctly
- [ ] RBAC middleware correctly restricts endpoints by role
- [ ] Plan gating blocks features per tier (starter < growth < scale)
- [ ] Encryption module encrypts/decrypts credentials
- [ ] All Zod schemas validate per BUSINESS_LOGIC rules
- [ ] RFQ/quote number generation is sequential per tenant per month
- [ ] Dedup hash matches spec (SHA-256 of normalized sender+subject+buyer+date)
- [ ] Priority scoring formula produces correct 1-5 values
- [ ] Audit log service records entity changes with old/new diffs
- [ ] TypeScript types match database schema exactly

---

## Feature 2: Auth Pages & Onboarding (medium)

**Description:** Login, signup, forgot-password pages and 3-step onboarding wizard. These are the entry points for new and returning users. Onboarding creates the factory profile, initial cost library, and connects email inbox.

**Database:** No new tables (uses tenants, users, cost_library_items, inbox_connections from Feature 1)

**Backend:** Uses auth endpoints from Feature 1. No new endpoints.

**Frontend:**
- Pages/screens:
  - `/login` - Email/password login, Google OAuth button, forgot password link, signup link. Centered card on gradient background with RFQX logo.
  - `/signup` - Full name, email, password, factory name, country select (Bangladesh pre-selected). Creates tenant + owner.
  - `/forgot-password` - Email input, send reset link button.
  - `/onboarding` - 3-step wizard (first-time only):
    - Step 1: Factory Profile (name, address, phone, website, logo upload, base currency, default incoterm)
    - Step 2: Cost Library Setup (upload Excel OR manually enter initial costs, category tabs)
    - Step 3: Connect Inbox (Gmail OAuth, Outlook OAuth, Skip option)
- Components: Auth layout (centered card), Stepper component, FileUpload
- Prototype files:
  - `prototypes/login.html`
  - `prototypes/signup.html`
  - `prototypes/forgot-password.html`
  - `prototypes/onboarding.html`

**Agent:** None

**Depends on:** Feature 1

**Done when:**
- [ ] Login page authenticates via email/password and Google OAuth
- [ ] Signup creates tenant + owner user and redirects to onboarding
- [ ] Forgot-password sends reset email via Supabase Auth
- [ ] Onboarding wizard completes 3 steps: factory profile, cost library setup, inbox connection
- [ ] Logo upload works via Supabase Storage (2MB limit, PNG/JPG/SVG)
- [ ] Cost library Excel import maps columns to RFQX fields
- [ ] Gmail/Outlook OAuth connection stores encrypted credentials
- [ ] Redirect to /dashboard after onboarding completes

---

## Feature 3: App Shell & Dashboard (medium)

**Description:** Dashboard layout (sidebar, topbar, mobile nav, dark/light theme, Cmd+K search) and the dashboard page with KPI cards, urgent RFQs widget, pipeline chart, activity feed, and quote value by buyer chart.

**Database:** No new tables. Reads from rfqs, quotes, audit_log.

**Backend:**
- `GET /api/v1/analytics/overview` - KPI summary (open RFQs count, avg response time, win rate, pipeline value, missed RFQs, revenue forecast)
- `GET /api/v1/audit-log` - Recent activity feed (last 10 entries for tenant)

**Frontend:**
- Pages/screens:
  - `/dashboard` - KPI cards (Open RFQs, Avg Response Time, Win Rate, Pipeline Value), Urgent RFQs table (top 5 by deadline), Recent Activity Feed (last 10 audit entries), Pipeline by Status stacked bar chart, Quote Value by Buyer bar chart
- Components:
  - `src/components/layout/sidebar.tsx` - Left sidebar with navigation links
  - `src/components/layout/topbar.tsx` - Breadcrumb, Search (Cmd+K), Notifications bell, QUINN toggle
  - `src/components/layout/mobile-nav.tsx` - Bottom navigation for mobile
  - `src/components/common/search-command.tsx` - Cmd+K command palette (search RFQs, quotes, buyers, cost library)
  - `src/components/dashboard/kpi-card.tsx` - Metric card with value, trend arrow, sparkline/donut
  - `src/components/dashboard/urgent-rfqs.tsx` - Table of top 5 urgent RFQs
  - `src/components/dashboard/pipeline-chart.tsx` - Horizontal stacked bar (recharts)
  - `src/components/dashboard/activity-feed.tsx` - Audit log entries with avatars and links
  - Dashboard layout: `src/app/(dashboard)/layout.tsx` (sidebar + topbar + QUINN panel slot)
- Theme: next-themes dark/light toggle, Inter font, primary blue-600
- Responsive: Full sidebar (>=1280px), icon-only sidebar (768-1279px), bottom nav (<768px)
- Prototype file: `prototypes/dashboard.html`

**Agent:** None

**Depends on:** Feature 1, Feature 2

**Done when:**
- [ ] Sidebar navigation works with all main routes (Dashboard, Inbox, Quotes, Cost Library, Buyers, Analytics, Settings)
- [ ] Topbar shows breadcrumb, Cmd+K search palette, notification bell placeholder, QUINN toggle placeholder
- [ ] Mobile bottom navigation works at <768px breakpoint
- [ ] Dark/light theme toggle works via next-themes
- [ ] Dashboard shows 4 KPI cards with real data from analytics/overview endpoint
- [ ] Urgent RFQs widget shows top 5 RFQs sorted by deadline with color-coded deadlines
- [ ] Pipeline by Status chart renders with recharts
- [ ] Activity feed shows last 10 audit log entries
- [ ] Date range picker defaults to last 30 days

---

## Feature 4: Buyers Management (medium)

**Description:** Buyer list page with computed stats and buyer detail page with overview, RFQ history, quote history, and settings tabs. This is foundational for RFQs and quotes which reference buyers.

**Database:** No new tables. Uses `buyers`, `rfqs`, `quotes` from Feature 1.

**Backend:**
- `GET /api/v1/buyers` - List buyers with computed stats (rfq_count, win_rate, avg_order_value). Searchable, filterable by tier/active.
- `GET /api/v1/buyers/:id` - Buyer detail with stats (total_rfqs, total_quotes, won_quotes, lost_quotes, win_rate, avg_order_value, avg_margin, last_rfq_date)
- `POST /api/v1/buyers` - Create buyer (roles: owner, admin, qm, merchandiser)
- `PATCH /api/v1/buyers/:id` - Update buyer (roles: owner, admin, qm)

**Frontend:**
- Pages/screens:
  - `/buyers` - Searchable table: Name, Code, Country, Tier (star rating), Contact, RFQ count, Win Rate %, Preferred Incoterm, Portal type, Active toggle. "Add Buyer" button.
  - `/buyers/[buyerId]` - Profile header (name, code, tier stars, country, contact, portal badge, Edit button) + 4 tabs:
    - Overview: Stats cards (Total RFQs, Win Rate, Avg Order Value, Avg Margin, Last RFQ Date) + volume trend chart (12 months)
    - RFQ History: Table of buyer's RFQs (RFQ#, Product, Qty, Status, FOB/Unit, Date)
    - Quote History: Table of buyer's quotes (Quote#, Product, Qty, FOB/Unit, Margin%, Status, Date)
    - Settings: Preferred template, default commission %, portal credentials (masked), payment terms, notes
- Components:
  - `src/components/buyer/buyer-list-table.tsx`
  - `src/components/buyer/buyer-detail-header.tsx`
  - `src/components/buyer/buyer-stats.tsx`
  - `src/components/buyer/buyer-form.tsx` (add/edit dialog)
- Prototype files:
  - `prototypes/buyers-list.html`
  - `prototypes/buyer-detail.html`

**Agent:** None

**Depends on:** Feature 1, Feature 3

**Done when:**
- [ ] Buyer list shows all buyers with computed stats (rfq_count, win_rate)
- [ ] Search filters buyers by name/code
- [ ] Tier filter and active filter work
- [ ] Add Buyer dialog creates buyer with validation (name required, code unique per tenant, tier 1-5)
- [ ] Buyer detail shows profile header with editable tier
- [ ] Overview tab shows stats cards and volume trend chart
- [ ] RFQ History and Quote History tabs show linked records
- [ ] Settings tab saves preferred template, commission %, payment terms

---

## Feature 5: Cost Library (medium)

**Description:** Cost library management with 11 category tabs, category-specific tables and forms, Excel/CSV import with column mapping, bulk update (Growth+), and snapshot creation.

**Database:** No new tables. Uses `cost_library_items`, `cost_library_snapshots` from Feature 1.

**Backend:**
- `GET /api/v1/cost-library` - List items by category (paginated, searchable by name/code/supplier)
- `GET /api/v1/cost-library/:id` - Item detail
- `POST /api/v1/cost-library` - Add item (category-specific fields validated)
- `PATCH /api/v1/cost-library/:id` - Update item
- `DELETE /api/v1/cost-library/:id` - Soft-delete (deactivate)
- `POST /api/v1/cost-library/import` - Import from Excel/CSV with column mapping
- `POST /api/v1/cost-library/bulk-update` - Bulk percentage/absolute price update (Growth+ gated)
- `POST /api/v1/cost-library/snapshot` - Create point-in-time snapshot
- `GET /api/v1/exchange-rates` - List exchange rates
- `POST /api/v1/exchange-rates` - Manual rate entry

**Frontend:**
- Pages/screens:
  - `/cost-library` - 11 category tabs (Fabric, CMT, Trims, Embellishment, Washing, Testing, Packing, Overhead, Freight, Commission, Finance). Each tab shows category-specific table columns:
    - Fabric: Name, Code, Composition, GSM, Construction, Width, Cost/Meter, Currency, Supplier, Consumption/Unit, Effective From, Active toggle
    - CMT: Product Type, Complexity tier badge, SAM, Rate/Unit, MOQ Min, MOQ Max, Effective From, Active toggle
    - (Other tabs: category-relevant columns)
  - Top actions: "Add Item", "Import from Excel", "Bulk Update" (Growth+), "Export"
  - Add/Edit Item dialog: Dynamic form based on selected category
  - Import dialog: File upload zone, column mapping interface (RFQX field <-> Excel header), preview first 5 rows, Import button
- Components:
  - `src/components/cost-library/cost-table.tsx` (per-category column configs)
  - `src/components/cost-library/cost-form.tsx` (dynamic form by category)
  - `src/components/cost-library/import-dialog.tsx` (upload + column mapping + preview)
- Prototype file: `prototypes/cost-library.html`

**Agent:** None

**Depends on:** Feature 1, Feature 3

**Done when:**
- [ ] All 11 category tabs display with correct category-specific columns
- [ ] Add/Edit dialog shows category-appropriate fields (fabric fields for fabric, CMT fields for CMT, etc.)
- [ ] Cost validation enforces rules: cost > 0, fabric_gsm 50-1000, sam_minutes 1-120, moq_max > moq_min
- [ ] Soft-delete deactivates items (is_active = false)
- [ ] Excel/CSV import with column mapping creates items correctly
- [ ] Import shows preview of first 5 rows and reports errors per row
- [ ] Bulk update (Growth+ gated) applies percentage or absolute changes
- [ ] Snapshot creates point-in-time copy of all cost library items
- [ ] Search filters by name, code, supplier_name
- [ ] Exchange rates CRUD works (manual entry for USD, BDT, EUR, GBP, VND, INR)

---

## Feature 6: RFQ Inbox & Detail (large)

**Description:** RFQ inbox with list view (filterable/sortable table), board view (Kanban), file upload, bulk actions, and RFQ detail page with extracted specs review, attachments viewer, cost sheet link, communication timeline, status timeline, and internal notes.

**Database:** No new tables. Uses `rfqs`, `rfq_attachments`, `buyers`, `users` from Feature 1.

**Backend:**
- `GET /api/v1/rfqs` - List RFQs (paginated, filterable by status/buyer/priority/assigned/date/search, sortable by created_at/deadline/priority)
- `GET /api/v1/rfqs/:id` - RFQ detail with extracted_data, attachments, linked quotes
- `POST /api/v1/rfqs` - Create RFQ via file upload (multipart, plan RFQ limit enforced, dedup hash check)
- `PATCH /api/v1/rfqs/:id` - Update RFQ fields/status (allowed transitions enforced per BUSINESS_LOGIC)
- `POST /api/v1/rfqs/:id/assign` - Assign to user
- `POST /api/v1/rfqs/bulk-action` - Bulk assign/archive/priority (max 50)
- `DELETE /api/v1/rfqs/:id` - Soft-delete (archive)
- `PATCH /api/v1/rfqs/:id/extracted-data` - Override extracted fields (logged in manual_overrides)
- `POST /api/v1/upload` - File upload to Supabase Storage (25MB limit, type validation)
- `GET /api/v1/upload/:id/signed-url` - Signed download URL (1hr expiry)

**Frontend:**
- Pages/screens:
  - `/inbox` (List View) - Filter bar (status multi-select, buyer select, priority select, assigned to select, date range, search input, "Upload RFQ" button, List/Board toggle). Table: Checkbox, RFQ#, Buyer, Product, Quantity, Priority badge, Status badge, Source icon, Deadline (color-coded), Assigned avatar, Created. Bulk actions bar (Assign, Priority, Archive, Export CSV).
  - `/inbox?view=board` (Board View) - Kanban columns: New, Extracting, Costing, Review, Sent, Followed Up. Cards: buyer, product, quantity, priority, deadline, assigned avatar. Drag between columns.
  - `/inbox/[rfqId]` (RFQ Detail) - Two-column layout:
    - Left: Tabs (Extracted Specs table with confidence bars + inline edit, Attachments grid with preview/download/OCR toggle, Cost Sheet link, Communication timeline)
    - Right: Status timeline, Internal notes with @mention, Activity feed
    - Top: Back button, RFQ#, status badge, priority badge, "Build Quote" button, "Assign" dropdown, "Archive"
- Components:
  - `src/components/rfq/rfq-table.tsx` (sortable/filterable data table)
  - `src/components/rfq/rfq-board.tsx` (Kanban board)
  - `src/components/rfq/rfq-detail.tsx`
  - `src/components/rfq/extraction-review.tsx` (confidence bars, inline edit, flagged field highlights)
  - `src/components/rfq/upload-dialog.tsx` (drag-and-drop file upload)
  - `src/components/rfq/status-badge.tsx` (12 status colors)
  - `src/components/common/data-table.tsx` (reusable sort/filter/paginate)
  - `src/components/common/file-upload.tsx` (drag-and-drop)
  - `src/components/common/date-range-picker.tsx`
- Prototype files:
  - `prototypes/inbox-list.html`
  - `prototypes/inbox-board.html`
  - `prototypes/rfq-detail.html`

**Agent:** None

**Depends on:** Feature 1, Feature 3, Feature 4

**Done when:**
- [ ] RFQ list view shows all RFQs with correct columns, sortable and filterable
- [ ] Board view shows Kanban columns with draggable cards
- [ ] Upload RFQ dialog accepts PDF/XLSX/DOCX/PNG/JPG/CSV files (25MB limit)
- [ ] File upload creates RFQ with status "extracting" and stores attachments in Supabase Storage
- [ ] Plan RFQ limit enforced (starter: 20/month, growth: 75, scale: unlimited)
- [ ] Dedup hash detects and merges duplicate RFQs within 7-day window
- [ ] Bulk actions (assign, archive, set priority) work on up to 50 selected RFQs
- [ ] RFQ detail shows extracted specs with confidence bars (green >=85%, amber 70-84%, red <70%)
- [ ] Inline edit on extracted fields saves to manual_overrides with audit trail
- [ ] Attachments show with preview, download, and OCR text toggle
- [ ] Status transitions enforced per allowed transitions in BUSINESS_LOGIC
- [ ] Priority scoring auto-calculates from buyer tier + deadline + value
- [ ] Status timeline shows chronological status changes with user and duration
- [ ] Internal notes support @mention

---

## Feature 7: AI Extraction Pipeline (large)

**Description:** ExtractorAgent that parses RFQ documents (PDF, Excel, email body, scanned images) into structured garment specs with per-field confidence scoring. Includes OCR support and BullMQ worker for async processing.

**Database:** No new tables. Updates `rfqs.extracted_data`, `rfqs.extraction_confidence_avg`, `rfqs.fields_flagged_for_review`, `rfq_attachments.ocr_text`, `rfq_attachments.extracted_data`.

**Backend:**
- `POST /api/v1/rfqs/:id/extract` - Trigger AI extraction (returns job_id, async via BullMQ rfq-extraction queue)
- ExtractorAgent: `src/agents/extractor/agent.ts`, `src/agents/extractor/prompts.ts`, `src/agents/extractor/tools.ts`
  - Tools: `ocr_document` (Google Cloud Vision + Tesseract.js fallback), `parse_pdf`, `parse_excel`, `parse_email_body`
  - Output: ExtractedSpecs (18 fields with per-field confidence 0-100): product_type, style_number, season, fabric_composition, fabric_gsm, fabric_construction, yarn_count, colorways, total_quantity, size_breakdown, delivery_date, delivery_port, incoterm, payment_terms, buyer_reference, certifications, testing_requirements, packaging_specs, print_embroidery_details
  - LLM: Claude claude-opus-4-5, temp=0.1, max_tokens=4096, 2x retry
  - Skill system: Garment (active), Footwear, Home Textile extraction models
- BullMQ worker: `src/workers/extraction-worker.ts` (consumes rfq-extraction queue, concurrency 3, 2 retries)
- Auto-proceed logic: If all fields >= 85% confidence AND required fields present (product_type, fabric_composition|fabric_gsm, total_quantity), auto-transition to "costing"
- Correction learning: After >10 overrides per field type in 30 days, augment prompt with 5 most recent corrections as few-shot examples

**Frontend:** None (uses existing RFQ detail extraction review from Feature 6)

**Agent:**
- Tools to build: `ocr_document`, `parse_pdf`, `parse_excel`, `parse_email_body`
- Skills to build: Garment spec extraction skill (structured prompt + output parsing)

**Depends on:** Feature 1, Feature 6

**Done when:**
- [ ] Extraction trigger enqueues BullMQ job and returns job_id
- [ ] Worker processes PDF attachments and extracts structured specs
- [ ] OCR works on scanned documents and images (Google Cloud Vision primary, Tesseract.js fallback)
- [ ] Excel files parsed for spec data
- [ ] All 18 fields extracted with confidence scores 0-100
- [ ] Fields with confidence < 85% flagged in fields_flagged_for_review
- [ ] extraction_confidence_avg calculated as average of non-null field confidences
- [ ] RFQ status transitions: new -> extracting -> extracted (or back to new on failure)
- [ ] Auto-proceed to costing when all fields >= 85% and required fields present
- [ ] Correction learning augments prompt after >10 overrides in 30 days
- [ ] Multi-attachment RFQs process all attachments (1-3 Claude API calls)

---

## Feature 8: Cost Engine & Quote Builder (large)

**Description:** CostEngine agent that calculates full cost breakdowns (11 categories + incoterm adjustments + margin), QuoteWriter agent for PDF/Excel generation, and the quote builder UI with cost breakdown table, margin slider, incoterm toggle, scenario comparison, and quote list page.

**Database:** No new tables. Uses `quotes`, `cost_library_items`, `cost_library_snapshots`, `quote_templates` from Feature 1.

**Backend:**
- `POST /api/v1/quotes` - Build quote from RFQ (triggers CostEngine, returns 422 if extraction incomplete)
- `GET /api/v1/quotes` - List quotes (paginated, filterable by status/buyer/date)
- `GET /api/v1/quotes/:id` - Quote detail with cost breakdown + version history
- `PATCH /api/v1/quotes/:id` - Update margin/incoterm/details (recalculates costs)
- `POST /api/v1/quotes/:id/approve` - Approve quote
- `POST /api/v1/quotes/:id/revise` - Create new version (previous set to "revised", same quote_number, version incremented)
- `POST /api/v1/quotes/:id/generate-pdf` - Generate PDF (signed URL, 1hr expiry)
- `POST /api/v1/quotes/:id/scenarios` - Compare up to 3 scenarios (Growth+ gated)
- CostEngine Agent: `src/agents/cost-engine/agent.ts`, `prompts.ts`, `tools.ts`, `formulas.ts`
  - Tools: `lookup_cost_library`, `get_exchange_rate`, `calculate_cmt`, `calculate_fabric_cost`
  - Formulas: Fabric (consumption * cost), CMT (rate card lookup by product_type + qty bracket), Trims (sum), Washing (rate by type), Embellishment (stitch/color-based), Packing (per-unit), Overhead (% of CMT), Testing (total/qty), Commission (% of subtotal), Finance (% of subtotal), FOB = sum of all
  - Margin: `MARGIN_PER_UNIT = COST_PER_UNIT * (margin% / (100 - margin%))`
  - Incoterm adjustments: FOB (base), CIF (+freight +0.5% insurance), EXW (-local transport), DDP (+freight +insurance +duty +destination)
  - Reverse calc from buyer counter-offer
  - LLM: Claude claude-opus-4-5, temp=0.0, max_tokens=2048
- QuoteWriter Agent: `src/agents/quote-writer/agent.ts`, `prompts.ts`, `tools.ts`
  - Tools: `get_buyer_template`, `generate_pdf` (@react-pdf/renderer + Puppeteer), `generate_excel`, `get_factory_letterhead`
  - LLM: Claude claude-opus-4-5, temp=0.3, max_tokens=2048
- Agent Orchestrator: `src/agents/orchestrator.ts` (deterministic router: extraction -> costing -> quote)
- BullMQ workers: `src/workers/extraction-worker.ts` (extended for cost-calculation queue), `src/workers/quote-worker.ts` (quote-generation queue)

**Frontend:**
- Pages/screens:
  - `/quotes` - Quote list: Filter bar (status, buyer, date, search). Table: Quote#, RFQ# (linked), Buyer, Product, Qty, FOB/Unit (4 decimals), FOB Total, Margin%, Status badge, Version, Sent date, Created.
  - `/quotes/[quoteId]` - Quote detail/builder: Header (quote#, version, status, buyer, RFQ ref, actions: Approve, Send, Revise, Download PDF/Excel, Compare Scenarios). Cost breakdown table (11 rows + Cost Price + Margin + FOB Price). Margin slider (0-50%) with real-time FOB update. Incoterm toggle (FOB/CIF/EXW/DDP). Currency toggle. Quote details (quantity, size breakdown, delivery terms, payment terms, validity, notes).
  - `/quotes/[quoteId]/preview` - Full-page PDF preview with buyer template formatting, factory letterhead, Download and Send buttons.
  - Scenario comparison panel: Side-by-side table of up to 3 scenarios with name, margin, FOB, total value.
- Components:
  - `src/components/quote/quote-table.tsx`
  - `src/components/quote/cost-breakdown.tsx` (11 cost rows + totals)
  - `src/components/quote/margin-control.tsx` (slider + input)
  - `src/components/quote/scenario-compare.tsx`
  - `src/components/quote/quote-preview.tsx`
- Prototype files:
  - `prototypes/quotes-list.html`
  - `prototypes/quote-builder.html`
  - `prototypes/quote-preview.html`

**Agent:**
- Tools to build: `lookup_cost_library`, `get_exchange_rate`, `calculate_cmt`, `calculate_fabric_cost`, `get_buyer_template`, `generate_pdf`, `generate_excel`, `get_factory_letterhead`
- Skills to build: FOB/CIF/EXW/DDP calculation, margin calculation, reverse calculation, template formatting

**Depends on:** Feature 1, Feature 3, Feature 5, Feature 6, Feature 7

**Done when:**
- [ ] Quote creation triggers CostEngine and produces correct cost breakdown for all 11 categories
- [ ] Fabric cost uses consumption * cost_per_unit (meters or kg)
- [ ] CMT rate card lookup matches by product_type + quantity bracket
- [ ] Margin formula: MARGIN_PER_UNIT = COST_PER_UNIT * (margin% / (100 - margin%))
- [ ] FOB = sum of all cost categories + margin
- [ ] Incoterm toggle recalculates: CIF (+freight +0.5% insurance), EXW (-local transport), DDP (+freight +insurance +duty +destination)
- [ ] Margin slider updates FOB in real-time (0-50% range)
- [ ] Currency toggle displays in USD/BDT/EUR/buyer currency
- [ ] Quote versioning: revise creates new version with same quote_number
- [ ] PDF generation uses buyer template + factory letterhead
- [ ] Scenario comparison shows up to 3 side-by-side (Growth+ gated)
- [ ] Quote list filterable and sortable
- [ ] Quote preview shows full PDF-style preview
- [ ] Returns 422 if extraction incomplete or has unresolved flagged fields
- [ ] Default values used when cost library entries missing (per BUSINESS_LOGIC defaults)
- [ ] Reverse calc from buyer counter-offer shows resulting margin with warnings

---

## Feature 9: Notifications & Realtime (medium)

**Description:** In-app notification system with bell dropdown, full page view, Supabase Realtime live badge updates, email notifications via Resend, and notification preferences.

**Database:** No new tables. Uses `notifications` from Feature 1.

**Backend:**
- `GET /api/v1/notifications` - List notifications (filterable by type/read status, paginated)
- `PATCH /api/v1/notifications/:id/read` - Mark single notification as read
- `POST /api/v1/notifications/mark-all-read` - Mark all as read
- Notification creation service: Triggers for all 13 notification types per BUSINESS_LOGIC section 13 (new_rfq, deadline_approaching, quote_opened, buyer_response, assignment, mention, etc.)
- Supabase Realtime subscription for live notification count updates
- BullMQ notification worker: `src/workers/notification-worker.ts` (consumes notifications queue, concurrency 10, 3 retries)
- Email sending via Resend (invites, notifications, password reset)
- Notification suppression: Respect user preferences, no email if active in-app within 5 min, batch >3 notifications in 10 min into digest

**Frontend:**
- Pages/screens:
  - Notification bell dropdown (in topbar): Last 20 notifications, read/unread indicator, type icon, title, body preview, relative timestamp, click to navigate, "Mark All Read", "View All" link
  - `/notifications` (full page) - Filterable list of all notifications (by type, read/unread, date range)
- Components:
  - `src/components/common/notification-center.tsx` (bell dropdown)
  - Notification page content
- Prototype file: `prototypes/notifications.html`

**Agent:** None

**Depends on:** Feature 1, Feature 3

**Done when:**
- [ ] Notification bell shows unread count badge in topbar
- [ ] Dropdown shows last 20 notifications sorted by created_at DESC
- [ ] Click notification navigates to linked RFQ or quote
- [ ] Mark single and mark all as read work
- [ ] Full notifications page filterable by type, read/unread, date range
- [ ] Supabase Realtime updates badge count without page refresh
- [ ] Email notifications sent via Resend for appropriate triggers
- [ ] User notification preferences respected (email on/off, in_app on/off)
- [ ] Suppression: no email if user active in-app within 5 min
- [ ] Batching: >3 notifications in 10 min sent as digest

---

## Feature 10: Settings Pages (medium)

**Description:** All settings pages: Factory Profile, Team Management, Integrations (placeholder), Templates, Routing Rules (Growth+), Notification Preferences, Billing (Owner only), Audit Log.

**Database:** No new tables. Uses `tenants`, `users`, `quote_templates`, `auto_routing_rules`, `audit_log` from Feature 1.

**Backend:**
- `GET /api/v1/tenant` - Tenant settings
- `PATCH /api/v1/tenant` - Update tenant settings
- `POST /api/v1/tenant/logo` - Upload logo (2MB, PNG/JPG/SVG)
- `POST /api/v1/tenant/letterhead` - Upload letterhead (5MB, PDF/PNG)
- `GET /api/v1/users` - List team members with workload counts (open_rfq_count, open_quote_count)
- `POST /api/v1/users/invite` - Invite user (plan user limit enforced)
- `PATCH /api/v1/users/:id` - Update role/status/notification preferences
- `DELETE /api/v1/users/:id` - Deactivate user (soft delete, owner only)
- `GET /api/v1/audit-log` - Audit trail (filterable by user/action/entity/date, paginated)
- `GET /api/v1/audit-log/export` - CSV export
- `CRUD /api/v1/routing-rules` - Auto-routing rules (Growth+ gated)
- `GET/POST /api/v1/follow-up-templates` - Follow-up email templates

**Frontend:**
- Pages/screens (tabbed layout under `/settings`):
  - `/settings/profile` - Factory name, address, phone, website, logo/letterhead upload, base currency, default margin/incoterm/validity, overhead/commission/finance %
  - `/settings/team` - User table (Name, Email, Role, Status, Last Login, workload, actions). "Invite User" dialog (email, name, role). MFA status.
  - `/settings/integrations` - Connected inboxes list (placeholder for Feature 12). Connect Gmail/Outlook/IMAP buttons.
  - `/settings/templates` - Quote template list (Name, Buyer, Type, Default toggle, Edit/Preview). "Create Template" button.
  - `/settings/routing` - Routing rules table (Growth+ gated): Name, Condition (buyer/product/priority/channel), Assign To, Active toggle, priority order. "Add Rule" form.
  - `/settings/notifications` - Per-user notification preferences matrix (rows: event types, columns: Email/In-App, toggle per cell)
  - `/settings/billing` - Current plan card (plan, price, RFQ usage X/Y, user count). "Upgrade Plan" modal with plan comparison. Stripe card, invoice history.
  - `/settings/audit-log` - Filterable table (Timestamp, User, Action, Entity, Changes expandable JSON, IP). Export CSV.
- Prototype files:
  - `prototypes/settings-profile.html`
  - `prototypes/settings-team.html`
  - `prototypes/settings-integrations.html`
  - `prototypes/settings-templates.html`
  - `prototypes/settings-routing.html`
  - `prototypes/settings-notifications.html`
  - `prototypes/settings-billing.html`
  - `prototypes/settings-audit-log.html`

**Agent:** None

**Depends on:** Feature 1, Feature 3

**Done when:**
- [ ] Factory profile edits save correctly (name, address, defaults, branding uploads)
- [ ] Team management: invite user enforces plan user limit (starter: 2, growth: 10, scale: unlimited)
- [ ] Invite sends email via Resend with signup link
- [ ] Role changes work (owner can change any role, admin cannot promote to owner)
- [ ] User deactivation soft-deletes (owner only)
- [ ] Integrations page shows connected inbox list with status
- [ ] Quote templates CRUD works with template config JSON
- [ ] Routing rules CRUD works (Growth+ gated), rules evaluated in priority_order
- [ ] Notification preferences matrix saves per-user (email/in_app toggles per event type)
- [ ] Billing shows current plan, usage, and Stripe integration placeholder
- [ ] Audit log shows filterable entries with expandable change JSON
- [ ] Audit log CSV export works

---

## Feature 11: QUINN Copilot (large)

**Description:** QUINN is the conversational AI chat panel available on every page. It can view RFQ details, build quotes, assign RFQs, send quotes, check analytics, and answer questions. SSE streaming, tool execution, action buttons, slash commands, conversation persistence.

**Database:** No new tables. Uses `quinn_conversations` from Feature 1.

**Backend:**
- `POST /api/v1/quinn/chat` - Chat message with SSE streaming (event types: text, tool_call, tool_result, action_button, done). Includes context injection (current page route, active RFQ/quote).
- `GET /api/v1/quinn/conversations` - List conversations
- `GET /api/v1/quinn/conversations/:id` - Conversation history
- QUINN Agent: `src/agents/quinn/agent.ts`, `prompts.ts`, `tools.ts`, `context.ts`
  - System prompt: QUINN copilot for RFQX, garment industry terminology, concise (<300 words), multi-language (EN, BN, HI, VI, TR)
  - 10 tools: `list_rfqs`, `get_rfq_details`, `build_quote`, `assign_rfq`, `send_quote`, `get_analytics`, `update_cost_library`, `search_quotes`, `set_reminder`, `get_buyer_info`
  - All tool execution respects user's RBAC permissions
  - LLM: Claude claude-opus-4-5, temp=0.4, max_tokens=1024, 3x retry
  - Context: Last 20 messages + current page context (active RFQ/quote data injected as system context)
- zustand store: `src/stores/quinn-store.ts` (panel state, messages, streaming)

**Frontend:**
- Components:
  - `src/components/quinn/quinn-panel.tsx` - Right sidebar (400px desktop, full-screen mobile), collapsible, toggle button bottom-right
  - `src/components/quinn/chat-message.tsx` - User messages (right, blue), QUINN messages (left, gray) with markdown, tables, action buttons, deep-link cards
  - `src/components/quinn/action-button.tsx` - Clickable action buttons in QUINN responses
  - `src/components/quinn/quick-actions.tsx` - "/" command autocomplete (/quote, /status, /report, /assign, /follow-up, /help)
  - Typing indicator during streaming
  - SSE event handling for text, tool_call, tool_result, action_button, done
- Prototype file: `prototypes/quinn-panel.html`

**Agent:**
- Tools to build: `list_rfqs`, `get_rfq_details`, `build_quote`, `assign_rfq`, `send_quote`, `get_analytics`, `update_cost_library`, `search_quotes`, `set_reminder`, `get_buyer_info`
- Skills to build: Context-aware conversation (page route + entity injection), slash command routing

**Depends on:** Feature 1, Feature 3, Feature 6, Feature 8

**Done when:**
- [ ] QUINN panel opens/closes from toggle button on every page
- [ ] Chat messages stream via SSE with typing indicator
- [ ] QUINN responds in markdown with tables, bold, links
- [ ] Action buttons in responses are clickable and trigger actions
- [ ] All 10 tools execute correctly (list_rfqs, build_quote, assign_rfq, etc.)
- [ ] Tool execution respects user's RBAC role
- [ ] Slash commands (/quote, /status, /report, /assign, /follow-up, /help) trigger appropriate tools
- [ ] Context injection: QUINN knows current page, active RFQ/quote data
- [ ] Conversation persistence: messages saved to quinn_conversations table
- [ ] Last 20 messages retained per conversation
- [ ] Panel is 400px on desktop, full-screen on mobile
- [ ] QUINN responds in user's language (EN, BN, HI, VI, TR)

---

## Feature 12: Email Integration & Inbox Monitoring (large)

**Description:** InboxWatcher agent for monitoring Gmail/Outlook/IMAP inboxes, detecting RFQs from email, classifying messages, and auto-creating RFQs. Includes OAuth connection flows and inbox sync worker.

**Database:** No new tables. Uses `inbox_connections`, `inbox_sync_state` from Feature 1.

**Backend:**
- `GET /api/v1/inbox-connections` - List connected inboxes
- `POST /api/v1/inbox-connections/gmail` - Connect Gmail via OAuth
- `POST /api/v1/inbox-connections/outlook` - Connect Outlook via OAuth
- `POST /api/v1/inbox-connections/imap` - Connect IMAP (host, port, username, password encrypted)
- `DELETE /api/v1/inbox-connections/:id` - Remove connection
- `POST /api/v1/webhooks/email` - Nylas email webhook handler
- InboxWatcher Agent: `src/agents/inbox-watcher/agent.ts`, `prompts.ts`, `tools.ts`
  - Tools: `fetch_emails` (IMAP/Nylas), `check_portal`, `parse_whatsapp_webhook`
  - Skills: Email classification (RFQ vs general), language detection, dedup hash
  - LLM: Claude claude-opus-4-5, temp=0.2, max_tokens=1024, 3x retry
  - Output: `{is_rfq, confidence, buyer_name, product_hint, language, dedup_hash}`
- BullMQ inbox sync worker: `src/workers/inbox-worker.ts` (consumes inbox-sync queue, concurrency 5, 3 retries)
- Scheduler: `src/workers/scheduler.ts` (cron every 60s via BullMQ repeatable job)
- Encryption for stored OAuth tokens and IMAP credentials (AES-256-GCM)

**Frontend:** Uses existing Settings > Integrations page from Feature 10 (connect/disconnect flows, last sync status)

**Agent:**
- Tools to build: `fetch_emails`, `check_portal`, `parse_whatsapp_webhook`
- Skills to build: Email classification, language detection, dedup hash

**Depends on:** Feature 1, Feature 6, Feature 7, Feature 10

**Done when:**
- [ ] Gmail OAuth flow connects and stores encrypted tokens
- [ ] Outlook OAuth flow connects and stores encrypted tokens
- [ ] IMAP connection with encrypted credentials works
- [ ] Inbox sync worker polls every 60 seconds for new emails
- [ ] Email classification correctly identifies RFQ vs non-RFQ messages
- [ ] Detected RFQs auto-create RFQ records with status "extracting"
- [ ] Attachments downloaded and stored in Supabase Storage
- [ ] Dedup hash prevents duplicate RFQ creation (7-day window)
- [ ] Language detection identifies EN, BN, HI, VI, TR
- [ ] Last sync timestamp and errors shown in Settings > Integrations
- [ ] Disconnect removes connection and stops syncing

---

## Feature 13: Quote Sending & Tracking (medium)

**Description:** Send approved quotes to buyers via email, track email opens, auto follow-up escalation (4h -> 48h -> 72h), follow-up templates, and win/loss tracking.

**Database:** No new tables. Uses `quotes`, `follow_up_templates`, `notifications` from Feature 1.

**Backend:**
- `POST /api/v1/quotes/:id/send` - Send quote to buyer via email (attach PDF/Excel, use buyer contact_email or specified recipient)
- TrackingAgent: `src/agents/tracker/agent.ts`, `prompts.ts`, `tools.ts`
  - Tools: `check_email_opens` (tracking pixel / read receipt), `check_portal_status`, `send_follow_up`, `update_quote_status`
  - Follow-up escalation: 1st at 4h, 2nd at 48h after 1st, 3rd at 72h after 2nd (max 3, min 24h gap, 3rd CCs owner)
  - LLM: Claude claude-opus-4-5, temp=0.4, max_tokens=512
- Follow-up templates: `GET/POST /api/v1/follow-up-templates` (12 template variables: buyer_name, product, quote_number, etc.)
- BullMQ workers:
  - `src/workers/quote-worker.ts` (extended for quote-tracking queue, concurrency 5)
  - `src/workers/notification-worker.ts` (extended for follow-up queue, concurrency 2)
- Scheduler: Auto-expiry checks (RFQ deadline+7d for sent, quote valid_until), tracking polls every 30min

**Frontend:** Uses existing quote detail from Feature 8 (Send button, sent_at display, opened_at display). Uses existing RFQ detail for win/loss status updates.

**Agent:**
- Tools to build: `check_email_opens`, `check_portal_status`, `send_follow_up`, `update_quote_status`
- Skills to build: Follow-up email generation, win/loss analysis

**Depends on:** Feature 1, Feature 8, Feature 9

**Done when:**
- [ ] Send quote delivers email with PDF attachment to buyer
- [ ] Quote status transitions to "sent" with sent_at timestamp
- [ ] RFQ status transitions to "sent" when quote is sent
- [ ] Email open detection updates opened_at (tracking pixel or read receipt)
- [ ] Auto follow-up: 1st at 4h, 2nd at 48h after 1st, 3rd at 72h after 2nd
- [ ] Min 24h gap between follow-ups enforced
- [ ] 3rd follow-up CCs factory owner
- [ ] No more than 3 auto follow-ups per quote
- [ ] Follow-up templates support 12 template variables (buyer_name, product, etc.)
- [ ] Auto-expiry: RFQs (deadline+7d for sent), quotes (valid_until date)
- [ ] Win/loss marking updates both quote and RFQ statuses
- [ ] won_lost_reason required (min 3 chars) when marking won/lost

---

## Feature 14: Analytics (medium)

**Description:** Analytics dashboard with performance metrics (win rate, response time, volume), buyer intelligence (scorecard, volume trends), cost intelligence (cost trends, margin erosion alerts), and export capabilities.

**Database:** No new tables. Aggregates from `rfqs`, `quotes`, `buyers`, `cost_library_items`.

**Backend:**
- `GET /api/v1/analytics/overview` - KPI summary (win rate, response time, volume, pipeline, forecast) with period comparison
- `GET /api/v1/analytics/win-rate` - Win rate breakdown by buyer/product/month + 12-month trend
- `GET /api/v1/analytics/response-time` - Avg/median hours + per-user breakdown
- `GET /api/v1/analytics/pipeline` - Pipeline value by status + by buyer + expected close (Growth+ gated)
- `GET /api/v1/analytics/buyer-scorecard` - Per-buyer metrics + volume trend (Growth+ gated)
- `GET /api/v1/analytics/cost-trends` - Cost category trends + margin erosion alert (Growth+ gated)
- `GET /api/v1/analytics/export` - Download report as XLSX/CSV
- Redis caching: analytics results cached 300s per tenant per metric

**Frontend:**
- Pages/screens:
  - `/analytics` - Date range picker + 4 tabs:
    - Performance: Win Rate (overall + trend chart 12mo), Response Time (avg + trend), RFQ Volume (monthly bar), Missed RFQs, Pipeline Value (stacked by status), Revenue Forecast
    - Buyer Intelligence: Scorecard table (Name, RFQs, Quotes, Win Rate, Avg FOB, Avg Margin, Trend arrow). Click for expanded view with volume chart. Price sensitivity scatter (Scale only).
    - Cost Intelligence: Input cost trend lines (Fabric, CMT, Trims 12mo). Margin erosion alert panel. Cost vs Budget (Scale only).
    - Export: "Export to Excel" and "Export to CSV" per view. Scheduled report setup (Growth+).
- Components:
  - `src/components/analytics/win-rate-chart.tsx` (recharts line + bar)
  - `src/components/analytics/response-time-chart.tsx`
  - `src/components/analytics/buyer-scorecard.tsx` (table with trend arrows)
  - `src/components/analytics/cost-trend-chart.tsx` (line chart)
- Prototype file: `prototypes/analytics.html`

**Agent:** None

**Depends on:** Feature 1, Feature 3, Feature 6, Feature 8

**Done when:**
- [ ] Overview shows win rate, response time, volume, pipeline, forecast with period comparison
- [ ] Win rate calculated as won / (won + lost) * 100 (expired excluded)
- [ ] Response time = avg hours from rfq.created_at to quote.sent_at
- [ ] Pipeline value = sum of fob_total for active quotes (latest version only)
- [ ] Revenue forecast = pipeline * win_rate (30% default if no history)
- [ ] Win rate breakdown by buyer/product/month with 12-month trend chart
- [ ] Per-user response time breakdown
- [ ] Pipeline analytics gated to Growth+ plan
- [ ] Buyer scorecard shows per-buyer metrics with volume trend (Growth+ gated)
- [ ] Cost trends show 12-month category trends (Growth+ gated)
- [ ] Margin erosion alert triggers when avg margin drops >2% month-over-month
- [ ] Export to XLSX/CSV works for each analytics view
- [ ] Analytics cached in Redis (300s TTL per tenant per metric)

---

## Feature 15: Billing & Stripe Integration (small)

**Description:** Stripe subscription management for the 3 pricing tiers (Starter $299, Growth $599, Scale $999), plan upgrades/downgrades, billing portal, webhook handling.

**Database:** No new tables. Updates `tenants.stripe_customer_id`, `tenants.subscription_status`, `tenants.plan`.

**Backend:**
- Stripe integration:
  - Create Stripe customer on tenant signup
  - Create subscription checkout session for plan selection
  - Handle Stripe webhooks: `POST /api/v1/webhooks/stripe` (subscription created, updated, canceled, payment failed)
  - Billing portal session for card management / invoice history
  - Subscription status check (cached 5 min): trialing, active, past_due, canceled
  - Grace period: 7 days on failed payment before account suspension
  - Trial: 14 days, no credit card required
  - Plan change updates tenant limits (max_rfqs_per_month, max_users, feature gates)

**Frontend:** Uses existing Settings > Billing page from Feature 10 (plan card, upgrade modal, payment method, invoice history)

**Agent:** None

**Depends on:** Feature 1, Feature 10

**Done when:**
- [ ] Stripe customer created on tenant signup
- [ ] Plan selection triggers Stripe checkout session
- [ ] Webhook handles subscription lifecycle (created, updated, canceled, payment_failed)
- [ ] Plan upgrade/downgrade updates tenant limits and feature gates immediately
- [ ] Billing portal opens for card management and invoice viewing
- [ ] 14-day trial works without credit card
- [ ] 7-day grace period on failed payment before suspension
- [ ] Subscription status cached 5 min, checked on API requests
- [ ] Billing page shows current plan, usage (X/Y RFQs, user count), and invoice history

---

## Feature 16: Security, Performance & Polish (medium)

**Description:** Rate limiting, security headers, Sentry error tracking, Redis caching layer, loading skeletons, error handling, and responsive testing.

**Database:** No new tables.

**Backend:**
- Rate limiting middleware: Auth 10/min/IP, QUINN 30/min/user, upload 20/min/user, general 100/min/user, Scale 1000/min/tenant
- Security headers: CSP, HSTS, X-Content-Type-Options, X-Frame-Options, Permissions-Policy, CORS policy
- Sentry integration for error tracking with user/tenant context
- Redis caching: tenant config (300s), cost library (60s), exchange rates (3600s), analytics (300s), user session (86400s)
- Structured logging via pino (production level: warn)

**Frontend:**
- Loading skeletons across all data-fetching areas
- Error boundaries with user-friendly error messages
- Toast notifications for success/error feedback
- Responsive testing: desktop (>=1280px), tablet (768-1279px), mobile (<768px)
- Tables become card lists on mobile

**Agent:** None

**Depends on:** All previous features

**Done when:**
- [ ] Rate limiting returns 429 with Retry-After header when exceeded
- [ ] All security headers present in responses
- [ ] Sentry captures unhandled exceptions with user/tenant context
- [ ] Redis caching reduces database load for frequently accessed data
- [ ] All pages show loading skeletons during data fetch
- [ ] Error boundaries catch and display errors gracefully
- [ ] Toast notifications appear for all user actions (create, update, delete, send)
- [ ] Mobile responsive: tables become cards, sidebar collapses, QUINN goes full-screen
- [ ] All Core Web Vitals pass (TTFB < 500ms, LCP < 2.5s)
