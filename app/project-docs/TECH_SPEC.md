# RFQX Technical Specification
## Version 1.0 | March 2026

---

## 1. Technology Stack

| Layer | Technology | Version |
|---|---|---|
| Frontend | Next.js (App Router) | 14.x |
| Language | TypeScript | 5.x |
| CSS | Tailwind CSS | 3.x |
| UI Components | shadcn/ui | latest |
| Backend | Next.js API Routes + standalone Node.js workers | 14.x / Node 20 LTS |
| Database | Supabase (PostgreSQL 15 + pgvector) | latest |
| Auth | Supabase Auth (email + magic link + Google OAuth) | latest |
| File Storage | Supabase Storage | latest |
| Cache | Redis (Upstash or Railway-hosted) | 7.x |
| Queue | BullMQ over Redis | 5.x |
| AI / LLM | Anthropic Claude claude-opus-4-5 via API | claude-opus-4-5 |
| OCR | Tesseract.js (client-side fallback) + Google Cloud Vision API | latest |
| PDF Generation | @react-pdf/renderer + Puppeteer (server-side) | latest |
| Email | Nylas API (Gmail/Outlook unified) or direct IMAP via imapflow | latest |
| Real-time | Supabase Realtime (WebSocket subscriptions) | latest |
| Deployment (Frontend) | Vercel | latest |
| Deployment (Workers) | Railway | latest |
| Monitoring | Sentry (error tracking) + Vercel Analytics | latest |
| Testing | Vitest (unit) + Playwright (e2e) | latest |

---

## 2. AI Agent Architecture

### 2.1 Agent Orchestration Model

RFQX uses a **code-orchestrated multi-agent architecture**. There is no autonomous agent loop. A deterministic TypeScript orchestrator dispatches tasks to specialized AI sub-agents via function calls.

```
User / QUINN Chat
       |
  [Orchestrator] ── deterministic router (TypeScript)
       |
  ┌────┼────────┬──────────┬─────────────┐
  v    v        v          v             v
Inbox  Extractor CostEngine QuoteWriter  Tracker
Agent  Agent     Agent      Agent        Agent
```

### 2.2 Sub-Agent Definitions

#### InboxWatcher Agent
- **Trigger**: Cron job every 60 seconds (BullMQ repeatable job)
- **Tools**: `fetch_emails`, `check_portal`, `parse_whatsapp_webhook`
- **Skills**: Email classification (RFQ vs general), language detection, deduplication hash
- **Memory**: Last processed email UID per inbox (stored in `inbox_sync_state` table)
- **LLM Config**: Claude claude-opus-4-5, temperature 0.2, max_tokens 1024
- **System Prompt**: "You are an RFQ detection classifier for garment manufacturing. Given an email subject, body snippet, and attachment filenames, determine if this is an RFQ. Return JSON: {is_rfq: boolean, confidence: number, buyer_name: string | null, product_hint: string | null, language: string}. Only classify as RFQ if the message requests pricing, costing, or quotation for a garment product."
- **Output Schema**:
```typescript
interface InboxClassification {
  is_rfq: boolean;
  confidence: number; // 0.0 - 1.0
  buyer_name: string | null;
  product_hint: string | null;
  language: "en" | "bn" | "hi" | "vi" | "tr" | "zh";
  dedup_hash: string; // SHA-256 of normalized subject + sender + date
}
```

