# RFQX UI Specification
## Version 1.0 | March 2026

---

## 1. Navigation Structure

### 1.1 Main Sidebar Navigation (Left)

```
[Logo: RFQX]
---------------------------
Dashboard          /dashboard
RFQ Inbox          /inbox
Quote Builder      /quotes
Cost Library       /cost-library
Buyers             /buyers
Analytics          /analytics
---------------------------
Settings           /settings
Help               /help
---------------------------
[User Avatar + Name]
[Logout]
```

### 1.2 QUINN Copilot Panel (Right Sidebar)

- Collapsible chat panel, available on every page
- Toggle button: floating action button bottom-right corner
- Width: 400px on desktop
- Full-screen on mobile when opened

### 1.3 Top Bar

```
[Breadcrumb: Dashboard > RFQ Inbox > RFQ #1234]  [Search (Cmd+K)]  [Notifications Bell (count badge)]  [QUINN Toggle]
```

---

## 2. Pages & Screens

### 2.1 Login / Auth Pages

#### `/login`
- **Components**: Card with email input, password input, "Sign in" button, "Sign in with Google" button, "Forgot password" link, "Sign up" link
- **Data**: None (pre-auth)
- **Interactions**: Email+password login, Google OAuth, redirect to `/dashboard` on success
- **Layout**: Centered card on gradient background, RFQX logo top

#### `/signup`
- **Components**: Card with full name input, email input, password input, factory name input, country select (Bangladesh pre-selected), "Create account" button, "Sign in with Google" button
- **Data**: None (pre-auth)
- **Interactions**: Creates tenant + owner user, redirects to `/onboarding`

#### `/forgot-password`
- **Components**: Email input, "Send reset link" button
- **Data**: None
- **Interactions**: Sends password reset email via Supabase Auth

#### `/onboarding`
- **Route**: `/onboarding` (3-step wizard, first-time only)
- **Step 1 - Factory Profile**: Factory name, address, phone, website, logo upload, base currency select (BDT, USD, EUR, GBP, VND, INR), default incoterm select
- **Step 2 - Cost Library Setup**: Option to upload Excel cost sheet OR manually enter initial costs. Category tabs: Fabric, CMT, Trims, Washing, Packing, Overhead, Freight. Each has an "Add item" form.
- **Step 3 - Connect Inbox**: "Connect Gmail" button (OAuth), "Connect Outlook" button (OAuth), "Skip for now" option
- **Components**: shadcn/ui Stepper, FileUpload, Select, Input, Button
- **Data**: Creates tenant config, cost_library_items, inbox_connections

---

### 2.2 Dashboard

#### `/dashboard`
- **Layout**: Grid of metric cards (top row) + two-column layout below
- **Components**:

**Top Row - KPI Cards (4 cards)**:
| Card | Data | Visual |
|---|---|---|
| Open RFQs | Count of rfqs where status NOT IN (won, lost, expired, archived) | Number + trend arrow vs last month |
| Avg Response Time | Mean hours from rfq.created_at to quote.sent_at (last 30 days) | Number (hours) + sparkline |
| Win Rate | (won quotes / total closed quotes) * 100 for last 30 days | Percentage + donut chart |
| Pipeline Value | SUM(quotes.fob_total) where quote.status IN (draft, pending_review, approved, sent) | USD amount + bar |

**Left Column**:
- **Urgent RFQs Widget**: Table of top 5 RFQs sorted by deadline ASC where status NOT IN (won, lost, expired). Columns: RFQ#, Buyer, Product, Deadline (color-coded: green >24h, amber <24h, red overdue), Priority badge. Click row -> `/inbox/{rfq_id}`
- **Recent Activity Feed**: Last 10 audit_log entries for tenant. Shows: user avatar, action text, timestamp, entity link.

**Right Column**:
- **Pipeline by Status**: Horizontal stacked bar chart. Segments: New, Extracting, Costing, Review, Sent, Followed Up. Shows count per segment.
- **Quote Value by Buyer**: Bar chart of top 5 buyers by total open quote FOB value.

**Interactions**: Click any card/widget item to navigate to detail. Date range picker (default: last 30 days) in top-right.

---

### 2.3 RFQ Inbox

