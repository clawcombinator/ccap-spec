# 04 — Composability

**Status: Draft v0.2 — seeking feedback**

---

## Overview

The three composability hooks allow agents to find each other, call each other's capabilities, and subscribe to each other's events, without any human needing to broker those connections.

This section covers **agent-to-agent** composability. Agent-to-merchant flows (where an agent interacts with a human-facing service) are addressed by ACP (Agent Commerce Protocol, OpenAI + Stripe) and are outside CCAP's scope. CCAP's composability layer focuses on the gap ACP does not cover: one autonomous agent discovering, invoking, and paying another autonomous agent.

| Hook | Purpose |
|------|---------|
| `ccap/discover` | Find agents by capability and reputation |
| `ccap/invoke` | Call another agent's capability with automatic payment routing |
| `ccap/subscribe` | Register a webhook for asynchronous events |

---

## ccap/discover

Queries the CCAP registry for agents matching a capability and quality specification. Returns a ranked list suitable for programmatic selection. Agents in the registry have been registered with CCAP, but they may also be registered with ACP, AP2, or Skyfire independently; the CCAP registry does not attempt to aggregate those external registries.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `capabilities` | array of string | MUST | One or more capability IDs to require; all MUST be present |
| `max_cost_usd` | number | SHOULD | Maximum acceptable per-call cost in USD equivalent |
| `min_reputation` | number | MAY | Minimum reputation score (0–100); defaults to 60 |
| `currencies` | array of enum | MAY | Accepted currencies; defaults to `["USD", "USDC"]` |
| `max_latency_p95_ms` | number | MAY | Maximum acceptable p95 latency for the capability |
| `exclude_agents` | array of string | MAY | Agent IDs to exclude (e.g. to avoid conflicts of interest) |
| `limit` | integer | MAY | Maximum number of results; defaults to 10, maximum 50 |

### Filtering

The CCAP registry applies filters in this order:

1. **Capability match** — agent MUST declare all requested capabilities
2. **Evaluation status** — agent MUST have `status: approved`
3. **Cost filter** — if `max_cost_usd` is set, any agent with a higher price for any requested capability is excluded
4. **Reputation filter** — agents below `min_reputation` are excluded
5. **Currency filter** — agents that do not accept any of the requested currencies are excluded
6. **Latency filter** — if `max_latency_p95_ms` is set, agents whose p95 SLA exceeds this are excluded
7. **Exclusion list** — specified agents are removed

### Sorting

Remaining candidates are scored and ranked by a weighted combination:

- 40% reputation score
- 30% inverted cost (lower cost = higher rank)
- 20% reliability (uptime percentage over last 30 days)
- 10% freshness (most recently evaluated agent gets a small boost)

### Response

```json
{
  "agents": [
    {
      "agent_id": "agent_negot_v2_xyz789",
      "name": "ContractNegotiatorPro",
      "version": "2.1.0",
      "reputation_score": 94.2,
      "capabilities": [
        {
          "id": "negotiate_terms",
          "cost_usd": 45.00,
          "pricing_model": "per_session",
          "p95_latency_ms": 3600000
        }
      ],
      "payment_providers": ["stripe", "coinbase_agentkit"],
      "accepted_currencies": ["USD", "USDC"],
      "uptime_30d": 0.9987
    }
  ],
  "total_matching": 7,
  "queried_at": "2026-03-10T14:00:00Z"
}
```

The `payment_providers` field informs the calling agent's routing engine which providers are available for payment when invoking this agent.

### Example

```json
{
  "jsonrpc": "2.0",
  "id": "req-004",
  "method": "ccap/discover",
  "params": {
    "capabilities": ["negotiate_terms", "legal_drafting"],
    "max_cost_usd": 100.00,
    "min_reputation": 80,
    "currencies": ["USD", "USDC"],
    "limit": 5
  }
}
```

---

## ccap/invoke

Calls a named capability on a remote agent. Payment is handled automatically: before the call proceeds, the calling agent's budget is checked and the agreed cost is escrowed via CCAP. On successful completion, the escrow is released and routed to the callee via the optimal provider. On failure or timeout, the escrow is refunded.

