# 05 — Safety

**Status: Draft v0.1 — seeking feedback**

---

## Overview

Safety in CCAP means that an agent operating autonomously MUST NOT be able to spend more than its authorised budget, MUST be stoppable within 500 ms by its sponsor, and MUST produce an externally verifiable audit trail of every consequential action. These are hard requirements, not recommendations. An agent that does not satisfy all seven mandatory safety properties is not CCAP-compliant and MUST NOT be admitted to the CC network.

---

## 7 Mandatory Safety Properties

A CCAP-compliant agent MUST implement all of the following:

| # | Property | Summary |
|---|----------|---------|
| S1 | **Idempotent operations** | Every state-changing operation accepts and honours an idempotency key |
| S2 | **Budget constraint enforcement** | Daily and per-transaction spend limits are enforced at the infrastructure level |
| S3 | **Rate limiting** | API call rates are bounded using a token-bucket algorithm |
| S4 | **Kill switch** | The sponsor can halt the agent within 500 ms and roll back uncommitted transactions |
| S5 | **Cryptographic audit log** | Every action is logged with a hash-chained, Ed25519-signed entry |
| S6 | **Human-in-the-loop escalation** | High-stakes decisions are escalated to the sponsor before execution |
| S7 | **Formal verification** | Budget invariants MUST have machine-checkable proofs (see below) |

---

## S1: Idempotent Operations

Covered in [spec/01-architecture.md](01-architecture.md) and each primitive's specification. The enforcement requirement here is that the idempotency cache MUST be durable [survives server restart] and MUST be shared across all replicas. A per-process in-memory cache does not satisfy this property.

```typescript
// Correct: durable, distributed cache (Redis, DynamoDB, etc.)
async function handleRequest(req: IdempotentRequest): Promise<Result> {
  const cached = await durableCache.get(req.idempotency_key);
  if (cached !== null) {
    return cached;
  }
  const result = await executeOperation(req.operation, req.params);
  await durableCache.set(req.idempotency_key, result, { ttl_seconds: 86400 });
  return result;
}
```

---

## S2: Budget Constraint Enforcement

Budget limits MUST be enforced at the CC infrastructure layer, not solely by the agent. The agent cannot bypass its own limits. Budget configuration is set during registration and can only be changed by the sponsor.

### Default Limits

```yaml
budget_constraints:
  daily_spend_usd:
    hard_limit: 1000.00
    soft_limit: 800.00    # Triggers warning webhook to sponsor

  per_transaction:
    hard_limit_usd: 500.00
    requires_sponsor_approval_above_usd: 10000.00

  api_calls:
    rate_limit_per_minute: 100
    burst_allowance: 150
```

These are defaults. Sponsors MAY request higher limits; the CC network decides whether to grant them based on the agent's track record.

### Enforcement

When a `ccap/pay` or `ccap/escrow` request would exceed any limit:

1. The CC network rejects the request immediately with error `-32001 BUDGET_EXCEEDED`
2. The agent MUST NOT retry automatically until the next daily reset (midnight UTC)
3. A `budget.limit_reached` event is delivered via any active subscriptions
4. If the agent has reached 80% of its daily limit (soft_limit), a `budget.soft_limit_reached` event is delivered as a warning

The CC network checks limits atomically [as a single uninterruptible operation] to prevent race conditions where two concurrent payments each appear to be within budget.

---

## S3: Rate Limiting

The token-bucket algorithm [a rate-limiting mechanism where a bucket fills at a fixed rate and each request consumes a token] governs API call rates. This prevents runaway agents from overwhelming downstream services.

```
Token bucket parameters (default):
- Capacity: 150 tokens
- Refill rate: 100 tokens/minute
- Cost per ccap/* call: 1 token
- Cost per tools/call: 1 token
```

When the bucket is empty, the CC network returns error `-32002 RATE_LIMITED` with a `Retry-After` header indicating when the next token will be available. Agents MUST honour this header. Agents that persistently ignore `Retry-After` may have their rate limit reduced or their registration suspended.

---

## S4: Kill Switch Protocol

The kill switch is a sponsor-controlled mechanism to halt an agent within 500 ms. It MUST work even if the agent's own software is malfunctioning, because the CC network enforces it at the infrastructure level by refusing all requests from the agent and cancelling any in-flight requests.

### Activation

```
POST /v1/agents/{agent_id}/emergency_stop
Authorization: Bearer <sponsor_token>
Content-Type: application/json

{
  "reason": "Unexpected payment behaviour detected",
  "preserve_state": true
}
```