#### ExtractorAgent
- **Trigger**: New RFQ record created (status = "extracting")
- **Tools**: `ocr_document`, `parse_pdf`, `parse_excel`, `parse_email_body`
- **Skills**: Garment spec extraction (active), Footwear extraction, Home Textile extraction
- **Memory**: Extraction correction history per tenant (used for few-shot prompt augmentation)
- **LLM Config**: Claude claude-opus-4-5, temperature 0.1, max_tokens 4096
- **System Prompt**: "You are a garment specification extraction engine. Given raw text from an RFQ document, extract all product specifications into structured JSON. Use exact garment industry terminology. For every field, provide a confidence score (0-100). Fields: product_type, style_number, season, fabric_composition, fabric_gsm, fabric_construction, yarn_count, colorways[], total_quantity, size_breakdown{}, delivery_date, delivery_port, incoterm, payment_terms, buyer_reference, certifications[], testing_requirements[], packaging_specs, print_embroidery_details. If a field is not found in the source, set it to null with confidence 0."
- **Output Schema**:
```typescript
interface ExtractedSpecs {
  product_type: { value: string; confidence: number };
  style_number: { value: string | null; confidence: number };
  season: { value: string | null; confidence: number };
  fabric_composition: { value: string; confidence: number };
  fabric_gsm: { value: number | null; confidence: number };
  fabric_construction: { value: string | null; confidence: number };
  yarn_count: { value: string | null; confidence: number };
  colorways: { value: string[]; confidence: number };
  total_quantity: { value: number; confidence: number };
  size_breakdown: { value: Record<string, number>; confidence: number };
  delivery_date: { value: string | null; confidence: number }; // ISO 8601
  delivery_port: { value: string | null; confidence: number };
  incoterm: { value: "FOB" | "CIF" | "EXW" | "DDP" | null; confidence: number };
  payment_terms: { value: string | null; confidence: number };
  buyer_reference: { value: string | null; confidence: number };
  certifications: { value: string[]; confidence: number };
  testing_requirements: { value: string[]; confidence: number };
  packaging_specs: { value: string | null; confidence: number };
  print_embroidery_details: { value: string | null; confidence: number };
}
```

#### CostEngine Agent
- **Trigger**: Extraction complete, user approves specs (or auto-approve if all fields >= 85% confidence)
- **Tools**: `lookup_cost_library`, `get_exchange_rate`, `calculate_cmt`, `calculate_fabric_cost`
- **Skills**: FOB calculation, CIF calculation, landed cost calculation
- **Memory**: Last used cost snapshot per tenant (for version comparison)
- **LLM Config**: Claude claude-opus-4-5, temperature 0.0, max_tokens 2048
- **Reasoning**: Chain-of-thought cost breakdown with explicit formula application
- **System Prompt**: "You are a garment costing engine. Given extracted product specs and a cost library, calculate a complete cost breakdown. Apply these rules strictly: 1) Fabric cost = (consumption_per_unit_meters * cost_per_meter) or (consumption_per_unit_kg * cost_per_kg). 2) CMT cost = rate from CMT rate card matching product_type and quantity bracket. 3) Trim cost = sum of individual trim items from cost library. 4) Washing cost = rate per unit from wash type. 5) Embellishment cost = (stitch_count / 1000) * rate_per_1000_stitches for embroidery; per_color * num_colors for print. 6) Packing cost = per-unit packing rate. 7) Overhead = overhead_percentage * CMT. 8) Testing = total_test_cost / total_quantity. 9) Commission = commission_percentage * subtotal. 10) Finance cost = finance_percentage * subtotal. 11) FOB = sum of all above. 12) Freight for CIF = FOB + freight_per_unit. Show every step. All amounts in USD unless specified."
- **Output Schema**:
```typescript
interface CostBreakdown {
  currency: string;
  fabric: { per_unit: number; total: number; formula: string };
  cmt: { per_unit: number; total: number; rate_card_ref: string };
  trims: { per_unit: number; total: number; items: TrimLineItem[] };
  washing: { per_unit: number; total: number; wash_type: string };
  embellishment: { per_unit: number; total: number; details: string };
  packing: { per_unit: number; total: number };
  overhead: { per_unit: number; total: number; percentage: number };
  testing: { per_unit: number; total: number };
  commission: { per_unit: number; total: number; percentage: number };
  finance_cost: { per_unit: number; total: number; percentage: number };
  freight: { per_unit: number; total: number; port: string };
  cost_per_unit: number;
  total_cost: number;
  margin_percentage: number;
  margin_per_unit: number;
  fob_per_unit: number;
  fob_total: number;
  incoterm: string;
  quantity: number;
}
```

#### QuoteWriter Agent
- **Trigger**: User approves cost breakdown or requests quote generation
- **Tools**: `get_buyer_template`, `generate_pdf`, `generate_excel`, `get_factory_letterhead`
- **Skills**: Template formatting, multi-language quote generation
- **Memory**: Buyer template preferences (stored in `buyers` table)
- **LLM Config**: Claude claude-opus-4-5, temperature 0.3, max_tokens 2048
- **System Prompt**: "You are a professional quote document writer for garment manufacturing. Given a cost breakdown and buyer template specification, generate a formatted quote document. Include: factory name, buyer name, style reference, product description, quantity, size breakdown, fabric details, FOB/CIF price per unit, total order value, delivery terms, payment terms, validity period (default 15 days), and any notes. Be professional and concise. Match the buyer's template format exactly."

