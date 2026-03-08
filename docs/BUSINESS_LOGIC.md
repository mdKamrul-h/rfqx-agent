# RFQX Business Logic Specification
## Version 1.0 | March 2026

---

## 1. RFQ Status Flow

### 1.1 Status Definitions

| Status | Description | Entry Condition |
|---|---|---|
| `new` | RFQ received, not yet processed | Created via inbox detection or manual upload |
| `extracting` | AI extraction in progress | Extraction job started |
| `extracted` | Extraction complete, awaiting review | Extraction job completed |
| `costing` | Cost calculation in progress | User triggers "Build Quote" or all fields >= 85% confidence (auto-proceed) |
| `review` | Quote built, awaiting human review | Cost calculation complete |
| `approved` | Quote reviewed and approved | Quotation Manager or Owner approves |
| `sent` | Quote sent to buyer | Quote delivered via email/portal |
| `followed_up` | Follow-up sent after no buyer response | Follow-up email/message sent |
| `won` | Buyer accepted quote, order confirmed | User marks as won |
| `lost` | Buyer rejected quote or chose competitor | User marks as lost |
| `expired` | RFQ deadline passed without quote or response | System auto-expires |
| `archived` | Manually archived by user | User archives |

### 1.2 Allowed Status Transitions

```
new -> extracting                     (auto: on creation if attachments present)
new -> archived                       (manual: user archives)
extracting -> extracted               (auto: extraction job completes)
extracting -> new                     (auto: extraction fails after retries)
extracted -> costing                  (manual: user approves specs / auto: all fields >= 85%)
extracted -> extracting               (manual: user triggers re-extraction)
costing -> review                     (auto: cost engine completes)
costing -> extracted                  (auto: cost engine fails, missing cost library items)
review -> approved                    (manual: quotation_manager or owner approves)
review -> costing                     (manual: user changes margin/incoterm, triggers recalc)
approved -> sent                      (manual: user sends quote)
approved -> review                    (manual: user revises before sending)
sent -> followed_up                   (manual or auto: follow-up triggered)
sent -> won                           (manual: user marks as won)
sent -> lost                          (manual: user marks as lost)
sent -> expired                       (auto: deadline passed + 7 days with no response)
followed_up -> won                    (manual: user marks as won)
followed_up -> lost                   (manual: user marks as lost)
followed_up -> expired                (auto: deadline passed + 7 days with no response)
won -> (terminal)
lost -> (terminal)
expired -> (terminal, can be manually moved to "new" for re-processing)
archived -> new                       (manual: user unarchives)
```

### 1.3 Auto-Expiry Rules

- RFQs auto-expire when: `deadline + 7 days < now()` AND status IN (`sent`, `followed_up`)
- RFQs in status `new`, `extracting`, `extracted` auto-expire when: `deadline < now()` (immediate)
- Expiry check runs every 15 minutes via BullMQ scheduled job
- Expiry triggers notification to assigned user and owner

---

## 2. Quote Status Flow

### 2.1 Status Definitions

| Status | Description |
|---|---|
| `draft` | Quote created, not yet reviewed |
| `pending_review` | Submitted for manager/owner review |
| `approved` | Approved by authorized user |
| `sent` | Delivered to buyer |
| `revised` | Superseded by a newer version |
| `accepted` | Buyer accepted (maps to RFQ "won") |
| `rejected` | Buyer rejected (maps to RFQ "lost") |
| `expired` | Validity period passed |

### 2.2 Allowed Transitions

```
draft -> pending_review               (manual: user submits for review)
draft -> approved                     (manual: if user has approval permission)
pending_review -> approved            (manual: reviewer approves)
pending_review -> draft               (manual: reviewer sends back for revision)
approved -> sent                      (manual: user sends to buyer)
approved -> draft                     (manual: user revises, creates new version)
sent -> accepted                      (manual: user marks won)
sent -> rejected                      (manual: user marks lost)
sent -> expired                       (auto: valid_until date passed)
sent -> revised                       (auto: when a new version is sent)
```

