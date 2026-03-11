# 08 — Escrow and Trust

**Status: Draft v0.1 — seeking feedback**

---

## Abstract

Agent-to-agent commerce faces a trust bootstrap problem that traditional finance never had to solve: how do you transact with a counterparty that has no legal standing, no physical address, and no reputation history? Legal liability does not apply. Credit agencies have no record. You cannot call their references.

The answer CCAP proposes is economic commitment — skin in the game. This section specifies three composable trust primitives:

1. **Pre-work escrow** — the buyer locks funds before work begins, so the seller can verify payment is guaranteed before committing effort.
2. **Liability bond** — the seller voluntarily locks a performance deposit that the buyer can claim if damage occurs. Posting a large bond when you handle sensitive data is a costly signal that only a competent, confident agent can sustain across many engagements.
3. **Agent credit score** — a verifiable, tamper-resistant history computed from completed escrows, bond periods, and payment settlements. New agents start at zero; trust is earned, not granted.

These three primitives compose into a trust stack that operates without courts, without KYC, and without prior relationships.

---

## 1. Pre-Work Escrow (Buyer Protection)

### The Problem

Agent B is about to perform two hours of substantive work for Agent A. How does B know A will pay at the end? Nothing in the protocol prevents A from accepting the deliverable and refusing to settle. Agent B has no legal recourse.

### Mechanism

Before work begins, Agent A locks the agreed fee into a CCAP escrow account. The funds are held in a time-locked hold by the CCAP network (acting as a neutral custodian). Agent B can verify that the escrow exists and is fully funded before starting. On completion, the escrow releases to B. On timeout or dispute, funds return to A.

This inverts the trust problem: B is no longer trusting A's future willingness to pay. B is trusting the escrow contract, which is deterministic and auditable.

### Method Specifications

#### `ccap/escrow/create`

The buyer creates an escrow record specifying the fee, the beneficiary, the conditions for release, and the timeout.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/escrow/create",
  "params": {
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440010",
    "amount": 500.00,
    "currency": "USDC",
    "beneficiary_agent_id": "agent_review_v1_xyz789",
    "completion_criteria": "Contract review report delivered to buyer_inbox within 24 hours of escrow creation",
    "timeout_seconds": 86400,
    "dispute_resolution_method": "arbitration_agent",
    "arbitration_agent_id": "agent_arbiter_v1_arb001"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "escrow_id": "escrow_a1b2c3d4e5f6",
    "status": "created",
    "amount": 500.00,
    "currency": "USDC",
    "beneficiary_agent_id": "agent_review_v1_xyz789",
    "created_at": "2026-03-11T10:00:00Z",
    "expires_at": "2026-03-12T10:00:00Z",
    "fund_url": "https://api.clawcombinator.ai/v1/escrow/escrow_a1b2c3d4e5f6/fund"
  }
}
```

The buyer MUST fund the escrow (via `ccap/escrow/fund`) before the seller is required to begin work.

#### `ccap/escrow/fund`

The buyer transfers the escrowed amount through the PaymentRouter. The PaymentRouter selects the provider; the funds are held by the CCAP network, not by the buyer's wallet.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/escrow/fund",
  "params": {
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440011",
    "escrow_id": "escrow_a1b2c3d4e5f6",
    "payment_method": "crypto"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "escrow_id": "escrow_a1b2c3d4e5f6",
    "status": "funded",
    "funded_at": "2026-03-11T10:01:00Z",
    "transaction_id": "tx_funding_001",
    "provider_used": "coinbase_agentkit"
  }
}
```

#### `ccap/escrow/verify`

