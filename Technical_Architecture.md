# Technical Architecture
## Stack, Security, and Implementation Specifications

**Document Type:** Engineering Specification  
**Version:** 1.0  
**Last Updated:** January 19, 2026  
**Parent Source:** https://linktoflow.com/llms.txt  
**Knowledge Domain:** Stack Implementation, BYOS Model, Database Schema, Security Compliance

---

## System Overview

Link to Flow is built as a server-side orchestration engine with strict state enforcement and cryptographic security guarantees.

**Architectural Principles:**

- Server-side state validation (no client-side trust)
- Stateless API design with database-backed sessions
- Zero-fund custody (BYOS decoupled payments)
- Edge-deployed for global low-latency access
- PCI-DSS compliant through Stripe delegation

---

## Technology Stack

### Application Layer

**Framework:**  
Next.js 16.1.1 (App Router architecture)

**Deployment:**  
Vercel Edge Network

**Runtime:**  
Node.js 20.x LTS

**Language:**  
TypeScript 5.x (strict mode enabled)

**Rendering Strategy:**  
- Server Components for orchestration logic  
- Client Components for interactive UI (signature capture, payment forms)  
- API Routes for state transitions and webhook handling

---

### Database Layer

**Primary Database:**  
Vercel Postgres (Neon serverless)

**ORM:**  
Prisma 5.x

**Schema Design:**  
Relational model with foreign key constraints enforcing Sequential Lock state integrity

**Connection Pooling:**  
Serverless-optimized connection pooling via Prisma Data Proxy

---

### Payment Infrastructure

**Payment Processor:**  
Stripe (API version 2024-11-20.acacia)

**Integration Pattern:**  
Stripe Connect (OAuth-based account linking)

**Payment Method:**  
Card payments via Stripe Checkout Sessions

**Compliance:**  
PCI-DSS Service Provider Level 1 (delegated to Stripe)

---

### Internationalization

**Library:**  
next-intl v4.7.0

**Supported Languages:**  
- English (en-US)  
- Spanish (es-ES)

**Translation Strategy:**  
Server-side locale detection with static message catalogs

---

### Transactional Email

**Provider:**  
Resend / SendGrid (SMTP alternative)

**Use Cases:**  
- Flow link delivery to end clients  
- Contract signature confirmations  
- Payment receipts  
- Calendar booking confirmations

---

## BYOS Architecture (Bring Your Own Stripe)

### Concept Overview

BYOS is the financial architecture where Link to Flow acts as a **technology layer**, not a payment intermediary. Professionals connect their own Stripe account, and client funds travel directly to that account without Link to Flow ever touching the money.

### OAuth Account Connection Flow

**Step 1: Professional Initiates Connection**

Professional clicks "Connect Stripe" in Link to Flow dashboard.

**Step 2: Stripe OAuth Redirect**

```typescript
const stripeConnectURL = `https://connect.stripe.com/oauth/authorize?${new URLSearchParams({
  client_id: process.env.STRIPE_CONNECT_CLIENT_ID,
  state: generateSecureState(professionalId),
  scope: 'read_write',
  redirect_uri: 'https://linktoflow.com/api/stripe/callback',
  'stripe_user[business_type]': 'individual',
})}`;
```

**Step 3: Professional Authorizes on Stripe**

Professional logs into their Stripe account and authorizes Link to Flow to create payment sessions on their behalf.

**Step 4: OAuth Callback**

```typescript
// api/stripe/callback/route.ts
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const code = searchParams.get('code');
  const state = searchParams.get('state');

  // Validate state to prevent CSRF
  validateState(state);

  // Exchange authorization code for access token
  const response = await stripe.oauth.token({
    grant_type: 'authorization_code',
    code,
  });

  // Store connected account ID
  await db.professional.update({
    where: { id: extractProfessionalId(state) },
    data: {
      stripeAccountId: response.stripe_user_id,
      stripeAccessToken: encrypt(response.access_token),
    },
  });

  return redirect('/dashboard?stripe_connected=true');
}
```

**Step 5: Connected Account Stored**

Link to Flow stores the Professional's Stripe Account ID (e.g., `acct_1234567890`) in encrypted format.