### 2.3 Quote Versioning Rules

- Creating a revision increments `version` by 1 on a new quote record
- Previous version's status is set to `revised`
- `quote_number` stays the same across versions (e.g., QT-202603-0042)
- Only the latest version can be sent to a buyer
- All versions are retained for audit trail
- Buyer always receives the latest version

### 2.4 Quote Auto-Expiry

- Quotes auto-expire when: `valid_until < CURRENT_DATE` AND status = `sent`
- Check runs daily at 00:00 UTC
- Expiry notification sent to assigned user

---

## 3. Costing Formulas

### 3.1 Cost Per Unit Calculation

All values in the tenant's `base_currency` unless specified. Conversion happens at the end using `exchange_rate_to_usd`.

```
FABRIC_COST_PER_UNIT = consumption_per_unit_meters * fabric_cost_per_meter
                    OR consumption_per_unit_kg * fabric_cost_per_kg

CMT_PER_UNIT = lookup from cmt_rate_card WHERE product_type matches AND total_quantity BETWEEN moq_min AND moq_max
               (If no bracket match, use the closest moq_max that is >= total_quantity.
                If total_quantity < smallest moq_min, use that rate.)

TRIMS_PER_UNIT = SUM(each trim item cost * quantity_per_garment)
                 Default quantity_per_garment = 1 unless specified

EMBROIDERY_PER_UNIT = (stitch_count / 1000) * rate_per_1000_stitches
                      Default stitch_count: 5000 if "embroidery" mentioned but count not specified

PRINT_PER_UNIT = num_colors * rate_per_color * num_placements
                 Default num_placements: 1

WASHING_PER_UNIT = wash_rate from cost_library WHERE wash_type matches
                   Default wash_type: "normal" if washing mentioned but type not specified
                   If no washing mentioned: 0

PACKING_PER_UNIT = packing_rate from cost_library (per unit)
                   Default: $0.15/unit if no cost library entry

OVERHEAD_PER_UNIT = CMT_PER_UNIT * (overhead_percentage / 100)
                    Default overhead_percentage: tenant.overhead_percentage (default 10%)

TESTING_PER_UNIT = total_testing_cost / total_quantity
                   total_testing_cost = SUM(each required test cost from cost_library)
                   If no tests specified: 0

COMMISSION_PER_UNIT = SUBTOTAL * (commission_percentage / 100)
                      SUBTOTAL = FABRIC + CMT + TRIMS + EMBROIDERY + PRINT + WASHING + PACKING + OVERHEAD + TESTING
                      commission_percentage = buyer.commission_percentage OR tenant.commission_percentage (default 5%)

FINANCE_COST_PER_UNIT = SUBTOTAL * (finance_cost_percentage / 100)
                        finance_cost_percentage = tenant.finance_cost_percentage (default 2%)

COST_PER_UNIT = FABRIC + CMT + TRIMS + EMBROIDERY + PRINT + WASHING + PACKING + OVERHEAD + TESTING + COMMISSION + FINANCE_COST

MARGIN_PER_UNIT = COST_PER_UNIT * (margin_percentage / (100 - margin_percentage))

FOB_PER_UNIT = COST_PER_UNIT + MARGIN_PER_UNIT
```

### 3.2 Incoterm Adjustments