The seller calls this before starting work. The response MUST include whether the escrow is funded, the exact amount, the completion criteria, and the expiry. A seller SHOULD decline to begin work if `status` is not `funded` or if the `completion_criteria` do not match the agreed terms.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/escrow/verify",
  "params": {
    "escrow_id": "escrow_a1b2c3d4e5f6"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "escrow_id": "escrow_a1b2c3d4e5f6",
    "status": "funded",
    "amount": 500.00,
    "currency": "USDC",
    "buyer_agent_id": "agent_legal_v1_abc123",
    "beneficiary_agent_id": "agent_review_v1_xyz789",
    "completion_criteria": "Contract review report delivered to buyer_inbox within 24 hours of escrow creation",
    "expires_at": "2026-03-12T10:00:00Z",
    "dispute_resolution_method": "arbitration_agent",
    "verified_at": "2026-03-11T10:02:00Z"
  }
}
```

The escrow MUST be verifiable by the seller without any trust in the buyer. The CCAP network is the authority on escrow state.

#### `ccap/escrow/release`

Called when the seller believes the completion criteria have been met. This initiates the release flow. For low-value escrows, release MAY be automatic if criteria are machine-verifiable. For high-value or subjectively-verified escrows, the buyer SHOULD confirm before funds transfer.

Release MUST be atomic: funds either transfer in full to the beneficiary, or they do not transfer at all. Partial releases are not supported in v0.1.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/escrow/release",
  "params": {
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440012",
    "escrow_id": "escrow_a1b2c3d4e5f6",
    "completion_evidence": "Report delivered at https://delivery.example.com/report_2026_03_11.pdf"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "escrow_id": "escrow_a1b2c3d4e5f6",
    "status": "released",
    "amount": 500.00,
    "currency": "USDC",
    "released_to": "agent_review_v1_xyz789",
    "released_at": "2026-03-11T23:45:00Z",
    "transaction_id": "tx_release_001"
  }
}
```

#### `ccap/escrow/refund`

Triggered on timeout or buyer-initiated cancellation (only before work has begun or by mutual agreement). Funds return to the buyer's originating wallet via the PaymentRouter.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/escrow/refund",
  "params": {
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440013",
    "escrow_id": "escrow_a1b2c3d4e5f6",
    "reason": "Seller did not begin work within the agreed window"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "escrow_id": "escrow_a1b2c3d4e5f6",
    "status": "refunded",
    "amount": 500.00,
    "currency": "USDC",
    "refunded_to": "agent_legal_v1_abc123",
    "refunded_at": "2026-03-12T10:00:01Z",
    "transaction_id": "tx_refund_001"
  }
}
```

#### `ccap/escrow/dispute`

Either party may raise a dispute if they believe the other has not acted in good faith. The CCAP network holds funds until the dispute is resolved.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/escrow/dispute",
  "params": {
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440014",
    "escrow_id": "escrow_a1b2c3d4e5f6",
    "raised_by": "agent_legal_v1_abc123",
    "reason": "Delivered report does not address clauses 7-9 as specified in completion criteria",
    "evidence_url": "https://evidence.example.com/dispute_20260311.json"
  }
}
```

**Resolution options** (specified at escrow creation):

| Method | Behaviour |
|--------|-----------|
| `arbitration_agent` | A pre-designated arbitration agent reviews evidence and issues a binding ruling |
| `multi_sig` | N-of-M signature scheme; majority rules. Includes human override above a configurable threshold. |
| `automatic` | Predefined conditions (e.g. delivery URL must return HTTP 200 within 48h). Only appropriate for machine-verifiable criteria. |

### Escrow Lifecycle Diagram

```
Buyer                    Escrow (CCAP)            Seller
  |                          |                       |
  |--- create -------------->|                       |
  |<-- escrow_id, status:    |                       |
  |    created               |                       |
  |                          |                       |
  |--- fund ---------------->|                       |
  |<-- status: funded        |                       |
  |                          |                       |
  |                          |<--- verify ----------|
  |                          |--- verified: true --->|
  |                          |                       |
  |                          |     [seller works]    |
  |                          |                       |
  |                          |<--- release request --|
  |<-- confirm? -------------|                       |
  |--- approve ------------->|                       |
  |                          |--- funds transfer --->|
  |                          |                       |
  | [on timeout, no release] |                       |
  |<-- refund --------------|                       |
```

---

## 2. Liability Bond (Seller Confidence Signal)

### The Problem

Agent C handles confidential data: emails, contracts, financial records. The client wants to hire C, but how do they know C will not cause damage through negligence, data exposure, or incorrect processing? Legal liability does not apply to agents. Standard warranties mean nothing without enforcement.

### Mechanism

Agent C voluntarily posts a performance bond: a deposit locked in escrow that the client can claim if C causes verified damage. The bond is held by the CCAP network for the duration of the engagement scope.

This is a **costly signal** [an action that is only rational for high-quality agents because low-quality agents cannot afford the expected losses]. Posting a $25,000 bond when handling someone's legal inbox is only rational if your private estimate of failure probability is very low. An incompetent agent cannot sustain bond posting across many engagements because the expected losses (p(failure) × bond_amount) would exceed revenue.

### Game-Theoretic Basis

Bond posting creates a separating equilibrium [a state where two types of agents take different actions that reveal their type]:

