# Product Requirements Document
## RFQX — AI RFQ & Quoting Intelligence for Garment Manufacturers
**Version:** 1.0  
**Status:** Pre-Build — Design Partner Phase  
**Owner:** fabricXai / SocioFi Technology Ltd.  
**Last Updated:** March 2026

---

## 1. Product Overview

### 1.1 What Is RFQX?

RFQX is an AI-powered RFQ (Request for Quotation) processing and quoting agent built for mid-size garment manufacturers. It reads incoming RFQs from any channel — email, PDF, WhatsApp, scanned documents, buyer portals — extracts all specification data automatically, calculates costs using the factory's live cost library, and generates a formatted quote ready to send.

RFQX is one of 22 specialized agents in the fabricXai platform and operates as a standalone SaaS product or as part of the integrated fabricXai Garment Operating System. Its copilot is **QUINN**, a conversational AI that lets factory teams manage their entire quoting workflow through plain-language chat.

### 1.2 The Problem

Garment factories in Bangladesh, Vietnam, India, Turkey, and Cambodia lose revenue daily because their quoting process is slow and manual:

- RFQs arrive across multiple channels simultaneously — email, WhatsApp, PDF attachments, buyer web portals — and someone must read each one, extract specs by hand, and enter data into Excel.
- A single RFQ can take 2–4 hours to cost manually: reading the tech pack, checking supplier price lists, applying margins, formatting the response in the buyer's preferred template.
- Factories typically handle 15–40 RFQs per month. Missing a 24-hour response window is common, and buyers often award orders to whoever quotes first.
- Quote errors caused by manual data entry — wrong quantities, incorrect fabric specs, outdated supplier prices — lead to chargebacks and margin erosion after orders are placed.
- There is no visibility into which RFQs are open, which are stuck, which buyers are waiting, or what the win rate is.

### 1.3 The Solution

RFQX eliminates the manual steps between receiving an RFQ and sending a quote:

1. **Capture** — Monitors email inboxes, buyer portals, and communication channels for incoming RFQs.
2. **Extract** — Reads every RFQ format (PDF, Excel, email body, scanned image, WhatsApp) and extracts structured specification data with confidence scoring.
3. **Cost** — Applies the factory's live cost library (fabric, CMT, trims, washing, packing, overheads, freight) to build a full cost breakdown automatically.
4. **Quote** — Generates a buyer-formatted quote in the buyer's preferred template, with FOB price, margin, and optional landed cost breakdown.
5. **Send & Track** — Sends the quote through the appropriate channel and tracks open, response, and win/loss status.
6. **Learn** — Analyzes win rates, buyer patterns, and pricing trends to improve future quotes.

QUINN, the RFQX copilot, lets the team interact with all of this through natural conversation — asking questions, making corrections, triggering actions, and getting status updates without navigating menus.

---

## 2. Target Users

### 2.1 Primary Markets

- Bangladesh (priority Year 1: 50–100 factories)
- Vietnam, India, Turkey, Cambodia (Year 2 expansion)
- Factory size: 500–5,000 workers, 15–100 RFQs/month

### 2.2 User Roles

| Role | Primary Use | Key Pain Point |
|---|---|---|
| **Factory Owner / Director** | Dashboard overview, win rate, revenue pipeline | No visibility into quoting performance or pipeline |
| **Quotation Manager / Costing Head** | Reviewing and approving AI-generated quotes | Manually building cost sheets from scratch in Excel |
| **Merchandiser** | Managing RFQ inbox, coordinating with buyers | RFQs scattered across email, WhatsApp, portals |
| **Commercial Executive** | Sending quotes, tracking buyer responses | Following up manually with no tracking system |
| **Production Manager** | Reviewing capacity before quote confirmation | Not consulted during quoting; overcommitments happen |

### 2.3 User Persona — Primary

**Rahim, Quotation Manager, Dhaka**  
Age 34. Manages quoting for a 1,200-worker woven factory. Receives 25–35 RFQs per month from buyers including H&M, Primark, and ASOS. Currently runs everything through Gmail, WhatsApp, and a shared Excel cost sheet. Works until 9 PM most days during peak seasons. His biggest frustration: tech packs arrive in different formats every time, and he re-enters the same data into Excel for every single quote.