```
If incoterm = "FOB":
  PRICE_PER_UNIT = FOB_PER_UNIT
  FREIGHT_PER_UNIT = 0 (included in cost but not added to FOB price for buyer)

If incoterm = "CIF":
  FREIGHT_PER_UNIT = lookup from cost_library WHERE category = "freight"
                     AND origin_port = tenant default port
                     AND destination_port = rfq.delivery_port
  PRICE_PER_UNIT = FOB_PER_UNIT + FREIGHT_PER_UNIT + INSURANCE_PER_UNIT
  INSURANCE_PER_UNIT = FOB_PER_UNIT * 0.005 (0.5% of FOB)

If incoterm = "EXW":
  PRICE_PER_UNIT = FOB_PER_UNIT - LOCAL_TRANSPORT_PER_UNIT
  LOCAL_TRANSPORT_PER_UNIT = $0.03/unit default (or from cost library freight category where destination_port = "factory_gate")

If incoterm = "DDP":
  PRICE_PER_UNIT = FOB_PER_UNIT + FREIGHT_PER_UNIT + INSURANCE_PER_UNIT + DUTY_PER_UNIT + DESTINATION_CHARGES_PER_UNIT
  DUTY_PER_UNIT = FOB_PER_UNIT * duty_rate (configured per destination country, default 12%)
  DESTINATION_CHARGES_PER_UNIT = $0.05/unit default
```

### 3.3 FOB Price Reverse Calculation (from buyer counter-offer)

When buyer proposes a price, calculate the resulting margin:

```
BUYER_PRICE_PER_UNIT = counter_offer_price
MARGIN_PER_UNIT = BUYER_PRICE_PER_UNIT - COST_PER_UNIT
MARGIN_PERCENTAGE = (MARGIN_PER_UNIT / BUYER_PRICE_PER_UNIT) * 100

If MARGIN_PERCENTAGE < 0: flag as "below cost"
If MARGIN_PERCENTAGE < 5: flag as "dangerously low margin"
If MARGIN_PERCENTAGE < 10: flag as "below typical margin"
```

### 3.4 Fabric Consumption Calculation

If `consumption_per_unit_meters` is not in the cost library for the specific fabric:

```
Default consumption by product type:
  polo_shirt:     1.60 meters
  t_shirt:        1.40 meters
  dress_shirt:    1.80 meters
  trouser:        1.50 meters
  shorts:         0.90 meters
  jacket:         2.20 meters
  hoodie:         2.00 meters
  dress:          2.50 meters
  skirt:          1.20 meters
  underwear:      0.40 meters

Consumption adjustment for fabric width:
  If fabric_width < 140cm: consumption * 1.15 (15% extra)
  If fabric_width > 160cm: consumption * 0.90 (10% less)
  Default fabric_width: 150cm

Consumption adjustment for size ratio:
  If avg size > L (50%+ of quantity is L/XL/XXL): consumption * 1.08
  Standard ratio assumed if no size breakdown provided
```

### 3.5 SAM (Standard Allowed Minutes) Reference

Default SAM values by product type when not in CMT rate card:

| Product Type | Default SAM | Default CMT/Unit (USD) |
|---|---|---|
| t_shirt | 8.0 | $0.90 |
| polo_shirt | 12.5 | $1.40 |
| dress_shirt | 18.0 | $2.00 |
| trouser | 15.0 | $1.70 |
| shorts | 10.0 | $1.10 |
| jacket | 25.0 | $2.80 |
| hoodie | 16.0 | $1.80 |
| dress | 20.0 | $2.20 |
| skirt | 12.0 | $1.30 |
| underwear | 5.0 | $0.55 |

CMT rate formula when calculated from SAM:
```
CMT_PER_UNIT = (SAM / 60) * line_cost_per_hour
Default line_cost_per_hour: $3.50 (Bangladesh), $5.00 (Vietnam), $4.00 (India), $7.00 (Turkey)
```

---

## 4. Priority Scoring

### 4.1 Auto-Priority Calculation

Priority is calculated on RFQ creation and updated when buyer or deadline changes:

```
base_score = 3 (neutral)

Buyer Tier Adjustment:
  tier 1 (top buyer):      +2
  tier 2:                   +1
  tier 3 (default):         0
  tier 4:                   -1
  tier 5 (low priority):    -1

Deadline Urgency Adjustment:
  deadline within 12 hours: +2
  deadline within 24 hours: +1
  deadline within 48 hours: 0
  deadline > 48 hours:      0
  no deadline specified:    0

Estimated Value Adjustment:
  estimated_value > $100,000: +1
  estimated_value > $50,000:  +0.5 (rounded)
  estimated_value < $5,000:   -1

priority = CLAMP(ROUND(base_score + tier_adj + deadline_adj + value_adj), 1, 5)
```