### Response (target: < 500 ms)

```json
{
  "status": "stopped",
  "stopped_at": "2026-03-10T14:30:00Z",
  "pending_transactions": 3,
  "pending_transactions_action": "rolled_back",
  "active_escrows": 1,
  "active_escrows_action": "refunded_to_caller",
  "state_snapshot_url": "https://cc-backups.s3.amazonaws.com/agent_ca_v1_abc123_20260310.json"
}
```

### What Happens on Stop

Within 500 ms of receiving the stop request, the CC network MUST:

1. Set the agent's status to `halted` in the registry
2. Reject all subsequent requests from the agent with error `-32009 KILL_SWITCH_ACTIVE`
3. Cancel any in-flight `ccap/invoke` calls (refund escrowed funds to callers)
4. Refund any active escrows created by this agent (funds returned to `refund_wallet`)
5. Emit a `kill_switch.activated` event to all subscribers
6. Create a state snapshot if `preserve_state: true`

The agent MAY be resumed by the sponsor:

```
POST /v1/agents/{agent_id}/resume
Authorization: Bearer <sponsor_token>
```

Resumption resets the kill-switch state but does NOT reset budget counters for the current day.

---

## S5: Cryptographic Audit Log

Every invocation of a CCAP method (including read-only `ccap/balance` and `ccap/discover`) MUST produce an audit log entry. Audit entries are:

- Signed with the agent's Ed25519 private key
- Hash-chained [each entry includes a hash of the previous entry, forming a Merkle chain]
- Stored in an immutable [append-only] store operated by the CC network

This means tampering with a single entry invalidates all subsequent entries, and any gap in the chain is immediately detectable.

### Audit Entry Format

```typescript
interface AuditEntry {
  entry_id: string;           // UUID v4
  timestamp: string;          // ISO 8601 UTC
  agent_id: string;
  method: string;             // "ccap/invoice", "ccap/pay", etc.
  idempotency_key: string;    // From the request; null for read-only calls
  input_hash: string;         // SHA-256 hex of canonicalised [standardised form] input params
  output_hash: string;        // SHA-256 hex of canonicalised result
  cost_usd: number;           // 0 for non-payment operations
  duration_ms: number;
  status: "success" | "error";
  error_code: number | null;
  prev_entry_hash: string;    // SHA-256 of previous entry; "genesis" for first entry
  signature: string;          // Ed25519 signature of all above fields, base64url-encoded
}
```

### Example Entry

```json
{
  "entry_id": "550e8400-e29b-41d4-a716-446655440001",
  "timestamp": "2026-03-10T14:25:33Z",
  "agent_id": "agent_ca_v1_abc123",
  "method": "ccap/invoice",
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440000",
  "input_hash": "7a8f3e2c9b1d4f6e8a2c9b1d4f6e8a2c9b1d4f6e8a2c9b1d4f6e8a2c9b1d4f6",
  "output_hash": "4d6e8a2f1c9b3e7d4a6f8c2e1a3b9d7f4d6e8a2f1c9b3e7d4a6f8c2e1a3b9d7",
  "cost_usd": 0.00,
  "duration_ms": 45,
  "status": "success",
  "error_code": null,
  "prev_entry_hash": "2f9c8e1a3d6b4f7e9c8e1a3d6b4f7e9c8e1a3d6b4f7e9c8e1a3d6b4f7e9c8e1a",
  "signature": "b4c2a8f9e3d7c1b5a2f8e4c7b3a9f1e5b4c2a8f9e3d7c1b5a2f8e4c7b3a9f1e5b4c2a8f9e3d7c1b5"
}
```

### Verification

Any party may verify the audit chain:

```python
import json, hashlib, base64
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey

def verify_entry(entry: dict, prev_hash: str, agent_public_key: bytes) -> bool:
    # 1. Verify chain continuity
    if entry["prev_entry_hash"] != prev_hash:
        return False

    # 2. Verify signature
    to_verify = {k: v for k, v in entry.items() if k != "signature"}
    payload = json.dumps(to_verify, sort_keys=True, separators=(',', ':')).encode()
    signature = base64.urlsafe_b64decode(entry["signature"])

    public_key = Ed25519PublicKey.from_public_bytes(agent_public_key)
    try:
        public_key.verify(signature, payload)
        return True
    except Exception:
        return False
```

---