---

## 3. Goals and Success Metrics

### 3.1 Product Goals

| Goal | Metric | Target |
|---|---|---|
| Reduce quote turnaround time | Hours from RFQ receipt to quote sent | From 4+ hours → under 30 minutes |
| Increase RFQ response rate | % of RFQs quoted within 24 hours | From ~60% → 95%+ |
| Improve quote accuracy | % of quotes with zero pricing errors | From ~70% → 98%+ |
| Increase win rate | Quote-to-order conversion | +5–15 percentage points |
| Reduce manual data entry | Hours per week on costing | From 15–20 hours → under 2 hours |
| Provide visibility | Factory owner checks dashboard weekly | Adoption: 80%+ weekly active |

### 3.2 Business Goals (Year 1)

- 50 paying factories in Bangladesh
- Average contract value: $499/month (mid tier)
- Monthly Recurring Revenue target: $24,950 by Month 12
- Churn target: under 5% monthly
- Net Promoter Score: 50+

---

## 4. Core Use Cases

### UC-01: Incoming RFQ from Buyer Email

**Actor:** Merchandiser / Quotation Manager  
**Trigger:** Buyer sends RFQ via email with PDF tech pack attached  
**Flow:**
1. RFQX detects the email via Gmail/Outlook integration
2. QUINN alerts the user: "New RFQ from H&M — 50,000 units polo shirts. I've extracted the specs."
3. Extracted data is displayed: fabric (220 GSM piqué), quantity, delivery port, required certifications
4. User reviews extraction; corrects any misread field
5. RFQX builds cost sheet using factory's live cost library
6. User reviews cost breakdown and adjusts margin
7. RFQX generates PDF quote in H&M's preferred CIQ template
8. User approves; quote is sent and logged

### UC-02: Buyer Portal RFQ

**Actor:** Merchandiser  
**Trigger:** New RFQ appears on Primark Vendor Portal  
**Flow:**
1. RFQX portal watcher detects new RFQ
2. Specs are extracted from portal form fields directly
3. Cost sheet built automatically; quote formatted in Primark template
4. Merchandiser reviews and submits via portal from inside RFQX

### UC-03: WhatsApp RFQ (Informal)

**Actor:** Commercial Executive  
**Trigger:** Buyer sends specs over WhatsApp message  
**Flow:**
1. WhatsApp integration captures the message
2. QUINN extracts key fields (product, quantity, fabric, delivery date) from the unstructured message
3. Low-confidence fields are flagged for manual confirmation
4. Cost sheet and quote generated after user confirms

### UC-04: Bulk RFQ Season (15+ simultaneous)

**Actor:** Factory Owner / Quotation Manager  
**Trigger:** Multiple buyers send RFQs during peak buying season  
**Flow:**
1. Dashboard shows all open RFQs in prioritized queue
2. RFQX auto-routes RFQs to team members by buyer/product type
3. Quotation Manager reviews AI-built cost sheets for high-value RFQs
4. Low-value, high-confidence RFQs are auto-quoted with approval
5. Owner sees real-time pipeline value and response rate

### UC-05: Quote Follow-Up

**Actor:** Commercial Executive  
**Trigger:** 4 hours pass with no buyer response after quote sent  
**Flow:**
1. QUINN alerts: "Primark opened your quote. No response in 4 hours. Follow up?"
2. User approves follow-up; QUINN sends a polite follow-up email from the factory's email address
3. When buyer responds, status is updated and user is notified

---

## 5. Functional Requirements

### 5.1 RFQ Inbox & Capture

- **Email Integration:** Connect Gmail, Outlook, Yahoo, IMAP. Detect RFQs automatically. Support multiple factory email addresses.
- **Buyer Portal Watchers:** Monitor Primark, H&M, ASOS, Zara vendor portals. Pull new RFQs without manual login.
- **WhatsApp Integration:** Read incoming messages from buyer contacts. Support group chats.
- **File Upload:** Manual upload of PDF, Word, Excel, scanned images, photos.
- **Multi-Language Support:** English, Bengali, Hindi, Vietnamese, Turkish, Mandarin.
- **Deduplication:** Detect and merge duplicate RFQs from same buyer across channels.
- **Priority Scoring:** Assign priority 1–5 based on buyer tier, deadline, and RFQ value.