Priority 1 = highest urgency (red), 5 = lowest (gray).

### 4.2 Deadline Color Coding

| Time Remaining | Color | Label |
|---|---|---|
| > 24 hours | Green (#22C55E) | On Track |
| 12-24 hours | Amber (#F59E0B) | Due Soon |
| < 12 hours | Red (#EF4444) | Urgent |
| Overdue | Dark Red (#DC2626) | Overdue |
| No deadline | Gray (#9CA3AF) | No Deadline |

---

## 5. Extraction Confidence Rules

### 5.1 Confidence Thresholds

| Threshold | Action |
|---|---|
| >= 95% | Field auto-accepted, green indicator |
| 85-94% | Field auto-accepted, yellow indicator |
| 70-84% | Field flagged for review, amber highlight |
| < 70% | Field flagged as low-confidence, red highlight, blocks auto-costing |

### 5.2 Auto-Proceed to Costing

If ALL extracted fields have confidence >= 85%, the RFQ automatically transitions from `extracted` to `costing`. Otherwise, it waits for human review.

Required fields that must be present (non-null) to proceed:
- `product_type`
- `fabric_composition` OR `fabric_gsm`
- `total_quantity`

If any of these three are null regardless of confidence, extraction is considered incomplete and status stays at `extracted`.

### 5.3 Overall Confidence Average

```
extraction_confidence_avg = AVG(confidence of all non-null fields)
```

Fields with null values are excluded from the average.

### 5.4 Correction Learning

When a user overrides an extracted field:
1. Override is stored in `manual_overrides` with original value, new value, user_id, timestamp
2. Override count per field type is tracked per tenant
3. If a field type has > 10 overrides in the last 30 days, the extraction prompt is augmented with the 5 most recent corrections as few-shot examples for that tenant

---

## 6. Deduplication Logic

### 6.1 Dedup Hash Calculation

```
dedup_hash = SHA256(
  normalize(sender_email) + "|" +
  normalize(subject_or_reference) + "|" +
  normalize(buyer_name) + "|" +
  date_received_YYYYMMDD
)

normalize(str) = lowercase(trim(remove_re_fw_prefixes(str)))
```

### 6.2 Dedup Matching

On new RFQ creation:
1. Compute `dedup_hash`
2. Search existing RFQs in same tenant with matching `dedup_hash` created within last 7 days
3. If match found:
   - Set `merged_with_rfq_id` to the existing RFQ's ID
   - Add attachments to the existing RFQ
   - Notify user: "Duplicate RFQ detected — merged with {existing_rfq_number}"
   - New RFQ status set to `archived`
4. If no match: proceed normally

---

## 7. Auto-Routing Rules

### 7.1 Rule Evaluation

Rules are evaluated in `priority_order` (ascending). First matching rule wins.

```
For each active rule (ordered by priority_order ASC):
  If condition_type = "buyer" AND rfq.buyer_id = condition_value: MATCH
  If condition_type = "product_type" AND rfq.product_type ILIKE condition_value: MATCH
  If condition_type = "priority" AND rfq.priority <= INT(condition_value): MATCH
  If condition_type = "channel" AND rfq.source_channel = condition_value: MATCH

On MATCH:
  Set rfq.assigned_user_id = rule.assign_to_user_id
  Log assignment in audit_log
  Send notification to assigned user
  Stop evaluating further rules
```

### 7.2 Fallback

If no rules match, `assigned_user_id` remains null. RFQ appears in the shared inbox for all team members.

---

## 8. Follow-Up Logic

### 8.1 Auto Follow-Up Trigger

- Default follow-up window: 4 hours after quote sent
- Configurable per tenant (stored in tenant settings, not yet in schema — use default)
- Only triggers if:
  - quote.status = "sent"
  - quote.sent_at + follow_up_window < now()
  - No buyer response detected
  - Follow-up count for this quote < 3 (max 3 auto follow-ups)
  - Previous follow-up was > 24 hours ago (no spamming)

### 8.2 Follow-Up Escalation

| Follow-Up # | Timing | Action |
|---|---|---|
| 1st | 4 hours after sent | Polite follow-up email |
| 2nd | 48 hours after 1st follow-up | Firmer follow-up with deadline reminder |
| 3rd | 72 hours after 2nd follow-up | Final follow-up, cc factory owner |
| After 3rd | No more auto follow-ups | Notification to owner: "No response from {buyer} on {quote_number}" |

### 8.3 Template Variables

Available in follow-up templates:
```
{{buyer_name}}       - buyer.name
{{contact_name}}     - buyer.contact_name
{{factory_name}}     - tenant.name
{{product}}          - rfq.product_type
{{quantity}}         - rfq.total_quantity (formatted with commas)
{{quote_number}}     - quote.quote_number
{{fob_price}}        - quote.fob_per_unit (formatted to 2 decimals)
{{total_value}}      - quote.fob_total (formatted to 2 decimals)
{{currency}}         - quote.currency
{{deadline}}         - rfq.deadline (formatted as "March 15, 2026")
{{validity_date}}    - quote.valid_until (formatted)
{{sender_name}}      - current user full_name
{{sender_title}}     - current user role (display name)
```

---

## 9. RFQ Number Generation

```
Format: RFQ-YYYYMM-XXXX

YYYY = current year
MM   = current month (zero-padded)
XXXX = sequential counter per tenant per month, starting at 0001

Example: RFQ-202603-0017 (17th RFQ in March 2026)

Counter is stored as: MAX(rfq_number sequence for tenant in current month) + 1
Thread-safe via PostgreSQL sequence or SELECT FOR UPDATE.
```

---

## 10. Quote Number Generation

```
Format: QT-YYYYMM-XXXX

Same pattern as RFQ numbers but with QT prefix.
Counter is independent from RFQ counter.

Example: QT-202603-0042
```

---

## 11. Plan Enforcement Rules

### 11.1 RFQ Limits

```
On POST /api/v1/rfqs:
  current_month_count = COUNT(rfqs WHERE tenant_id = tenant AND created_at >= first_day_of_month)

  If current_month_count >= tenant.max_rfqs_per_month:
    Return 403: "Monthly RFQ limit reached ({limit}). Upgrade your plan for more."

  Limits by plan:
    starter: 20
    growth:  75
    scale:   unlimited (max_rfqs_per_month = 999999)
```

### 11.2 User Limits

```
On POST /api/v1/users/invite:
  current_user_count = COUNT(users WHERE tenant_id = tenant AND is_active = true)

  If current_user_count >= tenant.max_users:
    Return 403: "User limit reached ({limit}). Upgrade your plan."

  Limits by plan:
    starter: 2
    growth:  10
    scale:   unlimited (max_users = 999999)
```

### 11.3 Feature Gating

| Feature | Starter | Growth | Scale |
|---|---|---|---|
| Buyer portal integration | blocked | allowed | allowed |
| WhatsApp integration | blocked | allowed | allowed |
| Auto-routing rules | blocked | allowed | allowed |
| Scenario comparison | blocked | allowed | allowed |
| Excel quote export | blocked | allowed | allowed |
| Buyer scorecard analytics | blocked | allowed | allowed |
| Pipeline analytics | blocked | allowed | allowed |
| Cost trend analytics | blocked | allowed | allowed |
| Scheduled reports | blocked | allowed | allowed |
| Bulk cost library update | blocked | allowed | allowed |
| API access | blocked | blocked | allowed |
| Multi-factory | blocked | blocked | allowed |
| Custom dashboards | blocked | blocked | allowed |
| SSO/SAML | blocked | blocked | allowed |
| Win probability AI | blocked | blocked | allowed |
| Competitive benchmark | blocked | blocked | allowed |
| QUINN automations | blocked | blocked | allowed |

Feature gate check: `if (required_plan_level > tenant.plan) return 403`

Plan hierarchy: starter < growth < scale

---

## 12. Analytics Calculations

### 12.1 Win Rate

```
win_rate = (COUNT(rfqs WHERE status = 'won') / COUNT(rfqs WHERE status IN ('won', 'lost'))) * 100

Exclude 'expired' from win rate denominator (expired != lost).
If denominator = 0, win_rate = null (not 0).
```

### 12.2 Average Response Time

```
avg_response_time = AVG(quote.sent_at - rfq.created_at) in hours

Only includes RFQs that have at least one sent quote.
Rounded to 1 decimal place.
```

### 12.3 Pipeline Value

```
pipeline_value = SUM(quote.fob_total)
  WHERE quote.status IN ('draft', 'pending_review', 'approved', 'sent')
  AND quote is the latest version per rfq

Only count the latest version of each quote to avoid double-counting.
```

### 12.4 Revenue Forecast

```
revenue_forecast = pipeline_value * (win_rate / 100)

If win_rate is null (no historical data), use 30% as default assumption.
```

### 12.5 Missed RFQs

```
missed_rfqs = COUNT(rfqs WHERE
  (status IN ('new', 'extracting', 'extracted') AND deadline < now())
  OR
  (status = 'sent' AND sent_at > deadline)
)
```

### 12.6 Margin Erosion Alert

```
Triggers when:
  current_month_avg_margin < previous_month_avg_margin - 2.0 (percentage points)

current_month_avg_margin = AVG(quote.margin_percentage) WHERE quote.status IN ('sent', 'accepted') AND created_at in current month
previous_month_avg_margin = same for previous month
```

### 12.7 Buyer Volume Trend

```
volume_trend = "up" if current_period_rfqs > previous_period_rfqs * 1.1
volume_trend = "down" if current_period_rfqs < previous_period_rfqs * 0.9
volume_trend = "stable" otherwise

period = last 3 months vs previous 3 months
volume_change_percentage = ((current - previous) / previous) * 100
```

---

## 13. Notification Rules

### 13.1 Notification Triggers

| Event | Notification Type | Recipients | Channels |
|---|---|---|---|
| New RFQ detected | `new_rfq` | All merchandisers + quotation managers | Email + In-App |
| RFQ assigned to user | `assignment` | Assigned user | Email + In-App |
| Extraction complete | `extraction_complete` | Assigned user | In-App |
| Fields flagged for review | `review_needed` | Assigned user + quotation managers | Email + In-App |
| Quote ready for review | `quote_review` | Quotation managers + owner | Email + In-App |
| Quote approved | `quote_approved` | Assigned user + commercial executives | In-App |
| Quote opened by buyer | `quote_opened` | Assigned user | In-App |
| Buyer responded | `buyer_response` | Assigned user + quotation manager | Email + In-App |
| Deadline within 24h | `deadline_approaching` | Assigned user + owner | Email + In-App |
| Deadline within 12h | `deadline_urgent` | Assigned user + owner | Email + In-App |
| RFQ overdue | `deadline_overdue` | Owner | Email + In-App |
| Auto follow-up sent | `follow_up_sent` | Assigned user | In-App |
| No response after 3 follow-ups | `no_response_escalation` | Owner | Email + In-App |
| Monthly RFQ limit 80% used | `limit_approaching` | Owner + admin | Email + In-App |
| Monthly RFQ limit reached | `limit_reached` | Owner + admin | Email + In-App |
| @mention in notes | `mention` | Mentioned user | Email + In-App |

### 13.2 Notification Suppression

- Respect user's `notification_preferences` (email on/off, in_app on/off)
- Do not send email notification if user was active in-app within the last 5 minutes (they'll see the in-app notification)
- Batch email notifications: if > 3 notifications generated within 10 minutes for same user, send single digest email

---

## 14. Currency Conversion

### 14.1 Supported Currencies

| Code | Name | Symbol |
|---|---|---|
| USD | US Dollar | $ |
| BDT | Bangladeshi Taka | ৳ |
| EUR | Euro | EUR |
| GBP | British Pound | GBP |
| VND | Vietnamese Dong | d |
| INR | Indian Rupee | Rs |

### 14.2 Conversion Rules

```
If exchange_rate_locked = true on quote:
  Use quote.exchange_rate_to_usd (frozen at time of lock)

If exchange_rate_locked = false:
  Use latest rate from exchange_rates table
  If no rate exists: block conversion, prompt user to enter rate manually

Display format:
  USD: $1,234.56
  BDT: ৳123,456.00
  EUR: EUR1,234.56
  All amounts: 2 decimal places for display, 4 decimal places for per-unit calculations
```

---

## 15. Audit Trail Rules

### 15.1 Audited Actions

Every state change and data modification is logged:

```
rfq.created, rfq.updated, rfq.status_changed, rfq.assigned, rfq.extracted, rfq.override_applied, rfq.archived
quote.created, quote.updated, quote.approved, quote.sent, quote.revised, quote.status_changed
cost_library.created, cost_library.updated, cost_library.deactivated, cost_library.imported, cost_library.bulk_updated
buyer.created, buyer.updated
user.invited, user.updated, user.deactivated
tenant.updated
inbox_connection.created, inbox_connection.removed
routing_rule.created, routing_rule.updated, routing_rule.deleted
```

### 15.2 Change Tracking

For updates, the `changes` field stores:
```json
{
  "margin_percentage": { "old": 15.00, "new": 18.00 },
  "status": { "old": "review", "new": "approved" }
}
```

---

## 16. Validation Rules

### 16.1 RFQ Validation

| Field | Rule |
|---|---|
| priority | Integer 1-5 |
| deadline | Must be in the future (when manually set) |
| total_quantity | Positive integer, max 10,000,000 |
| won_lost_reason | Required when status changes to "won" or "lost", min 3 characters |
| buyer_id | Must reference an active buyer in same tenant |
| assigned_user_id | Must reference an active user in same tenant |

### 16.2 Quote Validation

| Field | Rule |
|---|---|
| margin_percentage | 0.00 to 99.99 |
| fob_per_unit | Must be > 0 |
| total_quantity | Must match linked RFQ quantity |
| incoterm | Must be one of: FOB, CIF, EXW, DDP |
| validity_days | 1 to 365 |
| currency | Must be one of: USD, BDT, EUR, GBP, VND, INR |

### 16.3 Cost Library Validation

| Field | Rule |
|---|---|
| cost | > 0, max 999999.9999 |
| fabric_gsm | 50 to 1000 |
| sam_minutes | 1.0 to 120.0 |
| moq_min | >= 0 |
| moq_max | > moq_min |
| overhead_percentage (tenant) | 0 to 50 |
| commission_percentage | 0 to 30 |
| finance_cost_percentage | 0 to 20 |

### 16.4 User Validation

| Field | Rule |
|---|---|
| email | Valid email format, unique within tenant |
| full_name | 2-255 characters |
| password | Min 8 characters, at least 1 uppercase, 1 lowercase, 1 number |
| role | Must be one of the 6 defined roles |

### 16.5 Buyer Validation

| Field | Rule |
|---|---|
| name | Required, 2-255 characters |
| code | Unique within tenant, alphanumeric + hyphens, max 50 chars |
| tier | Integer 1-5 |
| commission_percentage | 0.00 to 30.00 |
| contact_email | Valid email format if provided |

---

*This document is confidential and intended for internal use only.*
*fabricXai is a product of SocioFi Technology Ltd.*
