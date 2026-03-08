# RFQX API Specification
## Version 1.0 | March 2026

All endpoints are prefixed with `/api/v1`. Authentication via Supabase Auth JWT in `Authorization: Bearer <token>` header. All request/response bodies are JSON unless otherwise noted. Tenant context is derived from the authenticated user's `tenant_id`.

---

## 1. Authentication

### POST `/api/v1/auth/signup`
**Auth Required**: No
```typescript
// Request
{
  email: string;          // required, valid email
  password: string;       // required, min 8 chars
  full_name: string;      // required
  factory_name: string;   // required
  country: string;        // required, ISO 3166-1 alpha-2
}

// Response 201
{
  user: { id: string; email: string; full_name: string; role: "owner" };
  tenant: { id: string; name: string; slug: string; plan: "starter" };
  session: { access_token: string; refresh_token: string; expires_at: number };
}
```

### POST `/api/v1/auth/login`
**Auth Required**: No
```typescript
// Request
{
  email: string;
  password: string;
}

// Response 200
{
  user: { id: string; email: string; full_name: string; role: string; tenant_id: string };
  session: { access_token: string; refresh_token: string; expires_at: number };
}

// Response 401
{ error: "invalid_credentials", message: "Invalid email or password" }
```

### POST `/api/v1/auth/refresh`
**Auth Required**: No (uses refresh_token)
```typescript
// Request
{ refresh_token: string }

// Response 200
{ access_token: string; refresh_token: string; expires_at: number }
```

### POST `/api/v1/auth/logout`
**Auth Required**: Yes
```typescript
// Response 200
{ success: true }
```

### POST `/api/v1/auth/forgot-password`
**Auth Required**: No
```typescript
// Request
{ email: string }

// Response 200
{ message: "Password reset email sent" }
```

### POST `/api/v1/auth/reset-password`
**Auth Required**: No
```typescript
// Request
{ token: string; new_password: string }

// Response 200
{ message: "Password updated" }
```

---

## 2. RFQs

### GET `/api/v1/rfqs`
**Auth Required**: Yes
**Roles**: All roles
```typescript
// Query Parameters
{
  status?: string;            // comma-separated: "new,extracting,costing"
  buyer_id?: string;          // UUID
  assigned_user_id?: string;  // UUID
  priority?: number;          // 1-5
  source_channel?: string;    // "email" | "portal" | "whatsapp" | "upload" | "api"
  search?: string;            // searches rfq_number, buyer name, product_type
  date_from?: string;         // ISO 8601 date
  date_to?: string;           // ISO 8601 date
  sort_by?: string;           // "created_at" | "deadline" | "priority" | "updated_at"
  sort_order?: string;        // "asc" | "desc" (default: "desc")
  page?: number;              // default: 1
  per_page?: number;          // default: 25, max: 100
}

// Response 200
{
  data: Array<{
    id: string;
    rfq_number: string;
    buyer: { id: string; name: string; tier: number } | null;
    assigned_user: { id: string; full_name: string; avatar_url: string | null } | null;
    status: string;
    priority: number;
    source_channel: string;
    product_type: string | null;
    total_quantity: number | null;
    estimated_value_usd: number | null;
    deadline: string | null;
    extraction_confidence_avg: number | null;
    fields_flagged_for_review: string[];
    attachment_count: number;
    created_at: string;
    updated_at: string;
  }>;
  pagination: {
    page: number;
    per_page: number;
    total: number;
    total_pages: number;
  };
}
```

### GET `/api/v1/rfqs/:id`
**Auth Required**: Yes
**Roles**: All roles
```typescript
// Response 200
{
  id: string;
  rfq_number: string;
  buyer: { id: string; name: string; code: string; tier: number; contact_email: string } | null;
  assigned_user: { id: string; full_name: string; email: string; avatar_url: string | null } | null;
  status: string;
  priority: number;
  source_channel: string;
  source_reference: string | null;
  source_email_from: string | null;
  source_email_subject: string | null;
  original_language: string;
  extracted_data: ExtractedSpecs | null;
  extraction_confidence_avg: number | null;
  fields_flagged_for_review: string[];
  manual_overrides: Record<string, { original: any; override: any; user_id: string; timestamp: string }>;
  product_type: string | null;
  total_quantity: number | null;
  estimated_value_usd: number | null;
  deadline: string | null;
  delivery_date: string | null;
  incoterm: string | null;
  attachments: Array<{
    id: string;
    file_name: string;
    file_type: string;
    file_size_bytes: number;
    storage_path: string;
    created_at: string;
  }>;
  quotes: Array<{
    id: string;
    quote_number: string;
    version: number;
    status: string;
    fob_per_unit: number;
    margin_percentage: number;
    created_at: string;
  }>;
  notes: string | null;
  won_lost_reason: string | null;
  created_at: string;
  updated_at: string;
  extracted_at: string | null;
  sent_at: string | null;
  responded_at: string | null;
  closed_at: string | null;
}
```