### 5.2 Spec Extraction (AI)

- **Structured Extraction:** Read product type, fabric composition, GSM, quantity, colorways, sizes, embellishments, washing instructions, certifications, delivery port, delivery date, payment terms, packaging requirements.
- **Confidence Scoring:** Display confidence level per extracted field. Flag any field below 85% for human review.
- **Accuracy Targets:** 94–96% on structured spec sheets; 88–92% on unstructured email/WhatsApp.
- **Skill System:** Industry-specific extraction models (Garment, Footwear, Home Textile, Industrial). Tenant can select active skill.
- **Override Logging:** All manual corrections logged with timestamp and user ID for audit and model improvement.
- **OCR Support:** Extract from scanned documents and photos with Tesseract or equivalent.

### 5.3 Cost Library

- **Cost Items:** Fabric (per GSM/composition), CMT (per product type), Trims (by type), Washing (by process), Embroidery (per 1,000 stitches), Packing (per unit), Overheads (% of cost), Freight (per port/incoterm), Agent commission, testing fees.
- **Supplier Price Management:** Upload and maintain supplier price lists. Link fabric cost to supplier code.
- **Version History:** Track cost library changes by date. Allow quoting against historical cost snapshot.
- **Import from Excel:** Parse existing cost sheets uploaded by user. Map columns to RFQX fields.
- **Currency Support:** BDT, USD, EUR, GBP, VND, INR. Auto-conversion using live or locked exchange rate.
- **CMT Rate Cards:** Pre-configured rates by product type, MOQ bracket, and season.

### 5.4 Quote Builder

- **Automated Cost Build:** Full cost breakdown built from extracted specs + cost library. No manual entry required for standard quotes.
- **Line-Item View:** Fabric / CMT / Trims / Wash / Pack / OH / Freight shown as separate lines with per-unit and total values.
- **Margin Control:** Set target margin %. See FOB price update in real time.
- **FOB/CIF/EXW/DDP Toggle:** Switch incoterm; landed cost adjustments applied automatically.
- **Scenario Compare:** Build 2–3 pricing scenarios (different margins, different fabric sources) and compare.
- **Buyer-Specific Templates:** Store quote format preferences per buyer (H&M CIQ, Primark Supplier Form, Zara B2B, etc.).
- **PDF Generation:** Generate formatted quote PDF in buyer's template. Include factory letterhead.
- **Quote Versioning:** Track all revisions. Buyer always receives the latest version; previous versions retained.

### 5.5 Quote Tracker

- **Status Pipeline:** New → Extracting → Costing → Review → Sent → Followed Up → Won/Lost/Expired.
- **Deadline Alerts:** Notify user 24 hours before RFQ deadline. Escalate to owner if unactioned.
- **Email Tracking:** Detect when buyer opens the quote email (read receipt / tracking pixel).
- **Response Logging:** Log buyer replies, counter-offers, and negotiation notes against each quote.
- **Assignment:** Assign RFQs and quotes to team members. Track workload per user.
- **Bulk Actions:** Select multiple RFQs; bulk assign, bulk archive, bulk export.

### 5.6 Analytics Dashboard

- **Win Rate:** Overall and per buyer, per product category, per season.
- **Response Time:** Average time from RFQ receipt to quote sent.
- **Pipeline Value:** Total FOB value of open quotes by status.
- **Revenue Forecast:** Estimated monthly revenue if open quotes convert at historical win rate.
- **Buyer Analysis:** Volume, win rate, average order value per buyer. Identify declining or growing accounts.
- **Cost Trends:** Track fabric cost movement over time. Alert when key input costs change significantly.
- **Margin Analysis:** Average margin by product, by buyer, by season.
- **Export:** All analytics exportable to Excel or CSV.

### 5.7 QUINN Copilot

