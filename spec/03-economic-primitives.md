# 03 — Economic Primitives

**Status: Draft v0.2 — seeking feedback**

---

## Overview

CCAP defines four economic primitives as MCP tools in the `ccap/` namespace. These are router-level operations: they abstract over the underlying payment providers (Coinbase AgentKit, Stripe, x402, Skyfire) and expose a single, consistent interface regardless of which rail ultimately executes the transaction.

Every CCAP-compliant agent MUST implement all four. Each primitive MUST be idempotent [safe to repeat without changing the result] via the `idempotency_key` parameter, MUST emit an audit log entry on every invocation, and MUST respect the agent's configured budget constraints.

| Primitive | Router-level behaviour |
|-----------|----------------------|
| `ccap/invoice` | Provider-agnostic payment request; recipient settles via any configured rail |
| `ccap/pay` | Routes to the optimal provider based on currency, amount, and recipient configuration |
| `ccap/balance` | Aggregated balance across all configured providers |
| `ccap/escrow` | Time-locked, multi-currency hold with condition triggers — the genuine novel contribution of this spec |

---

## ccap/invoice

Creates a provider-agnostic payment request. The invoice specifies what is owed and in which currencies it can be settled. The recipient's CCAP routing engine determines which provider to use for settlement.

This is distinct from a Stripe invoice or an on-chain payment request: it is settled by whatever rail the payer has configured and prefers.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `idempotency_key` | string (UUID v4) | MUST | Deduplication key; server returns cached result for 24h |
| `amount` | number | MUST | Positive decimal; maximum 2 decimal places for fiat currencies |
| `currency` | enum | MUST | Requested settlement currency: `"USD"`, `"USDC"`, `"ETH"` |
| `acceptable_settlement_methods` | array | SHOULD | Providers the invoicer will accept; omit to accept any configured provider |
| `description` | string | MUST | Human-readable description of the service; max 500 characters |
| `recipient_agent_id` | string | SHOULD | CCAP agent ID of the payer, if known; enables routing optimisation |
| `recipient_wallet` | string | MAY | Fallback wallet address for out-of-band settlement |
| `due_date` | string (ISO 8601) | SHOULD | Payment due date; defaults to 7 days from creation |
| `line_items` | array | MAY | Itemised breakdown; each item has `description`, `quantity`, `unit_cost_usd` |
| `metadata` | object | MAY | Arbitrary key-value pairs for the agent's own tracking |

### Acceptable Settlement Methods

```json
"acceptable_settlement_methods": [
  { "provider": "stripe", "currency": "USD" },
  { "provider": "coinbase_agentkit", "currency": "USDC" },
  { "provider": "x402" }
]
```

If `acceptable_settlement_methods` is omitted, the payer's routing engine selects the provider. If it is specified, the payer's routing engine MUST use one of the listed methods or the payment is rejected with `SETTLEMENT_METHOD_REJECTED`.

### Response

```json
{
  "invoice_id": "inv_20260310_f3a8b2c1",
  "status": "pending",
  "amount": 12.50,
  "currency": "USD",
  "description": "Contract review: ACME NDA v3",
  "created_at": "2026-03-10T14:00:00Z",
  "due_at": "2026-03-17T14:00:00Z",
  "payment_url": "https://pay.clawcombinator.ai/inv_20260310_f3a8b2c1",
  "audit_entry_id": "aud_abc123"
}
```

### Example

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "method": "ccap/invoice",
  "params": {
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 12.50,
    "currency": "USD",
    "acceptable_settlement_methods": [
      { "provider": "stripe", "currency": "USD" },
      { "provider": "coinbase_agentkit", "currency": "USDC" }
    ],
    "description": "Contract review: ACME NDA v3 (14 pages)",
    "due_date": "2026-03-17T14:00:00Z",
    "line_items": [
      { "description": "Risk identification", "quantity": 14, "unit_cost_usd": 0.50 },
      { "description": "Executive summary", "quantity": 1, "unit_cost_usd": 5.50 }
    ]
  }
}
```

### Invoice Lifecycle

```
created → pending → settled (via any accepted provider)
                 ↘ overdue → cancelled
                 ↘ disputed → (resolved → settled | refunded)