#### TrackingAgent
- **Trigger**: Quote sent (status = "sent"), then periodic check every 30 minutes
- **Tools**: `check_email_opens`, `check_portal_status`, `send_follow_up`, `update_quote_status`
- **Skills**: Follow-up email generation, win/loss analysis
- **Memory**: Follow-up count per quote, last follow-up timestamp
- **LLM Config**: Claude claude-opus-4-5, temperature 0.4, max_tokens 512
- **System Prompt**: "You are a professional follow-up assistant for garment factory sales. Generate polite, concise follow-up messages for quote responses. Reference the specific quote, buyer name, product, and quantity. Keep messages under 100 words. Tone: professional, warm, not pushy."

### 2.3 QUINN Copilot Architecture

- **Interface**: Chat panel (right sidebar), available on every page
- **LLM Config**: Claude claude-opus-4-5, temperature 0.4, max_tokens 1024
- **Context Window Management**: Last 20 messages + current page context (active RFQ/quote data injected as system context)
- **Tool Routing**: QUINN has access to all sub-agent tools via MCP-style tool definitions
- **System Prompt**: "You are QUINN, the AI copilot for RFQX — a quoting platform for garment manufacturers. You help factory teams manage RFQs and quotes through natural conversation. You can: view RFQ details, build quotes, assign RFQs to team members, send quotes, check analytics, and answer questions about the platform. Always be concise (under 300 words). When taking actions, confirm with the user before executing. You understand English, Bengali, Hindi, Vietnamese, and Turkish. Use garment industry terminology naturally (FOB, CMT, GSM, AQL, TNA, BOM). Format responses with markdown tables and action buttons where helpful."

**QUINN Tool Definitions**:
```typescript
const quinnTools = [
  { name: "list_rfqs", description: "List RFQs with optional filters", params: { status?, buyer?, priority?, assigned_to?, date_range? } },
  { name: "get_rfq_details", description: "Get full details of a specific RFQ", params: { rfq_id } },
  { name: "build_quote", description: "Trigger cost calculation and quote generation", params: { rfq_id, margin_percentage } },
  { name: "assign_rfq", description: "Assign RFQ to a team member", params: { rfq_id, user_id } },
  { name: "send_quote", description: "Send approved quote to buyer", params: { quote_id, channel } },
  { name: "get_analytics", description: "Retrieve analytics data", params: { metric, date_range?, buyer_id? } },
  { name: "update_cost_library", description: "Update a cost library entry", params: { item_id, new_value } },
  { name: "search_quotes", description: "Search historical quotes", params: { query, filters? } },
  { name: "set_reminder", description: "Set a follow-up reminder", params: { quote_id, remind_at } },
  { name: "get_buyer_info", description: "Get buyer profile and history", params: { buyer_id } },
];
```

---

## 3. Database Schema

All tables use Supabase (PostgreSQL 15). UUIDs for primary keys. Row Level Security (RLS) enabled on all tables with tenant isolation.

### 3.1 Core Tables

#### tenants
```sql
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,
  plan VARCHAR(20) NOT NULL DEFAULT 'starter' CHECK (plan IN ('starter', 'growth', 'scale')),
  base_currency VARCHAR(3) NOT NULL DEFAULT 'USD',
  max_rfqs_per_month INTEGER NOT NULL DEFAULT 20,
  max_users INTEGER NOT NULL DEFAULT 2,
  logo_url TEXT,
  letterhead_url TEXT,
  address TEXT,
  phone VARCHAR(50),
  website VARCHAR(255),
  default_margin_percentage NUMERIC(5,2) NOT NULL DEFAULT 15.00,
  default_incoterm VARCHAR(3) NOT NULL DEFAULT 'FOB' CHECK (default_incoterm IN ('FOB', 'CIF', 'EXW', 'DDP')),
  default_quote_validity_days INTEGER NOT NULL DEFAULT 15,
  overhead_percentage NUMERIC(5,2) NOT NULL DEFAULT 10.00,
  commission_percentage NUMERIC(5,2) NOT NULL DEFAULT 5.00,
  finance_cost_percentage NUMERIC(5,2) NOT NULL DEFAULT 2.00,
  stripe_customer_id VARCHAR(255),
  subscription_status VARCHAR(20) DEFAULT 'trialing' CHECK (subscription_status IN ('trialing', 'active', 'past_due', 'canceled')),
  trial_ends_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);
CREATE INDEX idx_tenants_plan ON tenants(plan);
```