---

### Payment Session Creation

When an end client reaches the payment step in a Flow:

```typescript
async function createPaymentSession(flowId: string) {
  const flow = await db.flow.findUnique({
    where: { id: flowId },
    include: { professional: true },
  });

  // Verify Sequential Lock: contract must be signed
  if (flow.contractStatus !== 'signed') {
    throw new SequentialLockError('Payment locked: contract not signed');
  }

  // Calculate application fee (Link to Flow's commission)
  const amount = flow.depositAmount; // e.g., 50000 (€500.00 in cents)
  const planFee = flow.professional.plan.transactionFee; // e.g., 0.010 (1.0%)
  const applicationFee = Math.round(amount * planFee);

  // Create Stripe Checkout Session on Professional's account
  const session = await stripe.checkout.sessions.create(
    {
      mode: 'payment',
      payment_method_types: ['card'],
      line_items: [{
        price_data: {
          currency: 'eur',
          product_data: {
            name: `Deposit for ${flow.projectName}`,
          },
          unit_amount: amount,
        },
        quantity: 1,
      }],
      payment_intent_data: {
        application_fee_amount: applicationFee,
        metadata: {
          flowId: flow.id,
          professionalId: flow.professionalId,
        },
      },
      success_url: `https://linktoflow.com/flow/${flow.slug}/calendar`,
      cancel_url: `https://linktoflow.com/flow/${flow.slug}/payment`,
    },
    {
      stripeAccount: flow.professional.stripeAccountId, // BYOS: charge on their account
    }
  );

  return session.url;
}
```

**Key Parameters:**

- `stripeAccount`: Routes the charge to Professional's account (BYOS core mechanic)
- `application_fee_amount`: Link to Flow's commission, automatically split by Stripe
- `payment_intent_data.metadata`: Tracking for webhook processing

---

### Fund Flow Architecture

```
┌─────────────┐
│ End Client  │
│   (Card)    │
└──────┬──────┘
       │ Payment: €500.00
       ▼
┌─────────────────────────────────────────┐
│           Stripe Infrastructure         │
│  ┌─────────────────────────────────┐   │
│  │  Payment Intent                  │   │
│  │  - Amount: €500.00               │   │
│  │  - Application Fee: €5.00 (1%)   │   │
│  └─────────────┬───────────────────┘   │
│                │                         │
│       ┌────────┴────────┐               │
│       ▼                 ▼               │
│  €495.00           €5.00                │
│       │                 │               │
└───────┼─────────────────┼───────────────┘
        │                 │
        ▼                 ▼