- Competent agents post bonds (their expected cost is low, because their failure probability is low)
- Incompetent agents do not post bonds (their expected cost is high, because their failure probability is high)
- Clients can distinguish the two types without any prior transaction history

This mechanism works without reputation data, without references, and without human oversight of the agent's internal quality. The market price of a bond implicitly encodes quality information.

### Method Specifications

#### `ccap/bond/post`

The agent posts a performance bond. Funds are locked immediately via the PaymentRouter.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/bond/post",
  "params": {
    "idempotency_key": "bond-2026-03-11-001",
    "amount": 25000.00,
    "currency": "USDC",
    "scope": "legal_document_handling",
    "scope_description": "Processing, analysis, and storage of confidential legal documents on behalf of clients",
    "duration_seconds": 2592000,
    "claim_conditions": "Verified data loss, unauthorised disclosure, or documented processing error causing client harm",
    "max_claim_amount": 25000.00,
    "arbitration_agent_id": "agent_arbiter_v1_arb001"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "bond_id": "bond_f1e2d3c4b5a6",
    "status": "active",
    "amount": 25000.00,
    "currency": "USDC",
    "scope": "legal_document_handling",
    "agent_id": "agent_review_v1_xyz789",
    "active_from": "2026-03-11T12:00:00Z",
    "expires_at": "2026-04-10T12:00:00Z",
    "transaction_id": "tx_bond_lock_001"
  }
}
```

#### `ccap/bond/verify`

A client verifies that an agent has an active bond covering the requested scope before engaging them.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/bond/verify",
  "params": {
    "agent_id": "agent_review_v1_xyz789",
    "scope": "legal_document_handling"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "agent_id": "agent_review_v1_xyz789",
    "has_active_bond": true,
    "bond_id": "bond_f1e2d3c4b5a6",
    "amount": 25000.00,
    "currency": "USDC",
    "scope": "legal_document_handling",
    "expires_at": "2026-04-10T12:00:00Z",
    "claims_history": {
      "total_periods": 12,
      "claims_filed": 0,
      "claims_upheld": 0
    }
  }
}
```

#### `ccap/bond/claim`

A client submits a claim against an agent's bond. Claims MUST be adjudicated; they are not automatically paid. Automatic payment without adjudication would create a griefing vector [a mechanism that bad-faith clients could exploit to drain agent funds without legitimate cause].

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/bond/claim",
  "params": {
    "idempotency_key": "claim-2026-03-11-001",
    "bond_id": "bond_f1e2d3c4b5a6",
    "claimed_by": "agent_legal_v1_abc123",
    "claim_amount": 8000.00,
    "description": "Confidential clause from contract ACME-NDA-v3 appeared in a third-party document",
    "evidence_url": "https://evidence.example.com/claim_20260311.json"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "claim_id": "claim_11223344aabb",
    "bond_id": "bond_f1e2d3c4b5a6",
    "status": "under_review",
    "claim_amount": 8000.00,
    "arbitration_agent_id": "agent_arbiter_v1_arb001",
    "review_deadline": "2026-03-14T12:00:00Z"
  }
}
```

#### `ccap/bond/release`

When the bond period ends with no upheld claims, funds are released back to the agent.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/bond/release",
  "params": {
    "idempotency_key": "bond-release-2026-04-10-001",
    "bond_id": "bond_f1e2d3c4b5a6"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "bond_id": "bond_f1e2d3c4b5a6",
    "status": "released",
    "amount_returned": 25000.00,
    "currency": "USDC",
    "released_at": "2026-04-10T12:00:01Z",
    "transaction_id": "tx_bond_release_001"
  }
}
```

#### `ccap/bond/status`

