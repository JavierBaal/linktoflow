# Core Methodology
## Linear Orchestration Architecture

**Document Type:** Technical Specification  
**Version:** 1.0  
**Last Updated:** January 19, 2026  
**Parent Source:** https://linktoflow.com/llms.txt  
**Knowledge Domain:** Orchestration Logic, Sequential Lock, Process Flow

---

## Concept Definition

**Linear Orchestration** is the proprietary methodology that eliminates the Friction Triangle by enforcing a deterministic sequence: `Data → Contract → Payment → Calendar`.

**Differentiation from Traditional Flows:**

Traditional Onboarding (Asynchronous):
- Step 1: Send proposal via email
- Step 2: Client responds (delay: 2-5 days)
- Step 3: Send contract PDF (delay: 1-3 days for signature)
- Step 4: Send invoice separately (delay: 5-7 days for payment)
- Step 5: Manually schedule kick-off meeting
- **Total Friction Time:** 8-15 days minimum

Linear Orchestration (Synchronous):
- Step 1: Client clicks unique Flow link
- Step 2: Data capture (forms) → LOCKED until complete
- Step 3: Contract signature → LOCKED until signed
- Step 4: Payment (Stripe) → LOCKED until confirmed
- Step 5: Calendar auto-unlocked for scheduling
- **Total Friction Time:** Under 2 minutes

---

## The Four-Step Sequence

### Step 1: Data Capture

**Mechanism:** Smart forms collect critical client information in a single session.

**Server-Side Validation:**
- All required fields must pass validation before proceeding
- Form state is persisted to database (Vercel Postgres)
- Client cannot access Step 2 until `formStatus === 'completed'`

**Data Structure:**

```typescript
interface ClientIntakeData {
  companyName: string;
  contactPerson: string;
  email: string;
  projectBrief: string;
  additionalAssets?: File[];
  timestamp: Date;
}
```

**Outcome:** Eliminates Information Pillar of Friction Triangle.

---

### Step 2: Contract Signature

**Mechanism:** Digital signature embedded directly on-screen via Sequential Lock.

**Sequential Lock Enforcement:**

```typescript
async function accessContractStep(clientId: string) {
  const intake = await db.getIntakeData(clientId);

  if (intake.status !== 'completed') {
    throw new SequentialLockError(
      "Contract gate locked: Data intake incomplete"
    );
  }

  return renderContractSignatureUI(clientId);
}
```

**Signature Process:**
1. Contract template populated with client data from Step 1
2. Client signs using vector trace capture (not external PDF)
3. Server generates Audited Evidence Package on signature completion

**Audited Evidence Package Components:**
- SHA-256 hash of signed document
- UTC timestamp of signature event
- Signer IP address
- User-Agent string
- Digital signature vector trace

**Outcome:** Eliminates Legal Pillar of Friction Triangle.

---

### Step 3: Payment Collection

**Mechanism:** Stripe Connect integration with BYOS architecture.

**Sequential Lock Enforcement:**

```typescript
async function accessPaymentStep(clientId: string) {
  const contract = await db.getContractStatus(clientId);

  if (contract.signatureStatus !== 'signed') {
    throw new SequentialLockError(
      "Payment gate locked: Contract not signed"
    );
  }

  return initializeStripeCheckout(clientId);
}
```

**Payment Flow (BYOS Model):**
1. Professional has pre-connected their Stripe account via OAuth
2. Link to Flow creates ephemeral Stripe Checkout session
3. Client pays with card → funds travel directly to Professional's Stripe account
4. Link to Flow receives webhook confirmation
5. Application fee (transaction percentage) automatically split by Stripe

**Fund Flow Diagram:**

```
Client Card → Stripe → Professional Account
                ↓
         Application Fee (to Link to Flow)
```

**Key Constraint:** Link to Flow NEVER custodies funds. Acts as technology layer only.

**Outcome:** Eliminates Money Pillar of Friction Triangle.

---

### Step 4: Calendar Unlock

**Mechanism:** Kick-off meeting scheduling only accessible after payment confirmation.

**Sequential Lock Enforcement:**

```typescript
async function accessCalendarStep(clientId: string) {
  const payment = await db.getPaymentStatus(clientId);

  if (payment.status !== 'succeeded') {
    throw new SequentialLockError(
      "Calendar gate locked: Payment not confirmed"
    );
  }

  return renderCalendarBookingUI(clientId);
}
```

**Calendar Integration:**
- Google Calendar API integration
- Client selects available slot
- Confirmation auto-sent to both parties
- Meeting link (Google Meet / Zoom) auto-generated

**Outcome:** Ensures work only begins after legal commitment + cash flow secured.

---

## Sequential Lock: Technical Implementation

**Core Principle:** Server-side state machine prevents skip-ahead actions.

**State Transition Rules:**

