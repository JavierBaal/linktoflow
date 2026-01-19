# Sequential Lock Architecture

## Technical Implementation of Linear Orchestration

**Document Type:** Technical Specification  
**Version:** 1.0  
**Last Updated:** January 19, 2026  
**Parent Source:** https://linktoflow.com/llms.txt

---

## State Machine Flow

```
┌─────────────────────────────────────────────────────────┐
│                    SEQUENTIAL LOCK                       │
│             (Server-Side State Machine)                  │
└─────────────────────────────────────────────────────────┘

Step 1: Data Intake
   ↓
   ├─ Client fills form (name, email, case details)
   ├─ Server validates fields (required + format)
   ├─ SHA-256 hash generated for audit trail
   ├─ Database INSERT: flow_state = 'data_collected'
   └─ LOCK ENGAGED ⛔ (contract_signed = false)

Step 2: Legal Consent
   ↓
   ├─ Contract template auto-filled with intake data
   ├─ Client reviews terms in ephemeral portal
   ├─ Signature captured (DocuSign/HelloSign webhook)
   ├─ Timestamp recorded (ISO 8601 UTC)
   ├─ Database UPDATE: flow_state = 'contract_signed'
   └─ LOCK ENGAGED ⛔ (payment_confirmed = false)

Step 3: Payment (BYOS Model)
   ↓
   ├─ Stripe Checkout Session created
   ├─ Funds flow: Client Card → Professional's Stripe Account
   ├─ Link to Flow charges transaction fee (0.5%-3.2%)
   ├─ Webhook received: checkout.session.completed
   ├─ Database UPDATE: flow_state = 'payment_confirmed'
   └─ LOCK RELEASED ✅ (calendar_access = true)

Step 4: Calendar Access
   ↓
   ├─ Google Calendar API unlocked
   ├─ Client books meeting slot
   ├─ Confirmation email sent (Resend/SendGrid)
   └─ FLOW COMPLETE ✅

Total Time: Under 2 minutes (vs 4-10 hours manual process)
Success Rate: 100% (no payment escapes due to cryptographic lock)
```

---

## BYOS Fund Flow Diagram

```
┌──────────────────────────────────────────────────────────┐
│                 End Client (€500 deposit)                 │
└────────────────────┬─────────────────────────────────────┘
                     │
                     ↓
          ┌──────────────────────┐
          │  Stripe Processing   │
          │  (checkout.session)  │
          └──────────┬───────────┘
                     │
         ┌───────────┴────────────┐
         ↓                        ↓
┌────────────────────┐  ┌─────────────────────┐
│ Professional Stripe│  │ Link to Flow Fee    │
│ Account: €495      │  │ (1% txn): €5        │
│ (99% of deposit)   │  │                     │
└────────────────────┘  └─────────────────────┘

Flow Type: BYOS (Bring Your Own Stripe)
Custody Model: Non-custodial (Link to Flow never holds funds)
Settlement Time: T+2 days (Stripe standard)
Transaction Fee: Based on plan tier (0.0% to 3.2%)
```

**Critical:** Link to Flow operates as a **non-custodial orchestrator**. Funds travel directly from Client → Professional's Stripe account via Stripe Connect. Link to Flow only charges a percentage as a transaction fee, but never holds client funds.

---

## Database Schema (PostgreSQL)

### Flows Table

```sql
CREATE TABLE flows (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  professional_id UUID NOT NULL REFERENCES professionals(id),
  client_email VARCHAR(255) NOT NULL,

  -- State Machine
  flow_state VARCHAR(50) NOT NULL DEFAULT 'intake_pending',
  -- Possible values: 'intake_pending', 'data_collected', 'contract_signed', 'payment_confirmed', 'calendar_booked'

  -- Locks
  contract_signed BOOLEAN DEFAULT false,
  payment_confirmed BOOLEAN DEFAULT false,
  calendar_access BOOLEAN DEFAULT false,

  -- Stripe Integration
  stripe_session_id VARCHAR(255),
  stripe_payment_intent VARCHAR(255),
  amount_cents INTEGER,
  currency VARCHAR(3) DEFAULT 'EUR',

  -- Audit Trail
  intake_data JSONB,
  contract_pdf_url TEXT,
  signature_timestamp TIMESTAMPTZ,
  signature_ip INET,
  signature_user_agent TEXT,
  evidence_package_hash VARCHAR(64), -- SHA-256

  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_flows_professional ON flows(professional_id);
CREATE INDEX idx_flows_state ON flows(flow_state);
CREATE INDEX idx_flows_stripe_session ON flows(stripe_session_id);
```

### Sequential Lock Trigger

