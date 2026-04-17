# Flomax Platform — API Specification & Workflow
Version 1.0 | 2026-04-16

---

## Base URL
```
https://api.flomax.io/v1
```

## Authentication
All protected routes require:
```
Authorization: Bearer <access_token>
```
Access token: JWT, expires in 15 minutes.
Refresh token: httpOnly cookie, expires in 30 days.

Role guard shorthand used below:
- `[A]` = admin only
- `[O]` = owner only
- `[M]` = marketer only
- `[B]` = buyer only
- `[A,O]` = admin or owner

---

## 1. Auth Endpoints

### POST /auth/register
Register a new user.
```json
Request:
{
  "role": "owner | marketer | buyer",
  "full_name": "Ahmed Hassan",
  "email": "ahmed@example.com",
  "phone": "+201001234567",
  "password": "StrongPass123!",
  "national_id": "29901011234567"   // optional, required for marketers
}

Response 201:
{
  "user": { "id": "uuid", "role": "owner", "status": "pending", "email": "..." },
  "message": "Account created. Awaiting admin verification."
}
```

### POST /auth/login
```json
Request:
{ "email": "ahmed@example.com", "password": "StrongPass123!" }

Response 200:
{
  "access_token": "eyJ...",
  "expires_in": 900,
  "user": { "id": "uuid", "role": "owner", "full_name": "Ahmed Hassan" }
}
// refresh_token set as httpOnly cookie
```

### POST /auth/refresh
Uses httpOnly refresh_token cookie. Returns new access_token.

### POST /auth/logout
Revokes the current refresh token.

---

## 2. Properties

### POST /properties  [O]
Owner submits a new property.
```json
Request:
{
  "title": "شقة 3 غرف - المعادي",
  "type": "apartment",
  "city": "Cairo",
  "district": "Maadi",
  "area_sqm": 145,
  "bedrooms": 3,
  "bathrooms": 2,
  "asking_price": 4500000,
  "currency": "EGP",
  "commission_pct": 3.0,
  "description": "...",
  "media_urls": ["https://..."]
}

Response 201:
{ "property": { "id": "uuid", "status": "draft", ... } }
```

### GET /properties  [A]
List all properties with filters: `?status=active&city=Cairo&type=apartment&page=1&limit=20`

### GET /properties/:id  [A, O]
Full property details. Owner only sees their own.

### PUT /properties/:id  [O]
Update property info (only if status = draft or active).

### PATCH /properties/:id/status  [A]
```json
Request: { "status": "active" }
// Valid transitions:
// draft → active
// active → withdrawn
// assigned → under_offer
// under_offer → sold | active
```

### GET /owners/:id/properties  [O, A]
Owner's property list with status summary.

---

## 3. Property Codes (Isolation Layer)

### POST /properties/:id/code  [A]
Generate a public code for a property.
```json
Response 201:
{
  "code": "FLX-00342-CAI",
  "property_id": "uuid",
  "is_active": true,
  "activated_at": "2026-04-16T10:00:00Z"
}
```
Rule: Only one active code per property at a time.

### GET /codes/:code  [M, A]
Marketer fetches property info by code. Returns property details WITHOUT owner identity.
```json
Response 200:
{
  "code": "FLX-00342-CAI",
  "property": {
    "title": "شقة 3 غرف - المعادي",
    "type": "apartment",
    "city": "Cairo",
    "area_sqm": 145,
    "asking_price": 4500000,
    "media_urls": [...]
    // ⚠ owner_id is NOT included
  }
}
```

### GET /codes  [A]
List all codes with status and assignment info.

### PATCH /codes/:code/deactivate  [A]
Deactivates the code (e.g. property sold or withdrawn).

---

## 4. Marketing Plans

### POST /properties/:id/marketing-plan  [A]
```json
Request:
{
  "title": "خطة إطلاق شقة المعادي",
  "allocated_budget": 15000,
  "channels": ["social_media", "website", "referral"],
  "start_date": "2026-04-20",
  "end_date": "2026-06-20",
  "notes": "Focus on Instagram and Facebook ads."
}
Response 201: { "plan": { "id": "uuid", "status": "draft", ... } }
```

### GET /properties/:id/marketing-plan  [A, O]
Get the active marketing plan. Owner sees budget summary only (not full details).

### PUT /marketing-plans/:id  [A]
Update plan details (title, budget, channels, dates).

### PATCH /marketing-plans/:id/status  [A]
```json
Request: { "status": "active" }
// Transitions: draft → active → paused → completed
```

---

## 5. Assignments

### POST /assignments  [A]
Assign a property code to a marketer.
```json
Request:
{
  "code_id": "uuid",
  "marketer_id": "uuid",
  "marketer_commission_pct": 1.5   // % of deal value going to marketer
}
Response 201:
{ "assignment": { "id": "uuid", "status": "pending", ... } }
```
Rules:
- Marketer must have status = active.
- Code must be active with no existing accepted assignment.