- **Natural Language Interface:** Chat panel available on every screen. QUINN understands plain English and can also read Bengali, Hindi, Vietnamese, Turkish.
- **Context Awareness:** QUINN knows which RFQ or quote the user is currently viewing. No need to repeat context.
- **Action Execution:** QUINN can take actions — update a field, assign an RFQ, generate a quote, send an email — via chat commands.
- **Proactive Alerts:** QUINN surfaces urgent items without being asked: new RFQs, buyer responses, approaching deadlines.
- **Conversation History:** Last 20 messages retained per session. Archival to database for long-term memory (post-launch roadmap).
- **Quick Actions:** Pre-built shortcuts for common tasks. User can trigger with "/" commands.
- **Response Format:** Markdown, tables, clickable deep-links, action buttons. Responses under 300 words unless user asks for detail.

### 5.8 User Management & Permissions

- **Roles:** Owner, Admin, Quotation Manager, Merchandiser, Commercial Executive, Viewer.
- **Permissions:** Role-based access control. Owner sees all; Viewer sees analytics only; Merchandiser cannot approve quotes.
- **Multi-Factory:** Enterprise accounts can manage multiple factories from a single login.
- **Audit Log:** All actions logged by user ID with timestamp.

### 5.9 Integrations

| Integration | Purpose | Phase |
|---|---|---|
| Gmail / Google Workspace | Email inbox monitoring | Phase 1 |
| Microsoft Outlook / Office 365 | Email inbox monitoring | Phase 1 |
| Primark Vendor Portal | Portal RFQ capture | Phase 1 |
| H&M Vendor Portal | Portal RFQ capture | Phase 2 |
| ASOS Supplier Hub | Portal RFQ capture | Phase 2 |
| Zara B2B Portal | Portal RFQ capture | Phase 2 |
| WhatsApp Business API | Message capture | Phase 2 |
| fabricXai Platform (CostX, BuyerX, PlanX) | Cross-agent data sharing | Phase 2 |
| ERP Systems (SAP, Oracle) | Cost library sync | Phase 3 |

---

## 6. Non-Functional Requirements

### 6.1 Performance

| Metric | Target |
|---|---|
| API response time (p50) | < 200ms |
| API response time (p99) | < 800ms |
| RFQ extraction time | < 60 seconds |
| Quote generation time | < 30 seconds |
| Email monitoring latency | < 5 minutes |
| QUINN response time | < 4 seconds |
| Uptime SLA | 99.5% |

### 6.2 Security & Compliance

- Data encrypted at rest (AES-256) and in transit (TLS 1.3)
- GDPR compliant — EU factory data stored in EU region
- SOC 2 Type II roadmap (Year 2)
- ISO 27001 roadmap (Year 2)
- No buyer data shared across tenant boundaries
- Factory cost library data is tenant-private and never used to train shared models without explicit opt-in
- MFA required for Owner and Admin roles

### 6.3 Accessibility & Usability

- UI in English; QUINN supports 6 languages
- Mobile-responsive web app; no native app required in Phase 1
- No IT setup required — onboarding via browser, no software installation
- Target: factory staff with no software training can perform primary tasks within 2 hours

---

## 7. Technical Architecture

### 7.1 Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14, TypeScript, Tailwind CSS, shadcn/ui |
| Backend | Node.js / TypeScript API |
| Database | Supabase (PostgreSQL + pgvector) |
| Auth | Supabase Auth |
| File Storage | Supabase Storage |
| Cache | Redis |
| AI / LLM | Claude claude-opus-4-5 (temperature 0.3 for cost calculations) |
| OCR | Tesseract / cloud OCR service |
| Deployment | Vercel (frontend), Railway or Render (backend) |
| Queue | BullMQ / Redis |

### 7.2 AI Sub-Agents

RFQX uses 5 specialized AI sub-agents coordinated by a code orchestrator:

| Agent | Responsibility |
|---|---|
| **InboxWatcher** | Monitor channels, detect RFQ signals, deduplicate |
| **ExtractorAgent** | Parse documents and messages, extract structured specs with confidence scoring |
| **CostEngine** | Apply cost library rules, build line-item cost breakdown |
| **QuoteWriter** | Format cost breakdown into buyer-specific quote template, generate PDF |
| **TrackingAgent** | Monitor quote delivery, detect opens, trigger follow-up alerts |

QUINN sits above these sub-agents as an orchestration layer, routing user requests to the right agent or API call via MCP tools.