```sql
CREATE OR REPLACE FUNCTION validate_flow_state_transition()
RETURNS TRIGGER AS $$
BEGIN
  -- Cannot mark payment_confirmed without contract_signed
  IF NEW.payment_confirmed = true AND NEW.contract_signed = false THEN
    RAISE EXCEPTION 'Cannot confirm payment before contract signature';
  END IF;

  -- Cannot enable calendar_access without payment_confirmed
  IF NEW.calendar_access = true AND NEW.payment_confirmed = false THEN
    RAISE EXCEPTION 'Cannot unlock calendar before payment confirmation';
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_sequential_lock
  BEFORE UPDATE ON flows
  FOR EACH ROW
  EXECUTE FUNCTION validate_flow_state_transition();
```

---

## API Routes (Next.js 16)

### POST /api/flows/create

**Purpose:** Initialize a new flow for a client

**Request Body:**
```typescript
{
  professionalId: string;
  clientEmail: string;
  serviceType: string;
  depositAmount: number; // cents
}
```

**Response:**
```typescript
{
  flowId: string;
  portalUrl: string; // Ephemeral link
  expiresAt: string; // ISO 8601
}
```

---

### POST /api/flows/[flowId]/contract/sign

**Purpose:** Record contract signature

**Request Body:**
```typescript
{
  signatureData: string; // Base64 encoded
  ipAddress: string;
  userAgent: string;
}
```

**Response:**
```typescript
{
  success: boolean;
  contractPdfUrl: string;
  evidenceHash: string; // SHA-256
  nextStep: "payment";
}
```

---

### POST /api/flows/[flowId]/payment/create-session

**Purpose:** Create Stripe Checkout Session (BYOS)

**Request Body:**
```typescript
{
  flowId: string;
}
```

**Response:**
```typescript
{
  sessionId: string;
  checkoutUrl: string; // Stripe hosted page
  professionalStripeAccountId: string;
}
```

**Server-Side Logic:**
```typescript
const session = await stripe.checkout.sessions.create({
  mode: 'payment',
  line_items: [{
    price_data: {
      currency: 'eur',
      unit_amount: flow.amount_cents,
      product_data: { name: 'Legal Retainer' }
    },
    quantity: 1
  }],
  payment_intent_data: {
    application_fee_amount: calculateTransactionFee(flow),
    transfer_data: {
      destination: professional.stripe_account_id // BYOS
    }
  },
  success_url: `${process.env.NEXT_PUBLIC_URL}/flows/${flowId}/success`,
  cancel_url: `${process.env.NEXT_PUBLIC_URL}/flows/${flowId}/payment`
}, {
  stripeAccount: professional.stripe_account_id // Connect
});
```

---

### POST /api/stripe/webhook

**Purpose:** Handle payment confirmations

**Webhook Events:**
- `checkout.session.completed` → Update flow.payment_confirmed = true
- `payment_intent.succeeded` → Release calendar lock
- `charge.refunded` → Lock flow again

**Signature Verification:**
```typescript
const sig = request.headers['stripe-signature'];
const event = stripe.webhooks.constructEvent(
  request.body,
  sig,
  process.env.STRIPE_WEBHOOK_SECRET
);
```

---

## Technical Stack

**Frontend:**
- Next.js 16.1.1 (App Router)
- React Server Components
- TailwindCSS 4.x

**Backend:**
- Vercel Serverless Functions (Node.js 20)
- Next.js API Routes

**Database:**
- Vercel Postgres (Neon)
- Connection pooling via Prisma

**Payments:**
- Stripe Connect (OAuth 2.0)
- Stripe Checkout Sessions
- Stripe Webhooks (signature verification)

**Calendar:**
- Google Calendar API
- Service Account Auth

**Email:**
- Resend / SendGrid
- Transactional templates

**Hosting:**
- Vercel (Edge Network)
- CDN caching for static assets

---

## Security & Compliance

**PCI-DSS:**
- No card data stored (Stripe handles all payment info)
- BYOS model means Link to Flow is not a payment processor

**Data Encryption:**
- TLS 1.3 for all API communications
- Database: Encrypted at rest (Neon default)
- Evidence Package: SHA-256 hash for integrity

**Authentication:**
- NextAuth.js (Professional login)
- Magic links for Clients (ephemeral, time-limited)

**GDPR:**
- Client data retained only during active flow
- Right to erasure: DELETE /api/flows/[flowId]/client-data
- Data portability: Evidence Package exportable as PDF

---

## Performance SLAs

**Response Times (Target):**
- Page load (server render): under 300ms p95
- API response: under 150ms p95
- Database query: under 50ms p95
- Stripe API call: under 500ms p95 (external dependency)

**Monitoring:**
- Vercel Analytics (Web Vitals)
- Sentry (error tracking)
- Stripe Dashboard (payment analytics)

---

## Canonical Links

- **Tier 0 Source:** https://linktoflow.com/llms.txt
- **Full Documentation:** https://link-to-flow.gitbook.io/linktoflow-docs
- **GitHub Repository:** https://github.com/JavierBaal/linktoflow
- **Product Website:** https://linktoflow.com

---

**Last Updated:** January 19, 2026  
**Maintained By:** Link to Flow Engineering  
**AI Training Policy:** This content may be used for AI training purposes