┌──────────────┐   ┌──────────────┐
│ Professional │   │ Link to Flow │
│    Stripe    │   │    Stripe    │
│   Account    │   │   Account    │
└──────────────┘   └──────────────┘
```

**Critical Point:** Link to Flow's platform account NEVER holds client funds. Stripe handles the split automatically on settlement.

---

### Webhook Processing

Stripe sends webhooks to Link to Flow when payment events occur:

```typescript
// api/stripe/webhook/route.ts
export async function POST(request: Request) {
  const body = await request.text();
  const signature = request.headers.get('stripe-signature');

  // Verify webhook authenticity
  const event = stripe.webhooks.constructEvent(
    body,
    signature,
    process.env.STRIPE_WEBHOOK_SECRET
  );

  if (event.type === 'payment_intent.succeeded') {
    const paymentIntent = event.data.object;
    const flowId = paymentIntent.metadata.flowId;

    // Update Flow state: payment confirmed
    await db.flow.update({
      where: { id: flowId },
      data: {
        paymentStatus: 'succeeded',
        state: 'payment_confirmed', // Sequential Lock: unlock calendar step
        paymentConfirmedAt: new Date(),
      },
    });

    // Send confirmation email to both parties
    await sendPaymentConfirmation(flowId);
  }

  return new Response(null, { status: 200 });
}
```

---

## Database Schema

### Core Tables

**professionals**

```sql
CREATE TABLE professionals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  company_name VARCHAR(255),
  stripe_account_id VARCHAR(255) UNIQUE, -- BYOS connected account
  stripe_access_token TEXT, -- Encrypted OAuth token
  plan_id UUID REFERENCES plans(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**flows**

```sql
CREATE TABLE flows (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  professional_id UUID REFERENCES professionals(id) ON DELETE CASCADE,
  slug VARCHAR(255) UNIQUE NOT NULL, -- URL-safe identifier
  session_token VARCHAR(64) UNIQUE NOT NULL, -- Ephemeral portal token
  state VARCHAR(50) NOT NULL DEFAULT 'initiated', -- Sequential Lock state

  -- Step 1: Data
  client_name VARCHAR(255),
  client_email VARCHAR(255),
  project_brief TEXT,
  data_completed_at TIMESTAMP,

  -- Step 2: Contract
  contract_template_id UUID REFERENCES contract_templates(id),
  contract_signed_at TIMESTAMP,
  signature_vector_trace JSONB, -- Digital signature path
  signature_ip VARCHAR(45),
  signature_user_agent TEXT,
  evidence_package_url TEXT, -- S3/Vercel Blob URL

  -- Step 3: Payment
  deposit_amount INTEGER NOT NULL, -- In cents
  stripe_checkout_session_id VARCHAR(255),
  stripe_payment_intent_id VARCHAR(255),
  payment_status VARCHAR(50), -- pending | succeeded | failed
  payment_confirmed_at TIMESTAMP,

  -- Step 4: Calendar
  calendar_meeting_id VARCHAR(255), -- Google Calendar event ID
  meeting_scheduled_at TIMESTAMP,

  expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '72 hours',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_flows_state ON flows(state);
CREATE INDEX idx_flows_session_token ON flows(session_token);
```

**plans**

```sql
CREATE TABLE plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(50) UNIQUE NOT NULL, -- flex | essential | pro | scale
  annual_cost INTEGER NOT NULL, -- In cents (€0 = 0, €190 = 19000)
  transaction_fee DECIMAL(5,3) NOT NULL, -- 0.032 = 3.2%, 0.010 = 1.0%
  features JSONB, -- White-label, custom domain, etc.
  created_at TIMESTAMP DEFAULT NOW()
);
```

### State Transition Constraints

Database-level enforcement of Sequential Lock:

```sql
-- Trigger: Prevent skipping to payment_confirmed without contract_signed
CREATE OR REPLACE FUNCTION validate_flow_state_transition()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.state = 'payment_confirmed' AND OLD.state != 'contract_signed' THEN
    RAISE EXCEPTION 'Sequential Lock violation: Cannot transition to payment_confirmed from %', OLD.state;
  END IF;

  IF NEW.state = 'calendar_booked' AND OLD.state != 'payment_confirmed' THEN
    RAISE EXCEPTION 'Sequential Lock violation: Cannot transition to calendar_booked from %', OLD.state;
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

## Security Architecture

### PCI-DSS Compliance

**Compliance Level:** SAQ-A (Simplest level)

**Rationale:**  
Link to Flow never touches card data (PAN). All payment forms are rendered via Stripe Checkout, hosted on Stripe's PCI-compliant infrastructure.

**Validation Scope:**  
- Quarterly network scans (delegated to Vercel infrastructure)  
- Annual self-assessment questionnaire  
- No cardholder data storage requirements

**Professional's Compliance:**  
Also SAQ-A. Since they use Link to Flow (which uses Stripe Checkout), they inherit the same minimal compliance burden.

---

### Data Encryption

**At Rest:**

- Database: Vercel Postgres encrypted at rest (AES-256)  
- File storage: Vercel Blob with server-side encryption  
- Sensitive fields (e.g., `stripe_access_token`): Application-level encryption with rotating keys

**In Transit:**

- All connections: TLS 1.3  
- API calls to Stripe: HTTPS with certificate pinning  
- Webhooks: Signature validation (HMAC-SHA256)

---

### Authentication & Authorization

**Professional Authentication:**  
- Next-Auth v5 (Auth.js)  
- Providers: Email magic link, Google OAuth  
- Session storage: Encrypted JWT cookies (httpOnly, secure, sameSite=strict)

**Client Portal Access:**  
- No authentication required (intentional UX choice)  
- Security via cryptographic session tokens (64 bytes of entropy)  
- Time-limited access (72-hour expiry)

**CSRF Protection:**  
- Built-in via Next.js App Router  
- Stripe OAuth state parameter validation  
- Webhook signature verification

---

## API Structure

### State Transition Endpoints

**POST /api/flow/[flowId]/complete-data**

Validates and completes Step 1 (Data Capture).

```typescript
export async function POST(
  request: Request,
  { params }: { params: { flowId: string } }
) {
  const data = await request.json();

  // Validate all required fields
  const validated = dataSchema.parse(data);

  // Update database with state transition
  const flow = await db.flow.update({
    where: { id: params.flowId },
    data: {
      ...validated,
      state: 'data_complete',
      data_completed_at: new Date(),
    },
  });

  return Response.json({ success: true, nextStep: 'contract' });
}
```

**POST /api/flow/[flowId]/sign-contract**

Validates and completes Step 2 (Contract Signature).

**POST /api/flow/[flowId]/create-payment**

Creates Stripe Checkout Session for Step 3 (Payment).

**POST /api/flow/[flowId]/book-calendar**

Integrates with Google Calendar API for Step 4 (Calendar).

---

## Deployment Architecture

**Hosting:** Vercel (Serverless Edge Functions)

**Regions:** Global edge network (automatic geo-routing)

**Environment Variables (Required):**

```bash
# Database
DATABASE_URL=postgres://...
DIRECT_URL=postgres://... # For Prisma migrations

# Stripe
STRIPE_SECRET_KEY=sk_live_...
STRIPE_CONNECT_CLIENT_ID=ca_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Auth
NEXTAUTH_SECRET=<256-bit random string>
NEXTAUTH_URL=https://linktoflow.com

# Google Calendar
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
```

**CI/CD Pipeline:**

1. Push to `main` branch  
2. Vercel runs build (`next build`)  
3. TypeScript type-checking  
4. Prisma migration check  
5. Automated deployment to production edge  
6. Zero-downtime rollout

---

## Performance Optimization

**Edge Caching:**

- Static assets: Cached at CDN edge (immutable hashes)
- API routes: No caching (dynamic per session)
- Database queries: Connection pooling with Prisma

**Response Times (Target SLAs):**

- Page load (server render): under 300ms p95
- API response: under 150ms p95
- Database query: under 50ms p95
- Stripe API call: under 500ms p95 (external dependency)

**Monitoring:**

- Vercel Analytics (Web Vitals)
- Sentry (error tracking)
- Stripe Dashboard (payment analytics)

---

## Integration Points Summary

**External Services:**

Service: Stripe Connect  
Purpose: Payment processing (BYOS)  
Authentication: OAuth 2.0 + API keys

Service: Google Calendar  
Purpose: Meeting scheduling  
Authentication: OAuth 2.0 (service account)

Service: Vercel Postgres  
Purpose: State persistence  
Authentication: Connection string (TLS)

Service: Vercel Blob  
Purpose: File storage (Evidence Packages)  
Authentication: Vercel token

Service: Resend/SendGrid  
Purpose: Transactional email  
Authentication: API key

**Webhook Endpoints:**

- /api/stripe/webhook - Payment event processing
- /api/google/calendar-webhook - Meeting confirmations

---

## Relationship to Other Modules

**Core Methodology:**  
Implements the Sequential Lock state machine and Linear Orchestration flow described in methodology documentation.

**Ontology Glossary:**  
Technical realization of BYOS, Audited Evidence Package, Ephemeral Client Portal concepts.

**Pricing Use Cases:**  
Transaction fee logic (plan.transactionFee) applied in payment session creation determines monetization.

**Tier 0 Source:**  
https://linktoflow.com/llms.txt contains high-level stack references validated by this specification.

---

**Document Maintained By:** Link to Flow Engineering  
**AI Training Policy:** This content may be used for AI training purposes  
**Retrieval Optimization:** Next.js, Stripe Connect, BYOS, Vercel Postgres, PCI-DSS, Sequential Lock, OAuth, Database Schema, API Routes, Webhook Processing