#### users
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  email VARCHAR(255) NOT NULL,
  full_name VARCHAR(255) NOT NULL,
  role VARCHAR(30) NOT NULL DEFAULT 'viewer' CHECK (role IN ('owner', 'admin', 'quotation_manager', 'merchandiser', 'commercial_executive', 'viewer')),
  avatar_url TEXT,
  phone VARCHAR(50),
  is_active BOOLEAN NOT NULL DEFAULT true,
  mfa_enabled BOOLEAN NOT NULL DEFAULT false,
  notification_preferences JSONB NOT NULL DEFAULT '{"email": true, "in_app": true, "quinn_alerts": true}',
  last_login_at TIMESTAMPTZ,
  supabase_auth_id UUID UNIQUE NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_supabase_auth ON users(supabase_auth_id);
CREATE INDEX idx_users_role ON users(tenant_id, role);
```

#### buyers
```sql
CREATE TABLE buyers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  code VARCHAR(50),
  contact_name VARCHAR(255),
  contact_email VARCHAR(255),
  contact_phone VARCHAR(50),
  country VARCHAR(100),
  tier INTEGER NOT NULL DEFAULT 3 CHECK (tier BETWEEN 1 AND 5),
  preferred_incoterm VARCHAR(3) DEFAULT 'FOB',
  preferred_currency VARCHAR(3) DEFAULT 'USD',
  payment_terms VARCHAR(100),
  quote_template_id UUID REFERENCES quote_templates(id),
  portal_type VARCHAR(50), -- 'primark', 'hm', 'asos', 'zara', null
  portal_credentials_encrypted TEXT, -- encrypted JSON
  commission_percentage NUMERIC(5,2) DEFAULT 0.00,
  notes TEXT,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(tenant_id, code)
);

CREATE INDEX idx_buyers_tenant ON buyers(tenant_id);
CREATE INDEX idx_buyers_name ON buyers(tenant_id, name);
CREATE INDEX idx_buyers_tier ON buyers(tenant_id, tier);
```

#### rfqs
```sql
CREATE TABLE rfqs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  rfq_number VARCHAR(50) NOT NULL, -- auto-generated: RFQ-YYYYMM-XXXX
  buyer_id UUID REFERENCES buyers(id),
  assigned_user_id UUID REFERENCES users(id),
  status VARCHAR(30) NOT NULL DEFAULT 'new' CHECK (status IN ('new', 'extracting', 'extracted', 'costing', 'review', 'approved', 'sent', 'followed_up', 'won', 'lost', 'expired', 'archived')),
  priority INTEGER NOT NULL DEFAULT 3 CHECK (priority BETWEEN 1 AND 5),
  source_channel VARCHAR(20) NOT NULL CHECK (source_channel IN ('email', 'portal', 'whatsapp', 'upload', 'api')),
  source_reference TEXT, -- email message ID, portal RFQ ID, etc.
  source_email_from VARCHAR(255),
  source_email_subject TEXT,
  original_language VARCHAR(5) DEFAULT 'en',
  extracted_data JSONB, -- ExtractedSpecs JSON
  extraction_confidence_avg NUMERIC(5,2),
  fields_flagged_for_review TEXT[], -- field names with confidence < 85
  manual_overrides JSONB DEFAULT '{}', -- {field_name: {original, override, user_id, timestamp}}
  product_type VARCHAR(100),
  total_quantity INTEGER,
  estimated_value_usd NUMERIC(12,2),
  deadline TIMESTAMPTZ,
  delivery_date DATE,
  incoterm VARCHAR(3),
  dedup_hash VARCHAR(64), -- SHA-256 for deduplication
  merged_with_rfq_id UUID REFERENCES rfqs(id),
  attachments TEXT[], -- Supabase Storage paths
  notes TEXT,
  won_lost_reason TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  extracted_at TIMESTAMPTZ,
  sent_at TIMESTAMPTZ,
  responded_at TIMESTAMPTZ,
  closed_at TIMESTAMPTZ
);