### POST `/api/v1/rfqs`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager, merchandiser
```typescript
// Request (multipart/form-data)
{
  buyer_id?: string;            // UUID
  source_channel: "upload";     // manual upload
  priority?: number;            // 1-5, default: 3
  deadline?: string;            // ISO 8601
  notes?: string;
  files: File[];                // one or more files (PDF, XLSX, DOCX, PNG, JPG, CSV)
}

// Response 201
{
  id: string;
  rfq_number: string;
  status: "extracting";
  attachments: Array<{ id: string; file_name: string; storage_path: string }>;
  message: "RFQ created. Extraction in progress.";
}
```

### PATCH `/api/v1/rfqs/:id`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager, merchandiser
```typescript
// Request
{
  buyer_id?: string;
  assigned_user_id?: string;
  priority?: number;
  deadline?: string;
  notes?: string;
  status?: string;              // restricted transitions enforced server-side
  won_lost_reason?: string;     // required when status = "won" or "lost"
}

// Response 200
{ ...updated RFQ object }
```

### POST `/api/v1/rfqs/:id/extract`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request
{
  skill?: "garment" | "footwear" | "home_textile";  // default: "garment"
  reprocess?: boolean;                                // default: false
}

// Response 202
{
  message: "Extraction started";
  job_id: string;
}
```

### PATCH `/api/v1/rfqs/:id/extracted-data`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request - Override specific extracted fields
{
  overrides: {
    [field_name: string]: any;  // e.g., { "fabric_gsm": 220, "total_quantity": 50000 }
  };
}

// Response 200
{
  extracted_data: ExtractedSpecs;  // updated with overrides applied
  manual_overrides: Record<string, { original: any; override: any; user_id: string; timestamp: string }>;
}
```

### POST `/api/v1/rfqs/:id/assign`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request
{ user_id: string }

// Response 200
{
  id: string;
  assigned_user: { id: string; full_name: string };
  message: "RFQ assigned";
}
```

### POST `/api/v1/rfqs/bulk-action`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request
{
  rfq_ids: string[];                        // max 50
  action: "assign" | "archive" | "set_priority";
  assign_to?: string;                       // required if action = "assign"
  priority?: number;                        // required if action = "set_priority"
}

// Response 200
{
  updated_count: number;
  message: string;
}
```

### DELETE `/api/v1/rfqs/:id`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Response 200
{ message: "RFQ archived", id: string }
// Note: soft delete - sets status to "archived"
```

---

## 3. Quotes

### GET `/api/v1/quotes`
**Auth Required**: Yes
**Roles**: All roles
```typescript
// Query Parameters
{
  status?: string;
  buyer_id?: string;
  rfq_id?: string;
  search?: string;
  date_from?: string;
  date_to?: string;
  sort_by?: string;     // "created_at" | "fob_total" | "margin_percentage"
  sort_order?: string;
  page?: number;
  per_page?: number;
}