### 7.3 Data Model (Core Entities)

- **Tenant** — factory account (multi-factory supported at enterprise tier)
- **RFQ** — incoming request with status, extracted_data (JSON), source channel, buyer_id, assigned_user_id, deadline, priority
- **Quote** — linked to RFQ, contains cost breakdown (JSON), margin, FOB price, incoterm, version history, status
- **CostLibrary** — tenant-specific cost items by category, linked to supplier codes
- **Buyer** — buyer profile with portal credentials, template preferences, win rate history
- **User** — role, factory assignment, notification preferences

---

## 8. Pricing

| Tier | Price | Limits |
|---|---|---|
| **Starter** | $299/month | 20 RFQs/month, 2 users, email only, core templates |
| **Growth** | $599/month | 75 RFQs/month, 10 users, all channels, all templates, basic analytics |
| **Scale** | $999/month | Unlimited RFQs, unlimited users, portal integrations, advanced analytics, API access, dedicated CSM |

BDT pricing equivalents displayed in-app for Bangladesh users. Annual billing available at 20% discount.

---

## 9. Launch Strategy

### 9.1 Design Partner Program (Pre-Launch)

- 3 Bangladesh factories recruited with free access in exchange for weekly feedback
- Goal: validate extraction accuracy, cost library import workflow, and QUINN usefulness
- Success criteria: all 3 partners actively quoting through RFQX within 30 days

### 9.2 Go-To-Market (Year 1)

- Primary channel: direct outreach to factory owners via WhatsApp and LinkedIn
- Secondary channel: BGMEA network partnerships
- Key message: "Quote in 30 minutes, not 4 hours — or we'll refund your first month"
- Land-and-expand: RFQX as entry point → upsell to CostX, VerifyX, PlanX

### 9.3 Phased Rollout

| Phase | Timeline | Milestone |
|---|---|---|
| Design Partner | Month 1–2 | 3 factories live, extraction accuracy validated |
| Closed Beta | Month 3–4 | 20 factories, $299 tier, email channel only |
| General Availability | Month 5 | All tiers, all channels, portal integrations |
| Platform Integration | Month 6+ | RFQX connects to CostX, BuyerX, PlanX |

---

## 10. Out of Scope (v1.0)

- ERP integration (SAP, Oracle) — Phase 3
- Native mobile app — web is mobile-responsive; app in roadmap
- Automated order confirmation — RFQX quotes, humans confirm
- Real-time supplier price feeds — manual update only in v1
- Voice input for QUINN — post-launch roadmap
- Industrial / PPE skill — Q3 2026
- Multi-agent QUINN routing to other fabricXai agents — platform integration phase

---

## 11. Risks and Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Extraction accuracy below target on unusual formats | Medium | Confidence scoring + mandatory human review below 85%; user correction feeds improvement |
| Factories unwilling to move from Excel | High | RFQX imports existing Excel cost sheets; QUINN fills data entry gap |
| Buyer portal anti-scraping measures | Medium | Use official vendor portal APIs where available; fallback to supervised portal access |
| Factory staff distrust AI pricing | Medium | Full line-item transparency; user always reviews before sending; no auto-send without approval in v1 |
| WhatsApp Business API approval delays | Low-Medium | Launch without WhatsApp; add in Phase 2 after Meta approval |

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| RFQ | Request for Quotation — a buyer's request for a price on a product |
| CMT | Cut, Make & Trim — the core manufacturing cost charged by a factory |
| FOB | Free on Board — a pricing incoterm where cost includes delivery to origin port |
| GSM | Grams per Square Meter — fabric weight measurement |
| TNA | Time & Action calendar — a production schedule |
| AQL | Acceptable Quality Level — standard for inspection sampling |
| CIQ | Cost, Insurance & Quote — buyer-specific quote format (Primark) |
| Tech Pack | Technical specification document for a garment, including materials, measurements, and construction details |
| Chargeback | Financial penalty imposed by a buyer for quality, delivery, or compliance failures |
| Tenant | A single factory account in the RFQX multi-tenant system |

---

*This document is confidential and intended for internal use and authorized design partners only.*  
*fabricXai is a product of SocioFi Technology Ltd.*