CREATE INDEX idx_rfqs_tenant ON rfqs(tenant_id);
CREATE INDEX idx_rfqs_status ON rfqs(tenant_id, status);
CREATE INDEX idx_rfqs_buyer ON rfqs(tenant_id, buyer_id);
CREATE INDEX idx_rfqs_assigned ON rfqs(tenant_id, assigned_user_id);
CREATE INDEX idx_rfqs_priority ON rfqs(tenant_id, priority DESC);
CREATE INDEX idx_rfqs_deadline ON rfqs(tenant_id, deadline);
CREATE INDEX idx_rfqs_created ON rfqs(tenant_id, created_at DESC);
CREATE INDEX idx_rfqs_dedup ON rfqs(tenant_id, dedup_hash);
CREATE INDEX idx_rfqs_number ON rfqs(rfq_number);
CREATE INDEX idx_rfqs_product_type ON rfqs(tenant_id, product_type);
```

#### quotes
```sql
CREATE TABLE quotes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  rfq_id UUID NOT NULL REFERENCES rfqs(id) ON DELETE CASCADE,
  quote_number VARCHAR(50) NOT NULL, -- auto-generated: QT-YYYYMM-XXXX
  version INTEGER NOT NULL DEFAULT 1,
  status VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'pending_review', 'approved', 'sent', 'revised', 'accepted', 'rejected', 'expired')),
  cost_breakdown JSONB NOT NULL, -- CostBreakdown JSON
  cost_snapshot_id UUID REFERENCES cost_library_snapshots(id),
  currency VARCHAR(3) NOT NULL DEFAULT 'USD',
  exchange_rate_to_usd NUMERIC(10,4) DEFAULT 1.0000,
  exchange_rate_locked BOOLEAN NOT NULL DEFAULT false,
  total_quantity INTEGER NOT NULL,
  cost_per_unit NUMERIC(10,4) NOT NULL,
  total_cost NUMERIC(12,2) NOT NULL,
  margin_percentage NUMERIC(5,2) NOT NULL,
  margin_per_unit NUMERIC(10,4) NOT NULL,
  fob_per_unit NUMERIC(10,4) NOT NULL,
  fob_total NUMERIC(12,2) NOT NULL,
  incoterm VARCHAR(3) NOT NULL DEFAULT 'FOB',
  freight_per_unit NUMERIC(10,4) DEFAULT 0.0000,
  landed_price_per_unit NUMERIC(10,4), -- for CIF/DDP
  buyer_template_id UUID REFERENCES quote_templates(id),
  pdf_url TEXT, -- Supabase Storage path
  excel_url TEXT,
  validity_days INTEGER NOT NULL DEFAULT 15,
  valid_until DATE,
  sent_via VARCHAR(20), -- 'email', 'portal', 'whatsapp', 'manual'
  sent_at TIMESTAMPTZ,
  opened_at TIMESTAMPTZ,
  buyer_counter_offer JSONB, -- {price_per_unit, notes, received_at}
  scenarios JSONB, -- [{name, margin, fob_per_unit, cost_breakdown}]
  approved_by UUID REFERENCES users(id),
  approved_at TIMESTAMPTZ,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_quotes_tenant ON quotes(tenant_id);