### GET /marketers/:id/assignments  [M, A]
```json
Response: {
  "assignments": [
    {
      "id": "uuid",
      "status": "accepted",
      "code": "FLX-00342-CAI",
      "property_summary": { "title": "...", "city": "...", "asking_price": 4500000 },
      "marketer_commission_pct": 1.5
    }
  ]
}
```

### PATCH /assignments/:id/accept  [M]
Marketer accepts the assignment.
```json
Response: { "assignment": { "status": "accepted", "accepted_at": "..." } }
```

### PATCH /assignments/:id/reject  [M]
```json
Request: { "reason": "Too far from my area" }
```

### GET /assignments  [A]
Full list with filters: `?status=pending&marketer_id=uuid&code=FLX-00342-CAI`

---

## 6. Deals

### POST /deals  [M]
Marketer submits a buyer lead.
```json
Request:
{
  "assignment_id": "uuid",
  "buyer_id": "uuid",           // buyer must be registered in the system
  "notes": "Buyer visited the unit. Very interested. Wants to negotiate price."
}
Response 201:
{ "deal": { "id": "uuid", "status": "lead", "lead_at": "..." } }
```

### GET /deals  [A]
All deals with filters: `?status=negotiation&marketer_id=uuid&page=1`

### GET /deals/:id  [A, M]
Full deal details. Marketer sees their own deals only.
```json
Response:
{
  "deal": {
    "id": "uuid",
    "status": "negotiation",
    "buyer": { "full_name": "Sara Ali", "phone": "+201..." },
    "property_code": "FLX-00342-CAI",
    "property": { "title": "...", "asking_price": 4500000 },
    "notes": [...],
    "timeline": {
      "lead_at": "...",
      "negotiation_at": "...",
      "contracted_at": null,
      "completed_at": null
    }
  }
}
```

### PATCH /deals/:id/status  [A]
Advance deal through lifecycle.
```json
Request:
{
  "status": "contracted",
  "agreed_price": 4200000     // required when moving to contracted
}
// State machine:
// lead → negotiation
// negotiation → contracted (requires agreed_price)
// contracted → completed  (triggers commission creation)
// any → cancelled          (requires cancellation_reason)
```
⚠ When status → completed, system auto-creates a commission record.

### GET /marketers/:id/deals  [M, A]
Marketer's deal history with summary stats.

### POST /deals/:id/notes  [A, M]
```json
Request: { "content": "Buyer confirmed final price. Signing next week." }
```

---

## 7. Commissions

### POST /deals/:id/commission  [A]
Manually finalize or override the commission split.
```json
Request:
{
  "platform_pct": 50,
  "marketer_pct": 40,
  "operational_pct": 10
}
// System auto-calculates:
// total_commission = agreed_price × commission_pct / 100
// platform_share   = total_commission × platform_pct / 100
// marketer_share   = total_commission × marketer_pct / 100
// operational_fund = total_commission × operational_pct / 100

Response 201:
{
  "commission": {
    "deal_id": "uuid",
    "deal_value": 4200000,
    "total_commission": 126000,
    "platform_share": 63000,
    "marketer_share": 50400,
    "operational_fund": 12600,
    "status": "pending"
  }
}
```

### GET /commissions/:id  [A, M]
Commission details. Marketer only sees their own share (not platform/operational split).

### PATCH /commissions/:id/pay  [A]
Mark marketer's share as paid.
```json
Request:
{
  "payment_reference": "BANK-TXN-20260416-001",
  "marketer_paid_status": "paid"
}
```

### GET /marketers/:id/commissions  [M, A]
Marketer earnings history.
```json
Response:
{
  "total_earned": 150400,
  "paid": 100000,
  "pending": 50400,
  "commissions": [...]
}
```

### GET /commissions/summary  [A]
Platform-wide revenue report.
```json
Response:
{
  "total_deals_completed": 12,
  "total_commissions": 1520000,
  "platform_revenue": 760000,
  "marketer_payouts": 608000,
  "operational_fund": 152000,
  "unpaid_to_marketers": 200000
}
```

---

## 8. Buyers

### GET /buyers/:id  [A, M]
Get buyer profile. Marketer can only view buyers linked to their own deals.

### GET /buyers/:id/deals  [B, A]
Buyer's deal history.

### GET /buyers  [A]
All registered buyers with filters: `?city=Cairo&verified=true`

---

## Deal State Machine

```
[lead] ──────────────→ [negotiation] ──────────────→ [contracted] ──→ [completed]
  │                          │                              │
  └──────────── [cancelled] ←┘                             └→ auto-create commission
```

## Key Business Rules

1. A marketer NEVER receives owner contact info — only property details via code.
2. An owner NEVER receives marketer contact info — platform handles all communication.
3. Commission is only triggered on deal status = completed.
4. One active assignment per code at any time.
5. A deal requires an accepted assignment before creation.
6. Commission split percentages must always sum to 100.
7. Only admin can advance deal status beyond 'lead'.
8. Marketer can submit lead and add notes, but cannot change deal status.

## Owner Workflow

The high‑level Owner workflow is documented in [Owner Workflow](owner_workflow.md).

