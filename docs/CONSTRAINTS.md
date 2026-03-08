# RFQX Constraints & Infrastructure Specification
## Version 1.0 | March 2026

---

## 1. Authentication & Authorization

### 1.1 Auth Method

- **Provider**: Supabase Auth
- **Methods**:
  - Email + password (primary)
  - Google OAuth 2.0 (social login)
  - Magic link (passwordless, email-based)
- **Token Format**: JWT (RS256), issued by Supabase Auth
- **Access Token Lifetime**: 3600 seconds (1 hour)
- **Refresh Token Lifetime**: 604800 seconds (7 days)
- **Session Storage**: HttpOnly secure cookie (access token) + localStorage (refresh token)

### 1.2 MFA

- Required for: `owner` and `admin` roles
- Optional for: all other roles
- Method: TOTP (Time-based One-Time Password) via authenticator app
- Enforcement: On login, if role requires MFA and not enrolled, redirect to MFA setup before granting access

### 1.3 Password Policy

- Minimum length: 8 characters
- Must contain: 1 uppercase letter, 1 lowercase letter, 1 digit
- No maximum length enforced (Supabase default)
- Failed login lockout: 5 failed attempts in 15 minutes triggers 30-minute lockout
- Password reset link expiry: 1 hour

### 1.4 Role-Based Access Control (RBAC)

| Permission | Owner | Admin | Quotation Manager | Merchandiser | Commercial Executive | Viewer |
|---|---|---|---|---|---|---|
| View dashboard | yes | yes | yes | yes | yes | yes |
| View RFQ list | yes | yes | yes | yes | yes | yes |
| View RFQ detail | yes | yes | yes | yes | yes | yes |
| Create RFQ (upload) | yes | yes | yes | yes | no | no |
| Edit RFQ fields | yes | yes | yes | yes | no | no |
| Assign RFQ | yes | yes | yes | no | no | no |
| Approve extraction | yes | yes | yes | no | no | no |
| View cost library | yes | yes | yes | no | no | no |
| Edit cost library | yes | yes | yes | no | no | no |
| Create quote | yes | yes | yes | no | no | no |
| Edit quote | yes | yes | yes | no | no | no |
| Approve quote | yes | yes | yes | no | no | no |
| Send quote | yes | yes | yes | no | yes | no |
| View analytics | yes | yes | yes | yes | yes | yes |
| Manage users | yes | yes | no | no | no | no |
| Manage billing | yes | no | no | no | no | no |
| Manage integrations | yes | yes | no | no | no | no |
| View audit log | yes | yes | no | no | no | no |
| Manage buyers | yes | yes | yes | yes | no | no |
| Use QUINN | yes | yes | yes | yes | yes | no |
| QUINN actions | scoped to user's permissions | | | | | |

### 1.5 Tenant Isolation

- All database queries scoped by `tenant_id` via PostgreSQL Row Level Security (RLS)
- RLS policies enforce that users can only access data belonging to their tenant
- No cross-tenant data access is possible at the database level
- Supabase Auth `uid()` mapped to `users.supabase_auth_id` -> `users.tenant_id`
- Storage bucket paths prefixed with `tenant_id`

---

## 2. Environment Variables

### 2.1 Required Variables

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxxxxxxxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_DB_URL=postgresql://postgres:password@db.xxxxxxxxxxxx.supabase.co:5432/postgres

# Redis
REDIS_URL=redis://default:password@hostname:6379

# Anthropic (Claude API)
ANTHROPIC_API_KEY=sk-ant-api03-...

# Encryption (for portal credentials, inbox credentials)
ENCRYPTION_KEY=32-byte-hex-string  # AES-256 key for encrypting stored credentials

# Application
NEXT_PUBLIC_APP_URL=https://app.rfqx.ai
NODE_ENV=production
```

### 2.2 Optional / Feature-Specific Variables

```bash
# Google OAuth (for Gmail integration + Google login)
GOOGLE_CLIENT_ID=xxxxxxxxxxxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-...
GOOGLE_REDIRECT_URI=https://app.rfqx.ai/api/v1/auth/callback/google

# Microsoft OAuth (for Outlook integration)
MICROSOFT_CLIENT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
MICROSOFT_CLIENT_SECRET=...
MICROSOFT_REDIRECT_URI=https://app.rfqx.ai/api/v1/auth/callback/microsoft