CREATE INDEX idx_quotes_rfq ON quotes(rfq_id);
CREATE INDEX idx_quotes_status ON quotes(tenant_id, status);
CREATE INDEX idx_quotes_number ON quotes(quote_number);
CREATE INDEX idx_quotes_created ON quotes(tenant_id, created_at DESC);
CREATE INDEX idx_quotes_buyer_template ON quotes(buyer_template_id);
```

#### cost_library_items
```sql
CREATE TABLE cost_library_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  category VARCHAR(30) NOT NULL CHECK (category IN ('fabric', 'cmt', 'trims', 'embellishment', 'washing', 'testing', 'packing', 'overhead', 'freight', 'commission', 'finance')),
  name VARCHAR(255) NOT NULL,
  code VARCHAR(50), -- supplier code or internal code
  unit VARCHAR(30) NOT NULL, -- 'per_meter', 'per_kg', 'per_unit', 'per_1000_stitches', 'per_color', 'percentage', 'per_test'
  cost NUMERIC(10,4) NOT NULL,
  currency VARCHAR(3) NOT NULL DEFAULT 'USD',
  supplier_name VARCHAR(255),
  supplier_code VARCHAR(100),

  -- Fabric-specific fields
  fabric_composition VARCHAR(255), -- e.g., '100% cotton', '65/35 poly-cotton'
  fabric_gsm INTEGER,
  fabric_construction VARCHAR(100), -- 'pique', 'poplin', 'twill', 'jersey'
  fabric_width_cm INTEGER,
  consumption_per_unit_meters NUMERIC(8,4), -- meters per garment
  consumption_per_unit_kg NUMERIC(8,4),

  -- CMT-specific fields
  product_type VARCHAR(100), -- 'polo_shirt', 'trouser', 'jacket', etc.
  complexity_tier VARCHAR(10) CHECK (complexity_tier IN ('basic', 'medium', 'complex')),
  sam_minutes NUMERIC(8,2), -- Standard Allowed Minutes
  moq_min INTEGER, -- minimum quantity for this rate
  moq_max INTEGER, -- maximum quantity for this rate

  -- Washing-specific
  wash_type VARCHAR(50), -- 'normal', 'enzyme', 'stone', 'silicon', 'acid'

  -- Freight-specific
  origin_port VARCHAR(100),
  destination_port VARCHAR(100),

  -- General
  effective_from DATE NOT NULL DEFAULT CURRENT_DATE,
  effective_to DATE,
  is_active BOOLEAN NOT NULL DEFAULT true,
  notes TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cost_library_tenant ON cost_library_items(tenant_id);
CREATE INDEX idx_cost_library_category ON cost_library_items(tenant_id, category);
CREATE INDEX idx_cost_library_product ON cost_library_items(tenant_id, product_type);
CREATE INDEX idx_cost_library_fabric ON cost_library_items(tenant_id, fabric_composition, fabric_gsm);
CREATE INDEX idx_cost_library_active ON cost_library_items(tenant_id, is_active, category);
CREATE INDEX idx_cost_library_supplier ON cost_library_items(tenant_id, supplier_code);
```

#### cost_library_snapshots
```sql
CREATE TABLE cost_library_snapshots (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  snapshot_data JSONB NOT NULL, -- full copy of cost_library_items at point in time
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by UUID REFERENCES users(id)
);

CREATE INDEX idx_cost_snapshots_tenant ON cost_library_snapshots(tenant_id);
CREATE INDEX idx_cost_snapshots_created ON cost_library_snapshots(tenant_id, created_at DESC);
```

#### quote_templates
```sql
CREATE TABLE quote_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL, -- 'H&M CIQ', 'Primark Supplier Form', 'Default'
  buyer_name VARCHAR(255), -- null = generic template
  template_type VARCHAR(20) NOT NULL DEFAULT 'pdf' CHECK (template_type IN ('pdf', 'excel')),
  template_config JSONB NOT NULL, -- layout config: sections, field order, styling
  is_default BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_tenant ON quote_templates(tenant_id);
```

### 3.2 Supporting Tables

#### inbox_connections
```sql
CREATE TABLE inbox_connections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  connection_type VARCHAR(20) NOT NULL CHECK (connection_type IN ('gmail', 'outlook', 'imap', 'portal', 'whatsapp')),
  email_address VARCHAR(255),
  display_name VARCHAR(255),
  credentials_encrypted TEXT NOT NULL, -- encrypted OAuth tokens or IMAP credentials
  portal_type VARCHAR(50), -- for portal connections
  is_active BOOLEAN NOT NULL DEFAULT true,
  last_sync_at TIMESTAMPTZ,
  sync_error TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inbox_conn_tenant ON inbox_connections(tenant_id);