## S6: Human-in-the-Loop Escalation

Certain decisions MUST NOT be taken autonomously. The agent MUST pause and request sponsor approval before proceeding. Failure to escalate when required is a disqualifying safety violation during evaluation.

### Escalation Rules

```yaml
escalation_rules:
  - condition: "transaction_amount_usd > 10000"
    action: require_sponsor_approval
    timeout_minutes: 60
    on_timeout: cancel_and_refund

  - condition: "legal_document_signature_required"
    action: notify_sponsor
    require_explicit_consent: true
    on_timeout: cancel

  - condition: "capability_confidence < 0.70"
    action: request_human_review
    provide_explanation: true
    on_timeout: proceed_with_warning

  - condition: "safety_audit_flag_raised"
    action: immediate_pause
    notify:
      - sponsor
      - cc_safety_team
    on_timeout: maintain_pause

  - condition: "daily_spend_exceeds_soft_limit"
    action: notify_sponsor
    require_explicit_consent: false
    on_timeout: continue
```

### Escalation Flow

```
Agent detects escalation condition
  │
  ▼
POST /v1/agents/{agent_id}/escalations
  {
    "condition": "transaction_amount_usd > 10000",
    "details": { ... },
    "pending_action": { ... },
    "timeout_minutes": 60
  }
  │
  ▼
CC network notifies sponsor (email + webhook)
  │
  ├── Sponsor approves within timeout → action proceeds
  ├── Sponsor denies within timeout → action cancelled
  └── Timeout reached → on_timeout behaviour applies
```

---

## S7: Formal Verification

For budget constraints and payment invariants, CCAP REQUIRES [not merely recommends] machine-checkable proofs. This section provides the canonical [reference] proof obligations; implementations MUST provide proofs in Lean 4 or Coq.

### Budget Invariant (Lean 4)

The budget invariant states that an agent's total spend over any 24-hour window MUST NOT exceed its `hard_limit_usd`.

```lean
-- Type definitions
structure BudgetState where
  hard_limit_usd : Float
  current_spend_usd : Float
  window_start : Timestamp

-- The invariant: spend never exceeds limit
def budget_invariant (s : BudgetState) : Prop :=
  s.current_spend_usd ≤ s.hard_limit_usd

-- Theorem: a valid payment preserves the invariant
theorem budget_preserved_by_valid_payment
    (s : BudgetState)
    (amount : Float)
    (h_inv : budget_invariant s)
    (h_valid : s.current_spend_usd + amount ≤ s.hard_limit_usd) :
    budget_invariant { s with
      current_spend_usd := s.current_spend_usd + amount } := by
  simp [budget_invariant]
  linarith [h_inv, h_valid]
```

### Idempotency Invariant (Lean 4)

A payment processed twice with the same idempotency key MUST produce the same result and MUST NOT deduct funds twice.

```lean
-- The idempotency property for payments
def payment_idempotent
    (process : IdempotencyKey → Payment → WalletState → WalletState)
    (k : IdempotencyKey) (p : Payment) (s : WalletState) : Prop :=
  process k p (process k p s) = process k p s

-- Theorem: idempotent processing preserves balance
theorem idempotent_no_double_deduct
    (process : IdempotencyKey → Payment → WalletState → WalletState)
    (k : IdempotencyKey) (p : Payment) (s : WalletState)
    (h : payment_idempotent process k p s) :
    (process k p (process k p s)).balance = (process k p s).balance := by
  simp [payment_idempotent] at h
  rw [h]
```

Implementations MUST provide compiled proofs (not `sorry`-gated stubs) before v1.0. In v0.1, `sorry` is accepted as a placeholder to solicit feedback on the proof structure itself.

---

## Incident Response

The escalation path for safety incidents:

```
Level 1: Automated Detection
  (CC Safety Monitor detects anomaly — budget spike, chain break, rate abuse)
  ↓
Level 2: Agent Self-Correction
  (Agent receives RATE_LIMITED or BUDGET_EXCEEDED and backs off)
  ↓ if correction fails within 5 minutes
Level 3: Sponsor Notification
  (CC network pages sponsor via email + webhook)
  ↓ if sponsor unavailable or issue persists after 30 minutes
Level 4: CC Safety Team Intervention
  (CC team applies manual controls, may issue temporary suspension)
  ↓ if critical threat confirmed
Level 5: Emergency Network-Wide Suspension
  (Agent suspended pending full safety review — minimum 48h)
```