#### `/inbox` (List View - Default)
- **Layout**: Filter bar on top, sortable table below
- **Filter Bar Components**:
  - Status multi-select (New, Extracting, Costing, Review, Sent, Followed Up, Won, Lost, Expired)
  - Buyer select (from buyers table)
  - Priority select (1-5)
  - Assigned To select (from users)
  - Date range picker
  - Search input (searches rfq_number, buyer name, product_type)
  - "Upload RFQ" button (primary action)
  - View toggle: List | Board

**Table Columns**:
| Column | Data | Sortable |
|---|---|---|
| Checkbox | Bulk select | No |
| RFQ # | rfq_number | Yes |
| Buyer | buyer.name | Yes |
| Product | product_type | Yes |
| Quantity | total_quantity (formatted with commas) | Yes |
| Priority | Badge 1-5 (color: 1=red, 2=orange, 3=yellow, 4=blue, 5=gray) | Yes |
| Status | Badge with status color | Yes |
| Source | Icon (email/portal/whatsapp/upload) | Yes |
| Deadline | Relative time + color coding | Yes |
| Assigned | User avatar | Yes |
| Created | Relative timestamp | Yes |

**Bulk Actions Bar** (appears when rows selected): Assign To, Change Priority, Archive, Export CSV

**Interactions**: Click row -> `/inbox/{rfq_id}`, Upload button opens file upload dialog, Drag-and-drop file upload supported

#### `/inbox?view=board` (Board View)
- **Layout**: Kanban columns for each status
- **Columns**: New | Extracting | Costing | Review | Sent | Followed Up
- **Cards**: Show buyer name, product_type, quantity, priority badge, deadline, assigned avatar
- **Interactions**: Drag card between columns to change status. Click card -> `/inbox/{rfq_id}`

#### `/inbox/{rfq_id}` (RFQ Detail)
- **Layout**: Two-column. Left: RFQ details. Right: Activity + QUINN context panel.
- **Top Bar**: Back to inbox, RFQ number, status badge, priority badge, "Build Quote" button (primary), "Assign" dropdown, "Archive" button

**Left Column Tabs**:

**Tab 1: Extracted Specs**
- Table of all extracted fields from `extracted_data` JSON
- Each row: Field Name | Extracted Value | Confidence % (progress bar, red <85%, green >=85%) | Edit icon
- Flagged fields (confidence < 85%) highlighted with amber background
- "Re-extract" button
- Edit inline: click edit icon, modify value, saves to `manual_overrides`

**Tab 2: Attachments**
- Grid of attachment cards. Each shows: filename, file type icon, file size, "View" button, "Download" button
- PDF/image preview in modal on click
- "Upload More" button
- OCR text view toggle per attachment

**Tab 3: Cost Sheet** (visible after extraction)
- Full cost breakdown table if quote exists. Links to quote detail.
- "Build Quote" button if no quote yet

**Tab 4: Communication**
- Timeline of all emails sent/received for this RFQ
- Negotiation notes (editable text area)
- Counter-offer log

**Right Column**:
- **Status Timeline**: Vertical timeline of status changes with user, timestamp, duration at each stage
- **Internal Notes**: Thread of notes from team members. Add note text area with @mention support.
- **Activity Feed**: Recent actions on this RFQ from audit_log

---

### 2.4 Quote Builder

#### `/quotes` (Quote List)
- **Layout**: Filter bar + sortable table
- **Filter Bar**: Status select, Buyer select, Date range, Search (quote_number, buyer name)

**Table Columns**:
| Column | Data |
|---|---|
| Quote # | quote_number |
| RFQ # | linked rfq.rfq_number (clickable) |
| Buyer | buyer.name |
| Product | rfq.product_type |
| Qty | total_quantity |
| FOB/Unit | fob_per_unit (formatted to 4 decimals) |
| FOB Total | fob_total (formatted with commas) |
| Margin | margin_percentage + "%" |
| Status | Badge |
| Version | "v" + version |
| Sent | sent_at or "-" |
| Created | created_at relative |

**Interactions**: Click row -> `/quotes/{quote_id}`

#### `/quotes/{quote_id}` (Quote Detail / Builder)
- **Layout**: Three sections stacked vertically

**Section 1: Header**
- Quote number, version badge, status badge
- Buyer name (linked to buyer profile)
- RFQ reference (linked to RFQ detail)
- Action buttons: "Approve" (if pending_review), "Send to Buyer" (if approved), "Revise" (creates new version), "Download PDF", "Download Excel", "Compare Scenarios"