# Google Cloud Vision (OCR)
GOOGLE_CLOUD_VISION_API_KEY=AIza...
# OR service account JSON
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json

# Stripe (billing)
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...

# Nylas (email API - alternative to direct IMAP)
NYLAS_CLIENT_ID=...
NYLAS_CLIENT_SECRET=...
NYLAS_API_KEY=...

# WhatsApp Business API
WHATSAPP_PHONE_NUMBER_ID=...
WHATSAPP_ACCESS_TOKEN=...
WHATSAPP_VERIFY_TOKEN=rfqx-webhook-verify-token
WHATSAPP_BUSINESS_ACCOUNT_ID=...

# Sentry (error tracking)
SENTRY_DSN=https://xxxxxxxxxxxx@oxxxxxxx.ingest.sentry.io/xxxxxxx
NEXT_PUBLIC_SENTRY_DSN=https://xxxxxxxxxxxx@oxxxxxxx.ingest.sentry.io/xxxxxxx

# Email sending (transactional emails: invites, notifications, password reset)
RESEND_API_KEY=re_...
EMAIL_FROM_ADDRESS=noreply@rfqx.ai
EMAIL_FROM_NAME=RFQX

# Exchange rate API (optional, for live rates)
EXCHANGE_RATE_API_KEY=...
EXCHANGE_RATE_API_URL=https://api.exchangerate-api.com/v4/latest/USD
```

### 2.3 Environment-Specific Defaults

| Variable | Development | Staging | Production |
|---|---|---|---|
| NODE_ENV | development | staging | production |
| NEXT_PUBLIC_APP_URL | http://localhost:3000 | https://staging.rfqx.ai | https://app.rfqx.ai |
| Log level | debug | info | warn |
| Rate limiting | disabled | enabled (relaxed) | enabled (strict) |
| Email sending | console output only | sandbox mode | live |
| Stripe | test keys | test keys | live keys |

---

## 3. Deployment

### 3.1 Deployment Targets

| Component | Platform | Region | Scaling |
|---|---|---|---|
| Next.js Frontend + API Routes | Vercel | `iad1` (US East) + edge | Auto-scaling (serverless) |
| Background Workers | Railway | US East | 1 instance per worker type (3 total), vertical scale |
| PostgreSQL | Supabase | US East (AWS `us-east-1`) | Supabase Pro plan (8GB RAM, 100GB storage) |
| Redis | Upstash (serverless) or Railway | US East | 256MB memory, auto-scaling |
| File Storage | Supabase Storage (S3-backed) | US East | 100GB included |

### 3.2 CI/CD Pipeline

```
GitHub Push -> GitHub Actions:
  1. Lint (ESLint + Prettier)
  2. Type check (tsc --noEmit)
  3. Unit tests (Vitest)
  4. E2E tests (Playwright, against staging Supabase)
  5. Build (next build)
  6. Deploy to Vercel (preview for PRs, production for main)
  7. Deploy workers to Railway (on main merge)
  8. Run database migrations (Supabase CLI)