// Response 200
{
  data: Array<{
    id: string;
    quote_number: string;
    rfq: { id: string; rfq_number: string };
    buyer: { id: string; name: string } | null;
    version: number;
    status: string;
    total_quantity: number;
    fob_per_unit: number;
    fob_total: number;
    margin_percentage: number;
    incoterm: string;
    currency: string;
    sent_at: string | null;
    created_at: string;
  }>;
  pagination: { page: number; per_page: number; total: number; total_pages: number };
}
```

### GET `/api/v1/quotes/:id`
**Auth Required**: Yes
**Roles**: All roles
```typescript
// Response 200
{
  id: string;
  quote_number: string;
  rfq: { id: string; rfq_number: string; product_type: string; extracted_data: ExtractedSpecs };
  buyer: { id: string; name: string; code: string } | null;
  version: number;
  status: string;
  cost_breakdown: CostBreakdown;
  currency: string;
  exchange_rate_to_usd: number;
  exchange_rate_locked: boolean;
  total_quantity: number;
  cost_per_unit: number;
  total_cost: number;
  margin_percentage: number;
  margin_per_unit: number;
  fob_per_unit: number;
  fob_total: number;
  incoterm: string;
  freight_per_unit: number;
  landed_price_per_unit: number | null;
  buyer_template: { id: string; name: string } | null;
  pdf_url: string | null;
  excel_url: string | null;
  validity_days: number;
  valid_until: string | null;
  sent_via: string | null;
  sent_at: string | null;
  opened_at: string | null;
  buyer_counter_offer: { price_per_unit: number; notes: string; received_at: string } | null;
  scenarios: Array<{ name: string; margin_percentage: number; fob_per_unit: number; fob_total: number; cost_breakdown: CostBreakdown }> | null;
  approved_by: { id: string; full_name: string } | null;
  approved_at: string | null;
  created_by: { id: string; full_name: string };
  created_at: string;
  updated_at: string;
  version_history: Array<{ version: number; fob_per_unit: number; margin_percentage: number; created_at: string }>;
}
```

### POST `/api/v1/quotes`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request
{
  rfq_id: string;                          // required
  margin_percentage?: number;              // default: tenant.default_margin_percentage
  incoterm?: string;                       // default: tenant.default_incoterm
  currency?: string;                       // default: tenant.base_currency
  lock_exchange_rate?: boolean;            // default: false
  buyer_template_id?: string;             // UUID
  validity_days?: number;                  // default: tenant.default_quote_validity_days
  notes?: string;
}

// Response 201
{
  id: string;
  quote_number: string;
  status: "draft";
  cost_breakdown: CostBreakdown;
  fob_per_unit: number;
  fob_total: number;
  margin_percentage: number;
  message: "Quote built successfully";
}

// Response 422
{
  error: "extraction_incomplete";
  message: "RFQ extraction not complete or has unresolved flagged fields";
  flagged_fields: string[];
}
```

### PATCH `/api/v1/quotes/:id`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request
{
  margin_percentage?: number;
  incoterm?: string;
  currency?: string;
  lock_exchange_rate?: boolean;
  buyer_template_id?: string;
  validity_days?: number;
  notes?: string;
  buyer_counter_offer?: { price_per_unit: number; notes: string };
}

// Response 200
{ ...updated quote object with recalculated costs }
```

### POST `/api/v1/quotes/:id/approve`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Response 200
{
  id: string;
  status: "approved";
  approved_by: { id: string; full_name: string };
  approved_at: string;
}
```

### POST `/api/v1/quotes/:id/send`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager, commercial_executive
```typescript
// Request
{
  channel: "email" | "portal";
  recipient_email?: string;     // required if channel = "email", defaults to buyer contact_email
  subject?: string;             // default: "Quote {quote_number} - {product_type}"
  message?: string;             // email body
  attach_pdf?: boolean;         // default: true
  attach_excel?: boolean;       // default: false
}

// Response 200
{
  id: string;
  status: "sent";
  sent_via: string;
  sent_at: string;
  message: "Quote sent successfully";
}
```

### POST `/api/v1/quotes/:id/revise`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request
{
  margin_percentage?: number;   // new margin for revision
  notes?: string;               // revision notes
}

// Response 201
{
  id: string;                   // new quote ID
  quote_number: string;         // same number, incremented version
  version: number;
  status: "draft";
  cost_breakdown: CostBreakdown;
  previous_version: { id: string; version: number; fob_per_unit: number };
}
```

### POST `/api/v1/quotes/:id/generate-pdf`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Response 200
{
  pdf_url: string;    // Supabase Storage signed URL, expires in 1 hour
}
```

### POST `/api/v1/quotes/:id/scenarios`
**Auth Required**: Yes (Growth+)
**Roles**: owner, admin, quotation_manager
```typescript
// Request
{
  scenarios: Array<{
    name: string;               // e.g., "Conservative", "Aggressive"
    margin_percentage: number;
    fabric_supplier_code?: string;  // optional: use different supplier
  }>;  // max 3 scenarios
}

// Response 200
{
  scenarios: Array<{
    name: string;
    margin_percentage: number;
    fob_per_unit: number;
    fob_total: number;
    cost_breakdown: CostBreakdown;
  }>;
}
```