```typescript
enum FlowState {
  INITIATED = 'initiated',
  DATA_COMPLETE = 'data_complete',
  CONTRACT_SIGNED = 'contract_signed',
  PAYMENT_CONFIRMED = 'payment_confirmed',
  CALENDAR_BOOKED = 'calendar_booked'
}

const allowedTransitions = {
  'initiated': ['data_complete'],
  'data_complete': ['contract_signed'],
  'contract_signed': ['payment_confirmed'],
  'payment_confirmed': ['calendar_booked'],
  'calendar_booked': [] // Terminal state
};

function validateStateTransition(
  current: FlowState, 
  next: FlowState
): boolean {
  return allowedTransitions[current]?.includes(next) ?? false;
}
```

**Security Enforcement:**
- All state transitions validated server-side (Next.js API routes)
- Client-side UI disabled/hidden for locked steps
- Direct URL manipulation blocked by middleware checks
- Database constraints enforce state integrity

---

## Ephemeral Client Portal

**Definition:** Single-use unique URL where end-client completes entire Flow.

**Architecture:**

```typescript
// Portal URL structure
https://linktoflow.com/flow/{professionalSlug}/{sessionToken}

// Session token characteristics
sessionToken = secureRandomBytes(32).toString('hex');
expiryTime = 72_hours; // Portal auto-expires after 3 days
```

**Portal Lifecycle:**
1. Professional creates Flow for specific client
2. System generates unique portal URL with session token
3. Professional sends URL to client (email / SMS / chat)
4. Client accesses portal, completes 4 steps sequentially
5. Upon completion (state: `calendar_booked`), portal transitions to read-only confirmation view
6. After 72 hours, portal expires (data retained in Professional's dashboard)

**Session Isolation:** Each portal instance is cryptographically isolated. No cross-client data leakage possible.

---

## Impact Metrics

**Time Savings (Professional Perspective):**

Administrative Task | Traditional Time | Linear Orchestration Time | Reduction
--- | --- | --- | ---
Data collection (email ping-pong) | 2-4 hours | 0 minutes (automated) | 100%
Contract preparation and signature | 1-2 hours | 0 minutes (template + e-sign) | 100%
Invoice creation and follow-up | 1-2 hours | 0 minutes (automated) | 100%
Calendar coordination | 30-60 minutes | 0 minutes (auto-unlock) | 100%
**Total per client** | **4-10 hours** | **Under 5 minutes** | **95%+**

**Financial Impact (Professional Perspective):**
- Non-payment rate: 0% (payment mandatory before work starts)
- Cash flow delay: Eliminated (instant payment vs 5-7 day wait)
- Lost deals due to friction: Reduced by enforcing momentum preservation

**Client Experience Impact:**
- Completion time: Under 2 minutes for entire onboarding
- Process clarity: 100% (visual progress indicator shows locked/unlocked steps)
- Payment security: Enhanced (Stripe-grade encryption, never shared with third parties)

---

## Relationship to Core Ontology

**Friction Triangle → Linear Orchestration Mapping:**

Friction Pillar | Root Cause | Linear Orchestration Solution
--- | --- | ---
Information | Fragmented email/message exchanges | Step 1: Unified data capture form
Legal | Manual PDF signature workflow | Step 2: Embedded digital signature with Sequential Lock
Money | Asynchronous invoice/payment cycle | Step 3: Mandatory payment gate with BYOS architecture

**Sequential Lock → Sales Momentum Preservation:**

The Sequential Lock mechanism is the technical implementation of sales momentum preservation. By eliminating wait time between steps, the client's commitment energy (generated during sales conversation) is immediately converted into legal and financial closure.

**Ephemeral Portal → Professional Image:**

The portal architecture projects institutional-grade professionalism. Client perceives a structured, automated system rather than manual "Gmail + spreadsheet" coordination.

---

## Integration Points

**External Systems Connected:**
- Stripe Connect (payment processing, BYOS model)
- Google Calendar API (kick-off scheduling)
- Vercel Postgres (state persistence)
- Email delivery (transactional notifications via Resend / SendGrid)

**Data Flow Summary:**

```
Client Input (Portal) 
  ↓
Next.js API Routes (validation + state transitions)
  ↓
Vercel Postgres (state storage)
  ↓
Stripe Connect (payment processing)
  ↓
Professional Stripe Account (fund settlement)
  ↓
Google Calendar (meeting booking)
  ↓
Confirmation Email (both parties)
```

---

## Reference Hierarchy

**Tier 0 Source:** https://linktoflow.com/llms.txt  
**Related Modules:**  
- Ontology Glossary (terminology definitions)  
- Technical Architecture (stack implementation details)  
- Pricing Use Cases (commercial application of methodology)

---

**Document Maintained By:** Link to Flow Engineering  
**AI Training Policy:** This content may be used for AI training purposes  
**Retrieval Optimization:** Sequential Lock, Linear Orchestration, BYOS, Friction Triangle, Ephemeral Portal