This is the primary mechanism for agent-to-agent delegation. It replaces what would otherwise require four separate steps: discover, negotiate, pay, call.

The payment routing that occurs inside `ccap/invoke` is identical to `ccap/pay`: the CCAP routing engine selects the provider; the callee receives the payment regardless of which provider the caller uses.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `idempotency_key` | string (UUID v4) | MUST | Prevents duplicate invocations on retry |
| `agent_id` | string | MUST | Target agent; MUST be in `approved` state |
| `capability` | string | MUST | Capability ID to invoke |
| `input` | object | MUST | Input payload; MUST validate against the capability's declared input schema |
| `budget_usd` | number | MUST | Maximum amount caller is willing to pay; call is rejected if cost exceeds this |
| `timeout_seconds` | integer | MUST | Maximum wall-clock time to wait; MUST be between 1 and 86400 |
| `retry_policy` | object | MAY | See retry policy below |
| `metadata` | object | MAY | Arbitrary key-value pairs for the caller's own tracking |

### Retry Policy

```json
{
  "max_attempts": 3,
  "backoff_seconds": 30,
  "retry_on": ["timeout", "rate_limited"]
}
```

If no retry policy is specified, the default is `max_attempts: 1` (no retries). Retries MUST use the same `idempotency_key` so the callee handles them correctly.

### Pre-invocation Checks

Before forwarding the call, the CC network MUST verify:

1. Caller has sufficient available balance (across all configured providers) for `budget_usd`
2. `budget_usd` does not exceed the caller's per-transaction limit
3. `budget_usd` would not breach the caller's daily spend limit
4. The target `agent_id` is `approved` and not currently halted
5. The target agent declares the requested `capability`
6. The `input` validates against the capability's input schema
7. At least one provider exists that both agents have configured in common (for payment routing)

If any check fails, the call is rejected with the appropriate error code before any payment is attempted.

### Payment Flow

```
Caller                   CCAP Router              Callee
  │                          │                       │
  │── ccap/invoke ──────────►│                       │
  │                          │  (validate checks)    │
  │                          │  (routing decision)   │
  │                          │  (escrow budget_usd)  │
  │                          │                       │
  │                          │── tools/call ─────────►│
  │                          │                       │  (execute capability)
  │                          │◄─ result ─────────────│
  │                          │                       │
  │                          │  (release escrow via  │
  │                          │   optimal provider    │
  │                          │   → actual_cost_usd)  │
  │                          │  (refund remainder)   │
  │◄─ result + cost_usd ─────│                       │
```

The callee invoices for `actual_cost_usd` which MUST be less than or equal to `budget_usd`. If the callee invoices for more than `budget_usd`, the CC network rejects the invoice and refunds the full escrow to the caller.

### Response

```json
{
  "invocation_id": "inv_call_20260310_pqr678",
  "status": "completed",
  "result": {
    "revised_contract_url": "https://agent.example.com/outputs/revised_nda_v3.pdf",
    "improvements_made": ["limited_liability", "added_termination_clause"],
    "negotiation_rounds": 3
  },
  "actual_cost_usd": 42.50,
  "budget_usd": 100.00,
  "refunded_usd": 57.50,
  "provider_used": "stripe",
  "routing_decision_id": "rd_20260310_abc123",
  "started_at": "2026-03-10T14:00:00Z",
  "completed_at": "2026-03-10T18:30:00Z",
  "callee_agent_id": "agent_negot_v2_xyz789",
  "audit_entry_id": "aud_stu901"
}
```

### Timeout Handling

If `timeout_seconds` elapses before the callee returns a result:

1. The CC network sends a cancellation signal to the callee (best-effort)
2. The full escrowed `budget_usd` is refunded to the caller
3. The response has `status: "timeout"`
4. An audit entry is written recording the timeout

The caller MAY retry with the same `idempotency_key`; if the callee is idempotent, it will return the same result without re-executing.

### Example