```

### 3.3 Database Migrations

- Tool: Supabase CLI (`supabase db push` for development, `supabase migration` for production)
- Migration files stored in: `supabase/migrations/`
- Naming: `YYYYMMDDHHMMSS_description.sql`
- All migrations run in a transaction
- Rollback: manual SQL scripts in `supabase/migrations/rollback/`

### 3.4 EU Data Residency

- For GDPR compliance, EU-based factory tenants will have their Supabase project in EU region (`eu-west-1`)
- Phase 1: Single region (US East) for all tenants
- Phase 2: Multi-region support with tenant-level region selection
- EU region deployment requires separate Supabase project and routing logic

---

## 4. Performance Requirements

### 4.1 Response Time Targets

| Metric | Target | Measurement |
|---|---|---|
| API response (p50) | < 200ms | Vercel function duration |
| API response (p99) | < 800ms | Vercel function duration |
| Page load (TTFB) | < 500ms | Vercel Edge |
| Page load (LCP) | < 2.5s | Lighthouse |
| RFQ extraction time | < 60 seconds | Worker job duration (queue to completion) |
| Quote generation (cost calc) | < 30 seconds | Worker job duration |
| PDF generation | < 15 seconds | Worker job duration |
| QUINN chat response (first token) | < 2 seconds | Streaming SSE |
| QUINN chat response (complete) | < 4 seconds | Streaming SSE |
| Email monitoring latency | < 5 minutes | Time between email arrival and RFQ creation |
| Dashboard data freshness | < 60 seconds | Analytics cache TTL |

### 4.2 Throughput Targets

| Metric | Target |
|---|---|
| Concurrent users per tenant | 25 |
| Total concurrent users (all tenants) | 500 |
| RFQ extraction jobs/minute | 10 |
| Quote generation jobs/minute | 20 |
| QUINN chat sessions concurrent | 100 |
| Email inboxes monitored simultaneously | 500 |
| API requests per minute (per tenant) | 1000 |

### 4.3 Uptime

- Target SLA: 99.5% uptime (monthly)
- Allowed downtime: ~3.65 hours/month
- Maintenance windows: Sundays 02:00-04:00 UTC (announced 48h in advance)
- Scale tier: 99.9% SLA (contractual)

### 4.4 Database Performance

- Connection pooling: Supabase PgBouncer (transaction mode)
- Max connections: 60 (Supabase Pro plan default)
- Slow query threshold: 1000ms (logged to Sentry)
- Index coverage: All foreign keys, all filter/sort columns (defined in TECH_SPEC.md)
- Vacuum schedule: Supabase managed (auto-vacuum)

---

## 5. File Upload Constraints

### 5.1 Size Limits

| Upload Type | Max File Size | Max Files Per Request |
|---|---|---|
| RFQ attachment (PDF, DOCX, XLSX) | 25 MB | 10 |
| RFQ attachment (image: PNG, JPG) | 10 MB | 10 |
| Cost library import (XLSX, CSV) | 10 MB | 1 |
| Factory logo (PNG, JPG, SVG) | 2 MB | 1 |
| Factory letterhead (PDF, PNG) | 5 MB | 1 |

### 5.2 Allowed File Types

| Purpose | Allowed MIME Types | Extensions |
|---|---|---|
| RFQ attachments | application/pdf, application/vnd.openxmlformats-officedocument.spreadsheetml.sheet, application/vnd.ms-excel, application/vnd.openxmlformats-officedocument.wordprocessingml.document, application/msword, text/csv, image/png, image/jpeg | .pdf, .xlsx, .xls, .docx, .doc, .csv, .png, .jpg, .jpeg |
| Cost library import | application/vnd.openxmlformats-officedocument.spreadsheetml.sheet, application/vnd.ms-excel, text/csv | .xlsx, .xls, .csv |
| Logo | image/png, image/jpeg, image/svg+xml | .png, .jpg, .jpeg, .svg |
| Letterhead | application/pdf, image/png | .pdf, .png |

### 5.3 Storage Limits by Plan

| Plan | Total Storage | Retention |
|---|---|---|
| Starter | 5 GB | 12 months |
| Growth | 25 GB | 24 months |
| Scale | 100 GB | Unlimited |

### 5.4 File Processing

- All uploaded files scanned for malware via ClamAV (or equivalent) before storage
- Images > 4000px in any dimension are resized to 4000px max
- PDFs are not modified
- OCR is triggered automatically on image uploads and scanned PDFs (detected by lack of text layer)
- Original files are always preserved; OCR text stored separately in `rfq_attachments.ocr_text`

---

## 6. Rate Limiting

### 6.1 API Rate Limits

| Endpoint Group | Limit | Window | Scope |
|---|---|---|---|
| Auth endpoints (`/auth/*`) | 10 requests | 1 minute | Per IP address |
| QUINN chat (`/quinn/chat`) | 30 requests | 1 minute | Per user |
| File upload (`/upload`) | 20 requests | 1 minute | Per user |
| All other API endpoints | 100 requests | 1 minute | Per user |
| Scale tier API access | 1000 requests | 1 minute | Per tenant |
| Webhooks (incoming) | 500 requests | 1 minute | Per source |

### 6.2 Rate Limit Response

```
HTTP 429 Too Many Requests
Headers:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 0
  X-RateLimit-Reset: 1709812345 (Unix timestamp)
  Retry-After: 45 (seconds)

Body:
{ "error": "rate_limited", "message": "Too many requests. Retry after 45 seconds." }
```

### 6.3 AI API Usage Limits

| Resource | Limit | Notes |
|---|---|---|
| Claude API calls per extraction | 1-3 calls | Multi-attachment RFQs may require multiple calls |
| Claude API calls per cost calculation | 1 call | Single structured call |
| Claude API calls per QUINN message | 1-2 calls | 1 for response, optional 1 for tool execution |
| Max tokens per extraction call | 4096 output | Configurable |
| Max tokens per QUINN response | 1024 output | Hard limit |
| Monthly Claude API budget per tenant | Starter: $50, Growth: $150, Scale: $500 | Monitored, alerts at 80% |

---

## 7. Security Constraints

### 7.1 Data Encryption

| Data State | Method |
|---|---|
| Data at rest (database) | AES-256 (Supabase managed, AWS RDS encryption) |
| Data in transit | TLS 1.3 (enforced on all endpoints) |
| Stored credentials (inbox, portal) | AES-256-GCM with `ENCRYPTION_KEY` env var |
| File storage | AES-256 (Supabase Storage, S3 server-side encryption) |
| Backup encryption | AES-256 (Supabase managed) |

### 7.2 CORS Policy

```
Allowed Origins:
  - https://app.rfqx.ai
  - https://staging.rfqx.ai
  - http://localhost:3000 (development only)

Allowed Methods: GET, POST, PATCH, DELETE, OPTIONS
Allowed Headers: Authorization, Content-Type, X-Request-ID
Credentials: true
Max Age: 86400 (24 hours)
```

### 7.3 Content Security Policy

```
default-src 'self';
script-src 'self' 'unsafe-eval' 'unsafe-inline' https://js.stripe.com;
style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
font-src 'self' https://fonts.gstatic.com;
img-src 'self' data: blob: https://*.supabase.co https://lh3.googleusercontent.com;
connect-src 'self' https://*.supabase.co wss://*.supabase.co https://api.anthropic.com https://api.stripe.com;
frame-src https://js.stripe.com;
```

### 7.4 Security Headers

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

### 7.5 Data Isolation

- Tenant data is never accessible cross-tenant
- Cost library data is tenant-private and never used for shared model training
- QUINN conversation data is per-user, per-tenant
- Audit logs are per-tenant
- Analytics data is computed per-tenant only
- Supabase RLS policies enforce isolation at the database level
- Storage bucket policies enforce tenant-level access

---

## 8. Browser Support

| Browser | Minimum Version |
|---|---|
| Chrome | 90+ |
| Firefox | 90+ |
| Safari | 15+ |
| Edge | 90+ |
| Mobile Chrome (Android) | 90+ |
| Mobile Safari (iOS) | 15+ |

No support for Internet Explorer.

---

## 9. Internationalization Constraints

### 9.1 UI Language

- Phase 1: English only (UI labels, navigation, system messages)
- Phase 2: Bengali UI translation (Q4 2026)

### 9.2 QUINN Language Support

QUINN understands and responds in:
- English (primary)
- Bengali (Bangla)
- Hindi
- Vietnamese
- Turkish

QUINN always responds in the language the user writes in. If language is ambiguous, default to English.

### 9.3 Number & Date Formatting

| Locale | Number Format | Date Format | Currency Position |
|---|---|---|---|
| en-US (default) | 1,234,567.89 | MM/DD/YYYY | $1,234.56 |
| en-GB | 1,234,567.89 | DD/MM/YYYY | GBP1,234.56 |
| bn-BD | 1,234,567.89 | DD/MM/YYYY | ৳1,234.56 |

All dates stored as ISO 8601 in UTC. Displayed in user's local timezone (detected from browser).

### 9.4 Text Direction

- All languages are LTR (left-to-right)
- No RTL support required in Phase 1

---

## 10. Monitoring & Observability

### 10.1 Error Tracking

- **Tool**: Sentry
- **Capture**: All unhandled exceptions, API errors (5xx), failed queue jobs
- **Context**: User ID, tenant ID, request path, request body (sanitized)
- **Alert**: Slack notification for error rate > 1% in 5-minute window

### 10.2 Application Metrics

| Metric | Tool | Alert Threshold |
|---|---|---|
| API response time p99 | Vercel Analytics | > 2000ms for 5 minutes |
| Error rate (5xx) | Sentry | > 1% of requests in 5 minutes |
| Queue job failure rate | BullMQ dashboard | > 5% in 1 hour |
| Queue depth | BullMQ dashboard | > 100 pending jobs |
| Redis memory usage | Upstash/Railway dashboard | > 80% of allocated |
| Database connections | Supabase dashboard | > 50 active connections |
| Claude API error rate | Application logging | > 5% in 1 hour |
| Storage usage per tenant | Application check | > 90% of plan limit |

### 10.3 Logging

- Structured JSON logs via `pino` logger
- Log levels: `fatal`, `error`, `warn`, `info`, `debug`, `trace`
- Production log level: `warn`
- Sensitive data (passwords, tokens, cost data) never logged
- Request ID (`X-Request-ID` header) propagated through all logs
- Log retention: 30 days (Vercel logs), 7 days (Railway worker logs)

---

## 11. Backup & Recovery

### 11.1 Database Backups

- **Frequency**: Daily automated backups (Supabase Pro)
- **Retention**: 7 days point-in-time recovery
- **RPO (Recovery Point Objective)**: 24 hours (daily backup) or near-zero (WAL-based PITR)
- **RTO (Recovery Time Objective)**: < 1 hour

### 11.2 File Storage Backups

- Supabase Storage uses S3 with 99.999999999% durability
- No additional backup required for files
- Deleted files: soft-deleted with 30-day recovery window

### 11.3 Redis

- Redis data is ephemeral (cache + queue state)
- Queue jobs are persisted in Redis; lost jobs are re-enqueued on restart
- No backup required; cold start re-syncs from database state

---

## 12. Third-Party API Constraints

### 12.1 Anthropic Claude API

| Constraint | Value |
|---|---|
| Model | claude-opus-4-5 |
| Max input tokens | 200,000 |
| Max output tokens | 4,096 (extraction), 1,024 (QUINN) |
| Rate limit (Tier 2) | 40 RPM, 200,000 tokens/min |
| Temperature (extraction) | 0.1 |
| Temperature (cost engine) | 0.0 |
| Temperature (QUINN) | 0.4 |
| Temperature (quote writer) | 0.3 |
| Timeout | 60 seconds |
| Retry on 429/500 | 3 retries with exponential backoff (1s, 2s, 4s) |

### 12.2 Google Cloud Vision (OCR)

| Constraint | Value |
|---|---|
| Max file size | 20 MB |
| Max pages per PDF | 5 (split larger PDFs) |
| Rate limit | 1800 requests/minute |
| Supported formats | PDF, PNG, JPG, TIFF, GIF, BMP |
| Timeout | 30 seconds |

### 12.3 Stripe

| Constraint | Value |
|---|---|
| Webhook timeout | 20 seconds |
| Subscription check | On every API request (cached 5 minutes) |
| Payment methods | Credit/debit card (Visa, Mastercard, Amex) |
| Billing cycle | Monthly (1st of month) or Annual |
| Trial period | 14 days (no credit card required) |
| Grace period on failed payment | 7 days before account suspension |

### 12.4 Nylas (Email API)

| Constraint | Value |
|---|---|
| Sync frequency | Real-time webhooks + polling every 60 seconds |
| Max accounts per application | 500 (Enterprise plan) |
| Webhook timeout | 10 seconds |
| Rate limit | 10 requests/second per account |

---

## 13. Development Constraints

### 13.1 Code Standards

- TypeScript strict mode enabled (`"strict": true`)
- ESLint with `@typescript-eslint/recommended` + `next/core-web-vitals`
- Prettier for formatting (2-space indent, single quotes, trailing commas)
- No `any` types allowed (use `unknown` and type guards)
- All API endpoints must have Zod validation schemas for request bodies
- All database queries through Supabase client (no raw SQL in application code except migrations)

### 13.2 Testing Requirements

- Unit test coverage target: 80% for business logic modules (cost engine, validation, status transitions)
- E2E test coverage: Critical user paths (login, upload RFQ, build quote, send quote)
- No tests required for: UI styling, third-party library wrappers
- Test framework: Vitest (unit), Playwright (e2e)
- CI must pass all tests before merge to `main`

### 13.3 Git Workflow

- Branch naming: `feature/`, `fix/`, `chore/` prefixes
- PR required for all changes to `main`
- Minimum 1 approval for merge
- Squash merge enforced
- Conventional commits: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`

---

*This document is confidential and intended for internal use only.*
*fabricXai is a product of SocioFi Technology Ltd.*