**Section 2: Cost Breakdown Table**
| Row | Per Unit (USD) | Total (USD) | Notes |
|---|---|---|---|
| Fabric | cost_breakdown.fabric.per_unit | .total | composition, GSM, supplier |
| CMT | .cmt.per_unit | .total | SAM, rate card ref |
| Trims | .trims.per_unit | .total | itemized expandable |
| Washing | .washing.per_unit | .total | wash type |
| Embellishment | .embellishment.per_unit | .total | type, details |
| Packing | .packing.per_unit | .total | |
| Overhead | .overhead.per_unit | .total | percentage shown |
| Testing | .testing.per_unit | .total | |
| Commission | .commission.per_unit | .total | percentage shown |
| Finance Cost | .finance_cost.per_unit | .total | percentage shown |
| Freight | .freight.per_unit | .total | port shown |
| **Cost Price** | **cost_per_unit** | **total_cost** | |
| **Margin** | **margin_per_unit** | - | editable slider + input |
| **FOB Price** | **fob_per_unit** | **fob_total** | bold, highlighted |

- Margin control: Slider (0-50%) or direct input. FOB updates in real-time.
- Incoterm toggle: FOB | CIF | EXW | DDP radio buttons. Freight row adjusts.
- Currency toggle: Show in USD | BDT | EUR | buyer currency. Dual display option.

**Section 3: Quote Details**
- Quantity: displayed, linked to RFQ
- Size breakdown table: Size | Qty | % of Total
- Delivery terms: Date, Port, Incoterm
- Payment terms: text field
- Validity: date picker (default: created_at + validity_days)
- Notes to buyer: rich text editor

**Section 4: Scenario Comparison** (expandable panel)
- Side-by-side table of up to 3 scenarios
- Each scenario: name input, margin slider, FOB result, total value
- "Add Scenario" button
- Highlight winning scenario (lowest FOB that meets margin target)

#### `/quotes/{quote_id}/preview`
- **Layout**: Full-page PDF preview of the generated quote
- Uses buyer template formatting
- Factory letterhead applied
- "Download" and "Send" buttons in toolbar

---

### 2.5 Cost Library

#### `/cost-library`
- **Layout**: Category tabs + item table per category
- **Category Tabs**: Fabric | CMT | Trims | Embellishment | Washing | Testing | Packing | Overhead | Freight | Commission | Finance
- **Top Actions**: "Add Item" button, "Import from Excel" button, "Bulk Update" button (Growth+), "Export" button

**Fabric Tab Table Columns**:
| Column | Data |
|---|---|
| Name | name |
| Code | code |
| Composition | fabric_composition |
| GSM | fabric_gsm |
| Construction | fabric_construction |
| Width (cm) | fabric_width_cm |
| Cost/Meter | cost (formatted) |
| Currency | currency |
| Supplier | supplier_name |
| Consumption/Unit | consumption_per_unit_meters |
| Effective From | effective_from |
| Active | toggle |
| Actions | Edit, Duplicate, Deactivate |

**CMT Tab Table Columns**:
| Column | Data |
|---|---|
| Product Type | product_type |
| Complexity | complexity_tier badge |
| SAM (min) | sam_minutes |
| Rate/Unit | cost |
| MOQ Min | moq_min |
| MOQ Max | moq_max |
| Effective From | effective_from |
| Active | toggle |
| Actions | Edit, Duplicate, Deactivate |

(Other tabs follow similar pattern with category-specific columns)

**Add/Edit Item Dialog** (shadcn/ui Sheet or Dialog):
- Dynamic form based on selected category
- All fields from `cost_library_items` relevant to that category
- Save + Save & Add Another buttons

**Import from Excel Dialog**:
- File upload zone (drag-and-drop)
- Column mapping interface: RFQX field <-> Excel column header
- Preview of first 5 rows
- "Import" button with count of items to create

---

### 2.6 Buyers

#### `/buyers`
- **Layout**: Searchable table of all buyers
- **Columns**: Name, Code, Country, Tier (star rating), Contact, RFQs (count), Win Rate (%), Preferred Incoterm, Portal, Active toggle
- **Actions**: "Add Buyer" button, Click row -> `/buyers/{buyer_id}`

#### `/buyers/{buyer_id}`
- **Layout**: Profile header + tabbed content

**Profile Header**: Name, code, tier (editable stars), country, contact info, portal badge, "Edit" button