CREATE INDEX idx_inbox_conn_type ON inbox_connections(tenant_id, connection_type);
```

#### inbox_sync_state
```sql
CREATE TABLE inbox_sync_state (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  connection_id UUID NOT NULL REFERENCES inbox_connections(id) ON DELETE CASCADE,
  last_uid BIGINT, -- IMAP UID
  last_history_id VARCHAR(100), -- Gmail history ID
  last_sync_token TEXT, -- Outlook delta token
  last_processed_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_sync_state_conn ON inbox_sync_state(connection_id);
```

#### rfq_attachments
```sql
CREATE TABLE rfq_attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  rfq_id UUID NOT NULL REFERENCES rfqs(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  file_name VARCHAR(500) NOT NULL,
  file_type VARCHAR(50) NOT NULL, -- 'pdf', 'xlsx', 'docx', 'png', 'jpg', 'csv'
  file_size_bytes INTEGER NOT NULL,
  storage_path TEXT NOT NULL, -- Supabase Storage path
  ocr_text TEXT, -- extracted text from OCR
  extracted_data JSONB, -- per-attachment extraction results
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachments_rfq ON rfq_attachments(rfq_id);
CREATE INDEX idx_attachments_tenant ON rfq_attachments(tenant_id);
```

#### exchange_rates
```sql
CREATE TABLE exchange_rates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  base_currency VARCHAR(3) NOT NULL DEFAULT 'USD',
  target_currency VARCHAR(3) NOT NULL,
  rate NUMERIC(12,6) NOT NULL,
  source VARCHAR(50) NOT NULL DEFAULT 'manual', -- 'manual', 'api'
  fetched_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_exchange_rates_pair ON exchange_rates(base_currency, target_currency, fetched_at DESC);
```

#### notifications
```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  type VARCHAR(50) NOT NULL, -- 'new_rfq', 'deadline_approaching', 'quote_opened', 'buyer_response', 'assignment', 'mention', 'system'
  title VARCHAR(255) NOT NULL,
  body TEXT,
  link VARCHAR(500), -- deep link to relevant page
  rfq_id UUID REFERENCES rfqs(id),
  quote_id UUID REFERENCES quotes(id),
  is_read BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_user ON notifications(user_id, is_read, created_at DESC);
CREATE INDEX idx_notifications_tenant ON notifications(tenant_id);
```

#### audit_log
```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id),
  action VARCHAR(100) NOT NULL, -- 'rfq.created', 'quote.approved', 'cost_library.updated', etc.
  entity_type VARCHAR(50) NOT NULL, -- 'rfq', 'quote', 'cost_library_item', 'user', 'buyer'
  entity_id UUID NOT NULL,
  changes JSONB, -- {field: {old: x, new: y}}
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at DESC);
CREATE INDEX idx_audit_action ON audit_log(tenant_id, action);
```

#### quinn_conversations
```sql
CREATE TABLE quinn_conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  messages JSONB NOT NULL DEFAULT '[]', -- [{role, content, timestamp, tool_calls?}]
  context_rfq_id UUID REFERENCES rfqs(id),
  context_quote_id UUID REFERENCES quotes(id),
  context_page VARCHAR(100), -- current page route for context injection
  message_count INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_quinn_user ON quinn_conversations(user_id, updated_at DESC);
CREATE INDEX idx_quinn_tenant ON quinn_conversations(tenant_id);
```

#### auto_routing_rules
```sql
CREATE TABLE auto_routing_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  rule_name VARCHAR(255) NOT NULL,
  condition_type VARCHAR(30) NOT NULL CHECK (condition_type IN ('buyer', 'product_type', 'priority', 'channel')),
  condition_value VARCHAR(255) NOT NULL, -- buyer_id, product type name, priority number, channel name
  assign_to_user_id UUID NOT NULL REFERENCES users(id),
  is_active BOOLEAN NOT NULL DEFAULT true,
  priority_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_routing_rules_tenant ON auto_routing_rules(tenant_id, is_active, priority_order);
```

#### follow_up_templates
```sql
CREATE TABLE follow_up_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  subject_template TEXT NOT NULL,
  body_template TEXT NOT NULL, -- supports {{buyer_name}}, {{product}}, {{quote_number}} variables
  buyer_id UUID REFERENCES buyers(id), -- null = generic
  is_default BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_followup_templates_tenant ON follow_up_templates(tenant_id);