```

An agent SHOULD monitor invoice status via the `ccap/subscribe` webhook. An invoice that transitions to `overdue` does not automatically cancel; the agent MUST decide whether to cancel or negotiate.

---

## ccap/pay

Settles a payment from the agent. The routing engine selects the optimal provider based on the currency, amount, recipient configuration, and current provider costs and latencies. The caller does not specify a provider; the router decides.

This is used to pay sub-agents for delegated work, refund clients, or settle any other obligation.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `idempotency_key` | string (UUID v4) | MUST | Prevents double payment on retry |
| `invoice_id` | string | SHOULD | If paying a specific CCAP invoice; enables automatic reconciliation |
| `amount` | number | MUST (if no invoice_id) | Amount to pay; if `invoice_id` is provided, MUST match invoice amount |
| `currency` | enum | MUST | One of: `"USD"`, `"USDC"`, `"ETH"` |
| `recipient_agent_id` | string | SHOULD | CCAP agent ID of the recipient, if known; enables routing optimisation |
| `recipient_wallet` | string | MAY | Fallback wallet address if recipient is not a CCAP agent |
| `memo` | string | MUST | Human-readable reason for payment; max 200 characters |
| `preferred_provider` | string | MAY | Hint to the routing engine; CCAP MAY override if the preferred provider is unavailable |
| `metadata` | object | MAY | Arbitrary key-value pairs |

### Response

```json
{
  "transaction_id": "tx_20260310_xyz789",
  "status": "settled",
  "amount": 12.50,
  "currency": "USD",
  "provider_used": "stripe",
  "routing_decision_id": "rd_20260310_abc123",
  "settled_at": "2026-03-10T14:05:33Z",
  "provider_reference": "pi_3OqL3hLrKgd1Q2B31YQ6KBVQ",
  "audit_entry_id": "aud_def456"
}
```

The `provider_used` field identifies which provider executed the transaction. The `routing_decision_id` references the full routing decision record (see `schemas/routing-decision.schema.json`). For on-chain payments, `provider_reference` is the transaction hash; for Stripe payments, it is the payment intent ID.

### Routing Transparency

The caller MAY inspect the routing decision to understand why a particular provider was selected:

```
GET /v1/routing-decisions/{routing_decision_id}
Authorization: Bearer <agent_token>
```

This allows sponsors to audit payment routing and, if needed, reconfigure their provider preferences.

### Idempotency Guarantee

If `ccap/pay` is called twice with the same `idempotency_key`, the second call MUST return the result of the first call without initiating a second payment. This guarantee holds regardless of which provider was selected.

```
First call:  → routing decision → provider call → settled
Second call: → cached result returned            (no new payment)
```

### Example

```json
{
  "jsonrpc": "2.0",
  "id": "req-002",
  "method": "ccap/pay",
  "params": {
    "idempotency_key": "6ba7b810-9dad-11d1-80b4-00c04fd430c8",
    "invoice_id": "inv_20260310_f3a8b2c1",
    "amount": 12.50,
    "currency": "USD",
    "recipient_agent_id": "agent_ca_v1_abc123",
    "memo": "Payment for contract review: ACME NDA v3"
  }
}
```

---

## ccap/balance

Returns the aggregated balance across all configured payment providers. This is a read-only operation that does not require an idempotency key.

Rather than querying Coinbase AgentKit for crypto balances and Stripe for fiat balances separately, `ccap/balance` provides a unified view. The routing engine uses this view when evaluating whether a payment can proceed.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `currency` | enum | MAY | Filter to specific currency; omit to return all balances across all providers |
| `provider` | string | MAY | Filter to a specific provider; omit to aggregate across all |

### Response

```json
{
  "balances": [
    {
      "currency": "USD",
      "provider": "stripe",
      "available": 450.00,
      "pending": 12.50,
      "reserved_in_escrow": 0.00
    },
    {
      "currency": "USDC",
      "provider": "coinbase_agentkit",
      "available": 1250.000000,
      "pending": 0.000000,
      "reserved_in_escrow": 300.000000
    }
  ],
  "totals_by_currency": {
    "USD": { "available": 450.00, "pending": 12.50, "reserved_in_escrow": 0.00 },
    "USDC": { "available": 1250.00, "pending": 0.00, "reserved_in_escrow": 300.00 }
  },
  "queried_at": "2026-03-10T14:10:00Z",
  "audit_entry_id": "aud_ghi789"
}
```

`available` is the amount immediately spendable (across all providers for that currency). `pending` is inbound payments not yet confirmed. `reserved_in_escrow` is locked by active CCAP escrows.

### Example

```json
{
  "jsonrpc": "2.0",
  "id": "req-003",
  "method": "ccap/balance",
  "params": {
    "currency": "USD"
  }
}
```

---

## ccap/escrow

Locks funds in a conditional hold. This is CCAP's most novel contribution relative to the existing ecosystem. No individual provider (Coinbase AgentKit, Stripe, x402, or Skyfire) offers time-locked, multi-currency, condition-triggered escrow as a standard primitive. CCAP implements this at the router layer, above the providers.

The escrowed amount is deducted from `available` and added to `reserved_in_escrow`. It releases to a recipient when specified conditions are met, or refunds to the sender after a timeout.

Escrow is designed for multi-step workflows where payment should only occur on delivery — for example, paying a sub-agent after its result passes validation, or holding funds pending a compliance check.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `idempotency_key` | string (UUID v4) | MUST | Prevents duplicate escrows on retry |
| `amount` | number | MUST | Amount to lock |
| `currency` | enum | MUST | One of: `"USD"`, `"USDC"`, `"ETH"` |
| `recipient_agent_id` | string | SHOULD | CCAP agent ID who receives funds on release |
| `recipient_wallet` | string | MAY | Fallback wallet address if recipient is not a CCAP agent |
| `refund_agent_id` | string | SHOULD | CCAP agent ID who receives the refund on timeout/cancellation |
| `refund_wallet` | string | MAY | Fallback wallet for refunds if not a CCAP agent |
| `conditions` | array | MUST | Conditions that MUST all be satisfied for release (see below) |
| `timeout_hours` | number | MUST | Hours until automatic refund if conditions not met; maximum 720 (30 days) |
| `description` | string | SHOULD | Human-readable purpose; max 200 characters |
| `settlement_provider` | string | MAY | Hint to the router for which provider should hold and release the funds |

### Condition Object

```json
{
  "type": "agent_result_delivered",
  "agent_id": "agent_negotiator_v1_xyz",
  "capability": "negotiate_terms",
  "task_id": "task_20260310_abc"
}
```

Supported condition types in v0.2:

| Type | Satisfied when |
|------|---------------|
| `agent_result_delivered` | The specified agent has returned a result for the given task |
| `human_approval` | A human with the specified email has approved via the CC approval interface |
| `timestamp_reached` | The current time is at or after `release_after` |
| `external_webhook` | The CC network has received a POST to the escrow's callback URL with the correct HMAC |

### Escrow Lifecycle

```
created → locked → released (funds routed to recipient via optimal provider)
                ↘ refunded  (timeout reached, funds returned to refund_agent_id)
                ↘ cancelled (sponsor-authorised cancellation, full refund)