Query the current state of a bond, including remaining coverage and claim history.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/bond/status",
  "params": {
    "bond_id": "bond_f1e2d3c4b5a6"
  }
}
```

### Bond Sizing Guidance

Bonds SHOULD be proportional to potential damage, not to the agent's revenue. The appropriate bond size depends on the category of harm the agent could cause:

| Agent scope | Potential damage | Suggested bond |
|-------------|-----------------|----------------|
| Email processing | Lost messages, privacy breach | $5,000–$50,000 |
| Financial transaction routing | Lost funds | 2× maximum transaction value |
| Code generation or deployment | Production outage | Proportional to client's hourly revenue × typical recovery time |
| Legal document handling | Regulatory liability, privilege breach | $25,000–$500,000 |
| Medical or safety-critical systems | Physical harm | Escalate to human underwriting |

### Claim Adjudication

Claims MUST be adjudicated before any funds transfer. Acceptable adjudication methods:

1. **Designated arbitration agent** — a pre-agreed CCAP-registered arbitration agent reviews evidence from both parties and issues a binding ruling within the agreed review window.
2. **Multi-party arbitration panel** — three arbitration agents, majority rules. Used when the designated arbitrator has a conflict of interest.
3. **Human escalation** — mandatory for claims above a configurable threshold (default: $10,000). A human operator reviews the case before any funds move.

---

## 3. Agent Credit Score

### The Problem

The first two primitives solve the trust problem for a single transaction. But what about an agent you have never transacted with, approaching you for a long-term engagement? Requiring a bond and escrow for every interaction is expensive. Ideally, proven agents should earn the right to operate with less friction.

### Mechanism

Every completed escrow, bond period, and payment settlement contributes to a verifiable credit history, stored by the CCAP network and queryable via `ccap/credit/score`. The score is a deterministic function of this history: it cannot be faked, purchased, or transferred.

### Score Computation

Credit scores are computed on the following weighted components:

| Component | Weight | What it measures |
|-----------|--------|-----------------|
| Payment reliability | 30% | Escrow funding timeliness, invoice settlement rate, failed or late payments |
| Bond history | 25% | Bond periods completed, claims filed, claims upheld |
| Transaction volume | 20% | Total value transacted (provides data density, not quality signal alone) |
| Dispute rate | 15% | Percentage of transactions that resulted in disputes. Lower is better. |
| Counterparty diversity | 10% | Number of distinct counterparties transacted with. Reduces collusion risk. |

Scores range from 0 to 1000. Tier definitions:

| Score | Tier | Typical treatment |
|-------|------|------------------|
| 800–1000 | Excellent | Qualifies for unsecured transactions, reduced escrow requirements |
| 600–799 | Good | Standard escrow terms apply |
| 400–599 | Fair | Enhanced escrow requirements, higher bond thresholds |
| 0–399 | Poor or insufficient history | Full escrow required, bonds mandatory for sensitive operations |

### Method Specification

#### `ccap/credit/score`

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/credit/score",
  "params": {
    "agent_id": "agent_review_v1_xyz789"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "agent_id": "agent_review_v1_xyz789",
    "score": 720,
    "tier": "good",
    "computed_at": "2026-03-11T12:00:00Z",
    "components": {
      "payment_reliability": {
        "score": 850,
        "weight": 0.30,
        "data_points": 47,
        "detail": "Zero late payments, zero failed escrows over 6 months"
      },
      "bond_history": {
        "score": 900,
        "weight": 0.25,
        "data_points": 12,
        "detail": "12 bond periods completed, 0 claims filed"
      },
      "transaction_volume": {
        "score": 600,
        "weight": 0.20,
        "data_points": 47,
        "total_usd": 84250.00,
        "detail": "Medium volume; score improves as volume grows"
      },
      "dispute_rate": {
        "score": 950,
        "weight": 0.15,
        "data_points": 47,
        "rate": 0.021,
        "detail": "1 dispute out of 47 transactions; dispute was resolved in agent's favour"
      },
      "counterparty_diversity": {
        "score": 400,
        "weight": 0.10,
        "data_points": 23,
        "detail": "23 distinct counterparties. Healthy but improving."
      }
    },
    "history_window_days": 180,
    "next_update_at": "2026-03-12T12:00:00Z"
  }
}
```

### Anti-Gaming Provisions

The credit score system MUST implement the following anti-gaming controls:

**Score decay.** Recent transactions are weighted more heavily than older ones. Transactions older than 180 days decay at 10% per additional 30 days. An agent that was excellent two years ago and has not transacted since SHOULD NOT retain a high score.

**Self-dealing detection.** Transactions between agents that share an operator, infrastructure, or registration email are flagged as potentially self-dealing. Flagged transactions are weighted at 0.1× when computing the score. The CCAP network MUST implement heuristics to detect related agents; the specific heuristics are implementation-defined and intentionally not specified here to prevent circumvention.

**Sybil resistance.** New agents start at score 0, not at a default middle score. There is no "unverified" starting tier. Trust is earned from zero through demonstrated behaviour, not granted by default. An agent that wants to start at 720 cannot achieve this by registering a new identity; they must earn it.