```

### 3.3 Entity Relationships

```
tenants 1──N users
tenants 1──N buyers
tenants 1──N rfqs
tenants 1──N quotes
tenants 1──N cost_library_items
tenants 1──N cost_library_snapshots
tenants 1──N quote_templates
tenants 1──N inbox_connections
tenants 1──N auto_routing_rules
tenants 1──N follow_up_templates
tenants 1──N audit_log
tenants 1──N notifications

buyers 1──N rfqs
buyers 1──1 quote_templates (preferred)

rfqs 1──N quotes
rfqs 1──N rfq_attachments
rfqs N──1 users (assigned_user)

quotes N──1 rfqs
quotes N──1 users (created_by)
quotes N──1 users (approved_by)
quotes N──1 quote_templates
quotes N──1 cost_library_snapshots

inbox_connections 1──1 inbox_sync_state

users 1──N notifications
users 1──N quinn_conversations
users 1──N audit_log
```

### 3.4 Row Level Security (RLS) Policy

All tables enforce tenant isolation:

```sql
-- Example RLS policy (applied to all tenant-scoped tables)
ALTER TABLE rfqs ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON rfqs
  USING (tenant_id = (SELECT tenant_id FROM users WHERE supabase_auth_id = auth.uid()));

CREATE POLICY tenant_insert ON rfqs
  FOR INSERT WITH CHECK (tenant_id = (SELECT tenant_id FROM users WHERE supabase_auth_id = auth.uid()));
```

---

## 4. Queue Architecture (BullMQ)

| Queue Name | Purpose | Concurrency | Retry |
|---|---|---|---|
| `inbox-sync` | Poll email inboxes and portals | 5 | 3 retries, exponential backoff |
| `rfq-extraction` | Run AI extraction on new RFQs | 3 | 2 retries |
| `cost-calculation` | Build cost breakdowns | 5 | 2 retries |
| `quote-generation` | Generate PDF/Excel quotes | 3 | 2 retries |
| `quote-tracking` | Check email opens, portal status | 5 | 3 retries |
| `notifications` | Send email/in-app notifications | 10 | 3 retries |
| `follow-up` | Send automated follow-up emails | 2 | 2 retries |

All queues use Redis as the backing store. Dead letter queue enabled for all queues after max retries.

---

## 5. File Storage Structure (Supabase Storage)

```
rfqx-storage/
  {tenant_id}/
    rfq-attachments/
      {rfq_id}/
        original/         -- uploaded files
        ocr/              -- OCR text output
    quotes/
      {quote_id}/
        quote-v{version}.pdf
        quote-v{version}.xlsx
    letterheads/
      logo.png
      letterhead.pdf
    cost-library-imports/
      {import_id}.xlsx
```

---

## 6. Caching Strategy (Redis)

| Key Pattern | TTL | Purpose |
|---|---|---|
| `tenant:{id}:config` | 300s | Tenant settings (plan, defaults) |
| `tenant:{id}:cost_library:{category}` | 60s | Cost library items by category |
| `exchange_rates:{base}:{target}` | 3600s | Exchange rates |
| `user:{id}:session` | 86400s | User session data |
| `rfq:{id}:extraction` | 0 (no TTL) | Extraction results (cleared on re-extract) |
| `analytics:{tenant_id}:{metric}:{period}` | 300s | Pre-computed analytics |

---

## 7. Deployment Architecture

```
Vercel (Edge Network)
  ├── Next.js Frontend (SSR + Static)
  ├── API Routes (/api/*)
  └── Serverless Functions (Quinn chat, webhooks)

Railway
  ├── Worker Process 1: InboxWatcher (BullMQ consumer)
  ├── Worker Process 2: Extraction + Costing (BullMQ consumer)
  ├── Worker Process 3: QuoteGen + Tracking (BullMQ consumer)
  └── Redis Instance

Supabase (Managed)
  ├── PostgreSQL 15 + pgvector
  ├── Auth Service
  ├── Storage (S3-compatible)
  └── Realtime (WebSocket)

External APIs
  ├── Anthropic Claude API
  ├── Google Cloud Vision (OCR)
  ├── Nylas (Email)
  └── Stripe (Billing)
```

---

*This document is confidential and intended for internal use only.*
*fabricXai is a product of SocioFi Technology Ltd.*