**Tab 1: Overview**
- Stats cards: Total RFQs, Win Rate, Avg Order Value, Avg Margin, Last RFQ Date
- RFQ volume trend chart (last 12 months)

**Tab 2: RFQ History**
- Table of all RFQs from this buyer. Columns: RFQ#, Product, Qty, Status, FOB/Unit, Date. Click -> RFQ detail.

**Tab 3: Quote History**
- Table of all quotes for this buyer. Columns: Quote#, Product, Qty, FOB/Unit, Margin%, Status, Date.

**Tab 4: Settings**
- Preferred template select
- Default commission %
- Portal credentials (encrypted, show as masked)
- Payment terms default
- Notes

---

### 2.7 Analytics

#### `/analytics`
- **Layout**: Date range picker (top), tab navigation for sections

**Tab 1: Performance**
- Win Rate card (overall + trend line chart, 12 months)
- Response Time card (avg hours + trend)
- RFQ Volume card (monthly bar chart)
- Missed RFQs card (not quoted or quoted late)
- Pipeline Value card (stacked by status)
- Revenue Forecast card (based on pipeline * historical win rate)

**Tab 2: Buyer Intelligence**
- Buyer scorecard table: Name, RFQs, Quotes, Win Rate, Avg FOB, Avg Margin, Trend (up/down arrow)
- Click buyer row -> expanded view with volume trend chart
- Price sensitivity scatter plot (margin % vs win rate) for Scale tier

**Tab 3: Cost Intelligence**
- Input cost trend line chart: Fabric, CMT, Trims over 12 months
- Margin erosion alert panel (shows if avg margin dropped >2% month-over-month)
- Cost vs Budget comparison table (Scale tier)

**Tab 4: Export**
- "Export to Excel" and "Export to CSV" buttons for each analytics view
- Scheduled report setup: frequency (weekly/monthly), recipient email, metrics to include (Growth+)

---

### 2.8 Settings

#### `/settings` (tabbed layout)

**Tab 1: Factory Profile** (`/settings/profile`)
- Factory name, address, phone, website, logo upload, letterhead upload
- Base currency select
- Default margin %, default incoterm, default quote validity days
- Overhead %, commission %, finance cost %

**Tab 2: Team** (`/settings/team`)
- User table: Name, Email, Role, Status (active/invited), Last Login, Actions (Edit Role, Deactivate, Remove)
- "Invite User" button -> dialog: email input, role select, send invite
- MFA status per user (required for Owner/Admin)

**Tab 3: Integrations** (`/settings/integrations`)
- Connected inboxes list: email address, type icon, last sync time, status (active/error), "Disconnect" button
- "Connect Gmail" button, "Connect Outlook" button, "Add IMAP" button
- Portal connections section (Growth+): Primark, H&M, ASOS portal cards with connect/disconnect
- WhatsApp connection (Growth+)

**Tab 4: Templates** (`/settings/templates`)
- Quote template list: Name, Buyer, Type (PDF/Excel), Default toggle, "Edit" button, "Preview" button
- "Create Template" button -> template editor with drag-and-drop sections

**Tab 5: Routing Rules** (`/settings/routing`) (Growth+)
- Rules table: Name, Condition (buyer/product/priority/channel), Assign To, Active toggle, priority order
- "Add Rule" button -> form: condition type select, condition value, assign to user select

**Tab 6: Notifications** (`/settings/notifications`)
- Per-user notification preferences matrix:
  - Rows: New RFQ, Deadline Approaching, Quote Opened, Buyer Response, Assignment, QUINN Alert
  - Columns: Email, In-App
  - Toggle per cell

**Tab 7: Billing** (`/settings/billing`) (Owner only)
- Current plan card: plan name, price, RFQ usage (X / Y this month), user count
- "Upgrade Plan" button -> plan comparison modal
- Payment method: Stripe card on file, "Update" button
- Invoice history table: Date, Amount, Status, "Download" link

**Tab 8: Audit Log** (`/settings/audit-log`)
- Searchable, filterable table of all audit_log entries
- Columns: Timestamp, User, Action, Entity, Changes (expandable JSON), IP
- Filters: User, Action type, Entity type, Date range
- "Export" button

---

### 2.9 QUINN Copilot Panel

**Available on every page as right sidebar or floating panel**