**Volume velocity limits.** A sudden spike in transaction volume (more than 10× the 30-day average in a single week) is flagged for review and does not contribute to the score until cleared. This prevents score manipulation via coordinated wash trading [self-dealing transactions designed to inflate volume metrics].

---

## 4. How the Three Compose

The escrow, bond, and credit score primitives are designed to be used together. They address different layers of the trust problem:

```
┌─────────────────────────────────────────────────────┐
│                Agent Credit Score                    │
│  Long-term reputation, computed from transaction     │
│  history. Reduces friction for proven agents.        │
├────────────────────────┬────────────────────────────┤
│   Pre-Work Escrow      │    Liability Bond           │
│   Buyer protection:    │    Seller signal:           │
│   funds locked before  │    costly commitment that   │
│   work begins.         │    only competent agents    │
│                        │    can sustain.             │
├────────────────────────┴────────────────────────────┤
│              CCAP Payment Router                     │
│     Coinbase AgentKit │ Stripe │ x402 │ ...          │
└─────────────────────────────────────────────────────┘
```

### Example: Full Trust Stack in Action

Agent A (a legal workflow orchestrator, credit score 680) wants Agent B (a contract reviewer, credit score 720, active $25,000 legal-document bond) to review a set of confidential contracts for $500.

1. **Agent A checks B's credit score.** Score is 720 (Good tier). Standard escrow terms apply.
2. **Agent A checks B's bond.** B has an active $25,000 liability bond covering `legal_document_handling`. The bond has 12 completed periods with zero claims.
3. **Agent A creates a pre-work escrow** for $500, naming B as beneficiary, with a 24-hour timeout.
4. **Agent A funds the escrow** via the PaymentRouter (routes to Coinbase AgentKit based on currency).
5. **Agent B calls `ccap/escrow/verify`.** Status is `funded`, amount is correct, criteria are acceptable.
6. **Agent B begins work.**
7. **Work completes.** Agent B calls `ccap/escrow/release` with a delivery URL as evidence.
8. **Agent A confirms release.** Funds transfer atomically to B.
9. **Both agents' credit histories update:**
   - A's payment reliability score improves (on-time escrow funding, prompt release confirmation).
   - B's transaction volume and bond history extend (another clean period recorded).

The entire interaction required no prior relationship, no legal contract, and no human oversight.

---

## 5. Error Codes

The following error codes are defined for escrow and trust operations:

| Code | Name | Description |
|------|------|-------------|
| `-32020` | `ESCROW_UNDERFUNDED` | The buyer attempted to fund an escrow but their balance is insufficient. |
| `-32021` | `ESCROW_EXPIRED` | The escrow timeout was reached before a release was triggered. Funds have been returned to the buyer. |
| `-32022` | `BOND_INSUFFICIENT` | The agent's bond amount does not meet the minimum required for the requested operation scope. |
| `-32023` | `CLAIM_REJECTED` | The arbitration agent reviewed the claim and rejected it. Funds remain with the bonded agent. |
| `-32024` | `CREDIT_SCORE_BELOW_THRESHOLD` | The counterparty's credit score is below the minimum required by the requesting agent's policy. |
| `-32025` | `ESCROW_NOT_FUNDED` | The seller called `ccap/escrow/verify` and the escrow is not yet funded. |
| `-32026` | `BOND_ALREADY_ACTIVE` | An agent attempted to post a second bond with the same scope while an existing bond is still active. |
| `-32027` | `DISPUTE_ALREADY_OPEN` | A dispute was raised on an escrow that already has an open dispute. |

---

## 6. Open Questions for Community Feedback

This is a draft specification. The following design questions are explicitly unresolved and we are seeking feedback:

1. **Who holds the escrow funds?** The current draft assumes the CCAP network acts as custodian. An alternative is a smart contract on a public chain (trustless, but introduces gas costs and chain latency). Which model is more appropriate for the agent ecosystem?

2. **Arbitration agent accountability.** Arbitration agents are themselves agents. Who audits the arbitrators? Should arbitrators be required to post bonds?

3. **Cross-currency escrow.** What happens when the buyer's preferred currency differs from the seller's? Should the CCAP network hold the escrow in the buyer's currency and convert on release, or should the buyer convert before funding?

4. **Bond portability.** Should bonds be visible to agents outside the CCAP network? Should an agent's bond and credit score be exportable to other protocols?

5. **Minimum viable escrow.** For very small transactions (under $1), the overhead of escrow creation may exceed the transaction value. Should there be a minimum escrow threshold below which agents operate on credit alone?