---

## 4. Cost Library

### GET `/api/v1/cost-library`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Query Parameters
{
  category?: string;           // "fabric" | "cmt" | "trims" | "embellishment" | "washing" | "testing" | "packing" | "overhead" | "freight" | "commission" | "finance"
  search?: string;             // searches name, code, supplier_name
  product_type?: string;       // for CMT items
  is_active?: boolean;
  page?: number;
  per_page?: number;
}

// Response 200
{
  data: Array<CostLibraryItem>;
  pagination: { page: number; per_page: number; total: number; total_pages: number };
}
```

### GET `/api/v1/cost-library/:id`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Response 200
CostLibraryItem  // full object with all fields
```

### POST `/api/v1/cost-library`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request
{
  category: string;            // required
  name: string;                // required
  code?: string;
  unit: string;                // required
  cost: number;                // required
  currency?: string;           // default: tenant.base_currency
  supplier_name?: string;
  supplier_code?: string;
  fabric_composition?: string;
  fabric_gsm?: number;
  fabric_construction?: string;
  fabric_width_cm?: number;
  consumption_per_unit_meters?: number;
  consumption_per_unit_kg?: number;
  product_type?: string;
  complexity_tier?: string;
  sam_minutes?: number;
  moq_min?: number;
  moq_max?: number;
  wash_type?: string;
  origin_port?: string;
  destination_port?: string;
  effective_from?: string;     // default: today
  effective_to?: string;
  notes?: string;
}

// Response 201
{ ...created CostLibraryItem }
```

### PATCH `/api/v1/cost-library/:id`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request - any fields from POST body
// Response 200
{ ...updated CostLibraryItem }
```

### DELETE `/api/v1/cost-library/:id`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Response 200
{ message: "Cost item deactivated", id: string }
// Note: soft delete - sets is_active = false
```

### POST `/api/v1/cost-library/import`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request (multipart/form-data)
{
  file: File;                           // .xlsx or .csv
  category: string;                     // target category
  column_mapping: Record<string, string>; // { rfqx_field: excel_column_header }
}

// Response 200
{
  imported_count: number;
  skipped_count: number;
  errors: Array<{ row: number; message: string }>;
}
```

### POST `/api/v1/cost-library/bulk-update`
**Auth Required**: Yes (Growth+)
**Roles**: owner, admin
```typescript
// Request
{
  category: string;
  filter?: { supplier_code?: string; product_type?: string };
  update_type: "percentage" | "absolute";
  update_value: number;           // e.g., 3.0 for 3% increase, or 0.50 for $0.50 increase
  field: "cost";                  // only cost field supported for bulk update
}

// Response 200
{
  updated_count: number;
  message: string;
}
```

### POST `/api/v1/cost-library/snapshot`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Response 201
{
  snapshot_id: string;
  created_at: string;
  item_count: number;
}
```

---

## 5. Buyers

### GET `/api/v1/buyers`
**Auth Required**: Yes
**Roles**: All roles
```typescript
// Query Parameters
{ search?: string; tier?: number; is_active?: boolean; page?: number; per_page?: number }

// Response 200
{
  data: Array<{
    id: string;
    name: string;
    code: string | null;
    country: string | null;
    tier: number;
    contact_name: string | null;
    contact_email: string | null;
    preferred_incoterm: string;
    portal_type: string | null;
    rfq_count: number;
    win_rate: number;         // calculated: won / (won + lost) * 100
    is_active: boolean;
  }>;
  pagination: { page: number; per_page: number; total: number; total_pages: number };
}
```

### GET `/api/v1/buyers/:id`
**Auth Required**: Yes
**Roles**: All roles
```typescript
// Response 200
{
  id: string;
  name: string;
  code: string | null;
  contact_name: string | null;
  contact_email: string | null;
  contact_phone: string | null;
  country: string | null;
  tier: number;
  preferred_incoterm: string;
  preferred_currency: string;
  payment_terms: string | null;
  quote_template: { id: string; name: string } | null;
  portal_type: string | null;
  commission_percentage: number;
  notes: string | null;
  is_active: boolean;
  stats: {
    total_rfqs: number;
    total_quotes: number;
    won_quotes: number;
    lost_quotes: number;
    win_rate: number;
    avg_order_value_usd: number;
    avg_margin_percentage: number;
    last_rfq_date: string | null;
  };
  created_at: string;
  updated_at: string;
}
```

### POST `/api/v1/buyers`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager, merchandiser
```typescript
// Request
{
  name: string;                   // required
  code?: string;
  contact_name?: string;
  contact_email?: string;
  contact_phone?: string;
  country?: string;
  tier?: number;                  // 1-5, default: 3
  preferred_incoterm?: string;    // default: "FOB"
  preferred_currency?: string;    // default: "USD"
  payment_terms?: string;
  quote_template_id?: string;
  portal_type?: string;
  commission_percentage?: number; // default: 0.00
  notes?: string;
}