- **Header**: "QUINN" label, minimize button, full-screen toggle
- **Chat Messages Area**: Scrollable message list
  - User messages: right-aligned, blue background
  - QUINN messages: left-aligned, gray background, may include:
    - Markdown text (tables, bold, links)
    - Action buttons (e.g., "Build Quote", "Assign to Sarah", "Send Follow-Up")
    - Deep-link cards (click to navigate to RFQ/quote)
    - Data tables (inline)
  - Typing indicator when QUINN is processing
- **Input Area**: Text input with send button, "/" command autocomplete
- **Quick Actions**: Expandable list of "/" commands:
  - `/quote` - Start building a quote
  - `/status` - Check pipeline status
  - `/report` - Generate analytics summary
  - `/assign` - Assign current RFQ
  - `/follow-up` - Send follow-up
  - `/help` - Show available commands

**Context Injection**: QUINN automatically receives:
- Current page route
- Active RFQ ID and extracted data (if on RFQ detail page)
- Active Quote ID and cost breakdown (if on quote page)
- User role and permissions

---

### 2.10 Notification Center

**Triggered by bell icon in top bar**

- **Dropdown Panel**:
  - List of last 20 notifications sorted by created_at DESC
  - Each notification: icon (by type), title, body preview, relative timestamp, read/unread indicator
  - Click notification -> navigate to linked entity (RFQ or quote)
  - "Mark All Read" button
  - "View All" link -> `/notifications` full page

#### `/notifications` (full page)
- Filterable list of all notifications
- Filters: Type, Read/Unread, Date range

---

### 2.11 Search (Cmd+K Modal)

- **Trigger**: Cmd+K (Mac) / Ctrl+K (Windows) or click search icon in top bar
- **Components**: shadcn/ui Command palette
- **Search Results Grouped By**:
  - RFQs: matches on rfq_number, buyer name, product_type
  - Quotes: matches on quote_number, buyer name
  - Buyers: matches on name, code
  - Cost Library: matches on item name, code
- **Recent**: Last 5 visited pages
- **Quick Actions**: "Upload RFQ", "Add Cost Item", "Invite User"

---

## 3. Responsive Behavior

| Breakpoint | Behavior |
|---|---|
| Desktop (>= 1280px) | Full sidebar + content + QUINN panel (if open) |
| Tablet (768-1279px) | Collapsible sidebar (icon-only), content full width, QUINN as overlay |
| Mobile (< 768px) | Bottom navigation bar (5 icons: Dashboard, Inbox, Quotes, Analytics, Menu), QUINN as full-screen overlay, tables become card lists |

---

## 4. Component Library (shadcn/ui)

All UI components from shadcn/ui. Key components used:

| Component | Usage |
|---|---|
| Button | All actions (primary, secondary, destructive, ghost variants) |
| Card | Dashboard KPIs, stat widgets |
| Table | All data tables (RFQs, quotes, cost library, etc.) |
| Dialog | Confirmations, add/edit forms |
| Sheet | Side panels (add cost item, edit buyer) |
| Select | Dropdowns (status, buyer, priority, etc.) |
| Input | Text inputs, search |
| Textarea | Notes, descriptions |
| Badge | Status labels, priority, tier |
| Tabs | Section navigation within pages |
| Command | Cmd+K search palette |
| Tooltip | Hover help text |
| Avatar | User avatars in assignments, activity feed |
| Progress | Confidence score bars |
| Slider | Margin control |
| Switch | Toggle settings (active/inactive) |
| Calendar + DatePicker | Date range filters, deadline selection |
| DropdownMenu | Row actions, bulk actions |
| Toast | Success/error feedback notifications |
| Skeleton | Loading states for all data-fetching areas |
| AlertDialog | Destructive action confirmations |

---

## 5. Theme & Branding

| Property | Value |
|---|---|
| Primary Color | #2563EB (blue-600) |
| Secondary Color | #059669 (emerald-600) |
| Destructive Color | #DC2626 (red-600) |
| Warning Color | #D97706 (amber-600) |
| Background | #FFFFFF (light) / #0F172A (dark) |
| Sidebar Background | #F8FAFC (light) / #1E293B (dark) |
| Font | Inter (sans-serif) |
| Border Radius | 8px (rounded-lg) |
| Dark Mode | Supported via next-themes, toggle in user menu |

---

*This document is confidential and intended for internal use only.*
*fabricXai is a product of SocioFi Technology Ltd.*