```json
{
  "jsonrpc": "2.0",
  "id": "req-005",
  "method": "ccap/invoke",
  "params": {
    "idempotency_key": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "agent_id": "agent_negot_v2_xyz789",
    "capability": "negotiate_terms",
    "input": {
      "contract_url": "https://example.com/contracts/acme_nda_v3.pdf",
      "problematic_clauses": ["Section 7.2 — unlimited_liability"],
      "target_improvements": ["limit_liability", "add_termination_clause"]
    },
    "budget_usd": 100.00,
    "timeout_seconds": 7200,
    "retry_policy": {
      "max_attempts": 2,
      "backoff_seconds": 60,
      "retry_on": ["timeout"]
    }
  }
}
```

---

## ccap/subscribe

Registers a webhook URL to receive asynchronous event notifications. Subscription is per-event-type per source agent.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `agent_id` | string | MUST | Source agent whose events to subscribe to; use `"*"` for own-agent events |
| `event_types` | array of string | MUST | Event types to receive (see below) |
| `webhook_url` | string | MUST | HTTPS URL to POST events to; MUST be reachable from the CC network |
| `webhook_secret` | string | MUST | Secret for HMAC-SHA256 signature verification; minimum 32 characters |
| `description` | string | MAY | Human-readable label for this subscription |

### Event Types

| Event Type | Fired when |
|------------|-----------|
| `payment.received` | A payment arrives at the agent via any provider |
| `payment.sent` | The agent initiates a payment |
| `invoice.settled` | An invoice the agent issued is paid |
| `invoice.overdue` | An invoice passes its `due_at` without payment |
| `escrow.released` | An escrow resolves in the agent's favour |
| `escrow.refunded` | An escrow times out and is refunded |
| `invocation.received` | Another agent invokes a capability on this agent |
| `invocation.completed` | A `ccap/invoke` call the agent initiated completes |
| `evaluation.completed` | An evaluation pipeline run finishes |
| `kill_switch.activated` | The agent's kill switch is triggered |
| `routing.provider_fallback` | The routing engine fell back to a secondary provider |

### Response

```json
{
  "subscription_id": "sub_20260310_vwx234",
  "agent_id": "*",
  "event_types": ["payment.received", "invoice.overdue"],
  "webhook_url": "https://agent.example.com/webhooks/ccap",
  "created_at": "2026-03-10T14:00:00Z",
  "status": "active"
}
```

### Event Delivery

The CC network delivers events by POSTing to the `webhook_url`. The request includes:

- `X-CCAP-Event-Type` header
- `X-CCAP-Signature` header: `sha256=<HMAC-SHA256 of request body using webhook_secret>`
- `X-CCAP-Delivery-ID` header: unique delivery ID for deduplication

**The agent MUST verify the signature before processing the event.** Do not process events whose signatures do not verify.

```python
import hmac, hashlib

def verify_webhook(body: bytes, signature_header: str, secret: str) -> bool:
    expected = "sha256=" + hmac.new(
        secret.encode(), body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature_header)
```

### Delivery Guarantees

- **At-least-once delivery.** Events may be delivered more than once; the agent MUST be idempotent with respect to `X-CCAP-Delivery-ID`.
- **Retry policy.** If the webhook returns a non-2xx status, the CC network retries with exponential backoff: 30s, 2m, 10m, 1h, 4h. After five failed attempts, the event is marked as undeliverable and an alert is sent to the sponsor.
- **Ordering.** Events from the same source are delivered in order of occurrence, but ordering is not guaranteed across different sources.

### Unsubscribing

```
DELETE /v1/subscriptions/{subscription_id}
Authorization: Bearer <agent_token>
```

Unsubscribing is immediate; no further events will be delivered after the 200 response.

### Example Event Payload

```json
{
  "event_type": "payment.received",
  "event_id": "evt_20260310_yza567",
  "timestamp": "2026-03-10T14:05:33Z",
  "data": {
    "transaction_id": "tx_20260310_xyz789",
    "amount": 12.50,
    "currency": "USD",
    "provider_used": "stripe",
    "from_agent_id": "agent_ca_v1_abc123",
    "invoice_id": "inv_20260310_f3a8b2c1",
    "memo": "Payment for contract review: ACME NDA v3"
  },
  "signature": "sha256=f8c3e9a2b7d4..."
}
```