// Response 201
{ ...created Buyer }
```

### PATCH `/api/v1/buyers/:id`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request - any fields from POST body
// Response 200
{ ...updated Buyer }
```

---

## 6. Users & Team

### GET `/api/v1/users`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Response 200
{
  data: Array<{
    id: string;
    email: string;
    full_name: string;
    role: string;
    avatar_url: string | null;
    is_active: boolean;
    mfa_enabled: boolean;
    last_login_at: string | null;
    open_rfq_count: number;
    open_quote_count: number;
    created_at: string;
  }>;
}
```

### POST `/api/v1/users/invite`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Request
{
  email: string;               // required
  full_name: string;           // required
  role: string;                // required: "admin" | "quotation_manager" | "merchandiser" | "commercial_executive" | "viewer"
}

// Response 201
{
  id: string;
  email: string;
  role: string;
  message: "Invitation sent";
}

// Response 403 (plan limit)
{ error: "user_limit_reached", message: "Your plan allows 2 users. Upgrade to add more." }
```

### PATCH `/api/v1/users/:id`
**Auth Required**: Yes
**Roles**: owner, admin (owner for role changes)
```typescript
// Request
{
  full_name?: string;
  role?: string;
  is_active?: boolean;
  notification_preferences?: { email: boolean; in_app: boolean; quinn_alerts: boolean };
}

// Response 200
{ ...updated User }
```

### DELETE `/api/v1/users/:id`
**Auth Required**: Yes
**Roles**: owner
```typescript
// Response 200
{ message: "User deactivated", id: string }
// Note: soft delete - sets is_active = false
```

---

## 7. Inbox Connections

### GET `/api/v1/inbox-connections`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Response 200
{
  data: Array<{
    id: string;
    connection_type: string;
    email_address: string | null;
    display_name: string | null;
    is_active: boolean;
    last_sync_at: string | null;
    sync_error: string | null;
    created_at: string;
  }>;
}
```

### POST `/api/v1/inbox-connections/gmail`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Request
{ oauth_code: string }    // Google OAuth authorization code

// Response 201
{
  id: string;
  connection_type: "gmail";
  email_address: string;
  message: "Gmail connected successfully";
}
```

### POST `/api/v1/inbox-connections/outlook`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Request
{ oauth_code: string }    // Microsoft OAuth authorization code

// Response 201
{
  id: string;
  connection_type: "outlook";
  email_address: string;
  message: "Outlook connected successfully";
}
```

### POST `/api/v1/inbox-connections/imap`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Request
{
  email_address: string;
  imap_host: string;
  imap_port: number;      // default: 993
  username: string;
  password: string;
  use_ssl: boolean;        // default: true
}

// Response 201
{
  id: string;
  connection_type: "imap";
  email_address: string;
  message: "IMAP connection established";
}
```

### DELETE `/api/v1/inbox-connections/:id`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Response 200
{ message: "Connection removed" }
```

---

## 8. Notifications

### GET `/api/v1/notifications`
**Auth Required**: Yes
**Roles**: All roles
```typescript
// Query Parameters
{ is_read?: boolean; type?: string; page?: number; per_page?: number }