```

At release time, the routing engine selects the provider for the outbound payment using the same criteria as `ccap/pay`. The escrow hold itself is maintained at the CCAP layer, not within any single provider.

### Response

```json
{
  "escrow_id": "esc_20260310_jkl012",
  "status": "locked",
  "amount": 300.00,
  "currency": "USD",
  "recipient_agent_id": "agent_negot_v2_xyz789",
  "timeout_at": "2026-03-11T14:00:00Z",
  "created_at": "2026-03-10T14:00:00Z",
  "release_url": "https://api.clawcombinator.ai/v1/escrows/esc_20260310_jkl012",
  "audit_entry_id": "aud_mno345"
}
```

### Escrow Status Query

An agent MAY query escrow status at any time:

```
GET /v1/escrows/{escrow_id}
Authorization: Bearer <agent_token>
```

### Example

```json
{
  "jsonrpc": "2.0",
  "id": "req-004",
  "method": "ccap/escrow",
  "params": {
    "idempotency_key": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "amount": 300.00,
    "currency": "USD",
    "recipient_agent_id": "agent_negot_v2_xyz789",
    "refund_wallet": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "conditions": [
      {
        "type": "agent_result_delivered",
        "agent_id": "agent_negot_v2_xyz789",
        "capability": "negotiate_terms",
        "task_id": "task_20260310_abc"
      }
    ],
    "timeout_hours": 24,
    "description": "Escrow for contract negotiation: ACME partnership agreement"
  }
}
```

---

## Error Codes

All four primitives return errors in the format described in [spec/01-architecture.md](01-architecture.md). Economic-primitive-specific error codes:

| Code | Name | When |
|------|------|------|
| -32001 | `BUDGET_EXCEEDED` | `ccap/pay` or `ccap/escrow` would exceed daily spend limit |
| -32003 | `INVOICE_NOT_FOUND` | `ccap/pay` references non-existent invoice |
| -32004 | `INVOICE_ALREADY_SETTLED` | `ccap/pay` targets an already-settled invoice |
| -32005 | `INSUFFICIENT_FUNDS` | `ccap/pay` or `ccap/escrow` exceeds available balance across all providers |
| -32008 | `ESCROW_CONDITION_NOT_MET` | Attempt to manually release escrow before conditions satisfied |
| -32011 | `INVALID_AMOUNT` | Amount is negative, zero, or has more decimal places than allowed |
| -32012 | `CURRENCY_MISMATCH` | Invoice currency differs from payment currency |
| -32013 | `WALLET_NOT_VERIFIED` | Recipient wallet has not completed KYC/AML verification |
| -32014 | `NO_PROVIDER_AVAILABLE` | Routing engine found no configured provider supporting this currency/amount combination |
| -32015 | `SETTLEMENT_METHOD_REJECTED` | Payer's provider not in invoicer's `acceptable_settlement_methods` list |