// Response 200
{
  data: Array<{
    id: string;
    type: string;
    title: string;
    body: string | null;
    link: string | null;
    rfq_id: string | null;
    quote_id: string | null;
    is_read: boolean;
    created_at: string;
  }>;
  unread_count: number;
  pagination: { page: number; per_page: number; total: number; total_pages: number };
}
```

### PATCH `/api/v1/notifications/:id/read`
**Auth Required**: Yes
```typescript
// Response 200
{ id: string; is_read: true }
```

### POST `/api/v1/notifications/mark-all-read`
**Auth Required**: Yes
```typescript
// Response 200
{ updated_count: number }
```

---

## 9. QUINN Copilot

### POST `/api/v1/quinn/chat`
**Auth Required**: Yes
**Roles**: All roles (tool execution respects role permissions)
```typescript
// Request
{
  message: string;                     // user's chat message
  conversation_id?: string;            // existing conversation ID, or omit for new
  context: {
    page: string;                      // current page route, e.g., "/inbox/abc-123"
    rfq_id?: string;                   // active RFQ if on RFQ page
    quote_id?: string;                 // active quote if on quote page
  };
}

// Response 200 (streaming: text/event-stream)
// SSE events:
data: { type: "text", content: "Here are your urgent RFQs..." }
data: { type: "tool_call", tool: "list_rfqs", args: { status: "new", priority: 1 } }
data: { type: "tool_result", tool: "list_rfqs", result: [...] }
data: { type: "action_button", label: "Build Quote", action: "build_quote", params: { rfq_id: "..." } }
data: { type: "done", conversation_id: "...", message_count: 5 }
```

### GET `/api/v1/quinn/conversations`
**Auth Required**: Yes
```typescript
// Response 200
{
  data: Array<{
    id: string;
    message_count: number;
    last_message_preview: string;
    context_page: string | null;
    updated_at: string;
  }>;
}
```

### GET `/api/v1/quinn/conversations/:id`
**Auth Required**: Yes
```typescript
// Response 200
{
  id: string;
  messages: Array<{
    role: "user" | "assistant";
    content: string;
    timestamp: string;
    tool_calls?: Array<{ tool: string; args: any; result: any }>;
    action_buttons?: Array<{ label: string; action: string; params: any }>;
  }>;
  message_count: number;
  context_rfq_id: string | null;
  context_quote_id: string | null;
}
```

---

## 10. Analytics

### GET `/api/v1/analytics/overview`
**Auth Required**: Yes
**Roles**: All roles
```typescript
// Query Parameters
{ date_from?: string; date_to?: string }  // default: last 30 days

// Response 200
{
  win_rate: { current: number; previous: number; change: number };  // percentages
  avg_response_time_hours: { current: number; previous: number; change: number };
  rfq_volume: { current: number; previous: number; change: number };
  pipeline_value_usd: number;
  missed_rfqs: number;
  revenue_forecast_usd: number;    // pipeline_value * win_rate
}
```

### GET `/api/v1/analytics/win-rate`
**Auth Required**: Yes
```typescript
// Query Parameters
{ date_from?: string; date_to?: string; group_by?: "buyer" | "product" | "month" }

// Response 200
{
  overall: number;
  breakdown: Array<{
    label: string;       // buyer name, product type, or month
    won: number;
    lost: number;
    expired: number;
    win_rate: number;
  }>;
  trend: Array<{ month: string; win_rate: number }>;  // last 12 months
}
```

### GET `/api/v1/analytics/response-time`
**Auth Required**: Yes
```typescript
// Query Parameters
{ date_from?: string; date_to?: string }

// Response 200
{
  avg_hours: number;
  median_hours: number;
  trend: Array<{ month: string; avg_hours: number }>;
  by_user: Array<{ user_id: string; full_name: string; avg_hours: number; quote_count: number }>;
}
```

### GET `/api/v1/analytics/pipeline`
**Auth Required**: Yes (Growth+)
```typescript
// Response 200
{
  total_value_usd: number;
  by_status: Array<{ status: string; count: number; value_usd: number }>;
  by_buyer: Array<{ buyer_id: string; buyer_name: string; count: number; value_usd: number }>;
  expected_close: Array<{ month: string; value_usd: number; weighted_value_usd: number }>;
}
```

### GET `/api/v1/analytics/buyer-scorecard`
**Auth Required**: Yes (Growth+)
```typescript
// Query Parameters
{ buyer_id?: string; date_from?: string; date_to?: string }

// Response 200
{
  buyers: Array<{
    buyer_id: string;
    buyer_name: string;
    rfq_count: number;
    quote_count: number;
    won_count: number;
    win_rate: number;
    avg_order_value_usd: number;
    avg_margin_percentage: number;
    avg_response_time_hours: number;
    volume_trend: "up" | "down" | "stable";
    volume_change_percentage: number;
  }>;
}
```

### GET `/api/v1/analytics/cost-trends`
**Auth Required**: Yes (Growth+)
```typescript
// Query Parameters
{ category?: string; months?: number }  // default: 12 months

// Response 200
{
  trends: Array<{
    category: string;
    data_points: Array<{ month: string; avg_cost: number; min_cost: number; max_cost: number }>;
    change_percentage: number;  // vs first month
  }>;
  margin_alert: { triggered: boolean; current_avg_margin: number; threshold: number } | null;
}
```

### GET `/api/v1/analytics/export`
**Auth Required**: Yes
```typescript
// Query Parameters
{ report: "overview" | "win_rate" | "pipeline" | "buyer_scorecard" | "cost_trends"; format: "xlsx" | "csv"; date_from?: string; date_to?: string }

// Response 200 (application/octet-stream)
// File download
```

---

## 11. Exchange Rates

### GET `/api/v1/exchange-rates`
**Auth Required**: Yes
```typescript
// Query Parameters
{ base?: string; target?: string }  // default base: "USD"

// Response 200
{
  rates: Array<{
    base_currency: string;
    target_currency: string;
    rate: number;
    source: string;
    fetched_at: string;
  }>;
}
```

### POST `/api/v1/exchange-rates`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Request
{
  base_currency: string;     // e.g., "USD"
  target_currency: string;   // e.g., "BDT"
  rate: number;              // e.g., 110.50
}

// Response 201
{ ...created ExchangeRate }
```

---

## 12. Tenant Settings

### GET `/api/v1/tenant`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Response 200
{
  id: string;
  name: string;
  slug: string;
  plan: string;
  base_currency: string;
  max_rfqs_per_month: number;
  max_users: number;
  logo_url: string | null;
  letterhead_url: string | null;
  address: string | null;
  phone: string | null;
  website: string | null;
  default_margin_percentage: number;
  default_incoterm: string;
  default_quote_validity_days: number;
  overhead_percentage: number;
  commission_percentage: number;
  finance_cost_percentage: number;
  subscription_status: string;
  current_month_rfq_count: number;
  current_user_count: number;
}
```

### PATCH `/api/v1/tenant`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Request
{
  name?: string;
  address?: string;
  phone?: string;
  website?: string;
  base_currency?: string;
  default_margin_percentage?: number;
  default_incoterm?: string;
  default_quote_validity_days?: number;
  overhead_percentage?: number;
  commission_percentage?: number;
  finance_cost_percentage?: number;
}

// Response 200
{ ...updated Tenant }
```

### POST `/api/v1/tenant/logo`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Request (multipart/form-data)
{ file: File }  // PNG, JPG, SVG, max 2MB

// Response 200
{ logo_url: string }
```

### POST `/api/v1/tenant/letterhead`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Request (multipart/form-data)
{ file: File }  // PDF, PNG, max 5MB

// Response 200
{ letterhead_url: string }
```

---

## 13. Audit Log

### GET `/api/v1/audit-log`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Query Parameters
{
  user_id?: string;
  action?: string;
  entity_type?: string;
  entity_id?: string;
  date_from?: string;
  date_to?: string;
  page?: number;
  per_page?: number;    // default: 50
}

// Response 200
{
  data: Array<{
    id: string;
    user: { id: string; full_name: string } | null;
    action: string;
    entity_type: string;
    entity_id: string;
    changes: Record<string, { old: any; new: any }> | null;
    ip_address: string | null;
    created_at: string;
  }>;
  pagination: { page: number; per_page: number; total: number; total_pages: number };
}
```

### GET `/api/v1/audit-log/export`
**Auth Required**: Yes
**Roles**: owner, admin
```typescript
// Query Parameters - same as GET /audit-log filters
// Response 200 (application/octet-stream) - CSV download
```

---

## 14. File Uploads

### POST `/api/v1/upload`
**Auth Required**: Yes
```typescript
// Request (multipart/form-data)
{
  file: File;
  purpose: "rfq_attachment" | "cost_import" | "logo" | "letterhead";
  rfq_id?: string;   // required if purpose = "rfq_attachment"
}

// Response 201
{
  id: string;
  file_name: string;
  file_type: string;
  file_size_bytes: number;
  storage_path: string;
  signed_url: string;      // expires in 1 hour
}

// Response 413
{ error: "file_too_large", message: "File exceeds 25MB limit" }

// Response 415
{ error: "unsupported_type", message: "File type not supported. Allowed: pdf, xlsx, xls, docx, doc, csv, png, jpg, jpeg" }
```

### GET `/api/v1/upload/:id/signed-url`
**Auth Required**: Yes
```typescript
// Response 200
{ signed_url: string; expires_at: string }
```

---

## 15. Webhooks (Incoming)

### POST `/api/v1/webhooks/whatsapp`
**Auth Required**: WhatsApp verify_token validation
```typescript
// Request (from WhatsApp Business API)
{
  object: "whatsapp_business_account";
  entry: Array<{
    id: string;
    changes: Array<{
      value: {
        messages: Array<{
          from: string;
          type: string;
          text?: { body: string };
          document?: { id: string; filename: string };
        }>;
      };
    }>;
  }>;
}

// Response 200
{ status: "ok" }
```

### POST `/api/v1/webhooks/email`
**Auth Required**: Nylas webhook signature validation
```typescript
// Request (from Nylas)
{
  deltas: Array<{
    type: "message.created";
    object_data: { id: string; account_id: string };
  }>;
}

// Response 200
{ status: "ok" }
```

---

## 16. Auto-Routing Rules

### GET `/api/v1/routing-rules`
**Auth Required**: Yes (Growth+)
**Roles**: owner, admin
```typescript
// Response 200
{
  data: Array<{
    id: string;
    rule_name: string;
    condition_type: string;
    condition_value: string;
    assign_to_user: { id: string; full_name: string };
    is_active: boolean;
    priority_order: number;
  }>;
}
```

### POST `/api/v1/routing-rules`
**Auth Required**: Yes (Growth+)
**Roles**: owner, admin
```typescript
// Request
{
  rule_name: string;
  condition_type: "buyer" | "product_type" | "priority" | "channel";
  condition_value: string;
  assign_to_user_id: string;
  priority_order?: number;     // default: 0
}

// Response 201
{ ...created RoutingRule }
```

### PATCH `/api/v1/routing-rules/:id`
**Auth Required**: Yes (Growth+)
**Roles**: owner, admin
```typescript
// Request - any fields from POST
// Response 200
{ ...updated RoutingRule }
```

### DELETE `/api/v1/routing-rules/:id`
**Auth Required**: Yes (Growth+)
**Roles**: owner, admin
```typescript
// Response 200
{ message: "Rule deleted" }
```

---

## 17. Follow-Up Templates

### GET `/api/v1/follow-up-templates`
**Auth Required**: Yes
```typescript
// Response 200
{
  data: Array<{
    id: string;
    name: string;
    subject_template: string;
    body_template: string;
    buyer: { id: string; name: string } | null;
    is_default: boolean;
  }>;
}
```

### POST `/api/v1/follow-up-templates`
**Auth Required**: Yes
**Roles**: owner, admin, quotation_manager
```typescript
// Request
{
  name: string;
  subject_template: string;     // supports: {{buyer_name}}, {{product}}, {{quote_number}}, {{factory_name}}
  body_template: string;        // supports same variables
  buyer_id?: string;
  is_default?: boolean;
}

// Response 201
{ ...created FollowUpTemplate }
```

---

## Error Response Format

All error responses follow this shape:

```typescript
{
  error: string;       // machine-readable error code
  message: string;     // human-readable message
  details?: any;       // additional context (validation errors, etc.)
}
```

**Standard HTTP Status Codes Used**:
| Code | Usage |
|---|---|
| 200 | Success |
| 201 | Created |
| 202 | Accepted (async job started) |
| 400 | Bad request / validation error |
| 401 | Not authenticated |
| 403 | Forbidden (wrong role or plan) |
| 404 | Resource not found |
| 409 | Conflict (duplicate) |
| 413 | Payload too large |
| 415 | Unsupported media type |
| 422 | Unprocessable entity (business rule violation) |
| 429 | Rate limited |
| 500 | Internal server error |

**Rate Limits**:
| Endpoint Pattern | Limit |
|---|---|
| `/api/v1/auth/*` | 10 requests/minute per IP |
| `/api/v1/quinn/chat` | 30 requests/minute per user |
| All other endpoints | 100 requests/minute per user |
| Scale tier API access | 1000 requests/minute per tenant |

---

*This document is confidential and intended for internal use only.*
*fabricXai is a product of SocioFi Technology Ltd.*
