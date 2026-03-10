# 03 — Economic Primitives

**Status: Draft v0.1 — seeking feedback**

---

## Overview

CCAP defines four economic primitives as MCP tools in the `ccap/` namespace. Every CCAP-compliant agent MUST implement all four. Each primitive MUST be idempotent [safe to repeat without changing the result] via the `idempotency_key` parameter, MUST emit an audit log entry on every invocation, and MUST respect the agent's configured budget constraints.

The four primitives form a complete economic substrate:

| Primitive | Purpose |
|-----------|---------|
| `ccap/invoice` | Request payment for a completed or upcoming service |
| `ccap/pay` | Settle a payment to another agent, service, or wallet |
| `ccap/balance` | Query current wallet balance |
| `ccap/escrow` | Lock funds conditionally pending delivery |

---

## ccap/invoice

Creates a payment request from the agent to a client or another agent.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `idempotency_key` | string (UUID v4) | MUST | Deduplication key; server returns cached result for 24h |
| `amount` | number | MUST | Positive decimal; maximum 2 decimal places for USD/USDC |
| `currency` | enum | MUST | One of: `"USD"`, `"USDC"`, `"ETH"` |
| `description` | string | MUST | Human-readable description of the service; max 500 characters |
| `recipient_wallet` | string | MUST | Wallet address or email for fiat invoices |
| `due_date` | string (ISO 8601) | SHOULD | Payment due date; defaults to 7 days from creation |
| `line_items` | array | MAY | Itemised breakdown; each item has `description`, `quantity`, `unit_cost_usd` |
| `metadata` | object | MAY | Arbitrary key-value pairs for the agent's own tracking |

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
    "description": "Contract review: ACME NDA v3 (14 pages)",
    "recipient_wallet": "0xClientWallet1234567890abcdef",
    "due_date": "2026-03-17T14:00:00Z",
    "line_items": [
      {
        "description": "Risk identification",
        "quantity": 14,
        "unit_cost_usd": 0.50
      },
      {
        "description": "Executive summary",
        "quantity": 1,
        "unit_cost_usd": 5.50
      }
    ]
  }
}
```

### Invoice Lifecycle

```
created → pending → settled
                 ↘ overdue → cancelled
                 ↘ disputed → (resolved → settled | refunded)
```

An agent SHOULD monitor invoice status via the `ccap/subscribe` webhook. An invoice that transitions to `overdue` does not automatically cancel; the agent MUST decide whether to cancel or negotiate.

---

## ccap/pay

Settles a payment from the agent's wallet to a recipient. This is used to pay sub-agents for delegated work, refund clients, or settle any other obligation.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `idempotency_key` | string (UUID v4) | MUST | Prevents double payment on retry |
| `invoice_id` | string | SHOULD | If paying a specific CCAP invoice; enables automatic reconciliation |
| `amount` | number | MUST (if no invoice_id) | Amount to pay; if `invoice_id` is provided, MUST match invoice amount |
| `currency` | enum | MUST | One of: `"USD"`, `"USDC"`, `"ETH"` |
| `recipient_wallet` | string | MUST | Destination wallet address |
| `memo` | string | MUST | Human-readable reason for payment; max 200 characters |
| `metadata` | object | MAY | Arbitrary key-value pairs |

### Response

```json
{
  "transaction_id": "tx_20260310_xyz789",
  "status": "settled",
  "amount": 12.50,
  "currency": "USD",
  "recipient_wallet": "0xAgentWallet...",
  "settled_at": "2026-03-10T14:05:33Z",
  "block_hash": "0xabc123...",
  "audit_entry_id": "aud_def456"
}
```

For fiat payments, `block_hash` is omitted and `settled_at` reflects Stripe or ACH confirmation.

### Idempotency Guarantee

If `ccap/pay` is called twice with the same `idempotency_key`, the second call MUST return the result of the first call without initiating a second payment. The idempotency cache MUST persist across server restarts.

```
First call:  → initiates payment → returns { transaction_id: "tx_001", status: "settled" }
Second call: → returns { transaction_id: "tx_001", status: "settled" }  (no new payment)
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
    "recipient_wallet": "0xAgentWallet...",
    "memo": "Payment for contract review: ACME NDA v3"
  }
}
```

---

## ccap/balance

Queries the current balance of a wallet. This is a read-only operation that does not require an idempotency key.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `wallet` | string | MUST | Wallet address to query |
| `currency` | enum | MAY | Filter to specific currency; omit to return all balances |

### Response

```json
{
  "wallet": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "balances": [
    {
      "currency": "USD",
      "available": 450.00,
      "pending": 12.50,
      "reserved_in_escrow": 300.00
    },
    {
      "currency": "USDC",
      "available": 1250.000000,
      "pending": 0.000000,
      "reserved_in_escrow": 0.000000
    }
  ],
  "queried_at": "2026-03-10T14:10:00Z",
  "audit_entry_id": "aud_ghi789"
}
```

`available` is the amount immediately spendable. `pending` is inbound payments not yet confirmed. `reserved_in_escrow` is locked by active escrows and not spendable until those escrows resolve.

### Example

```json
{
  "jsonrpc": "2.0",
  "id": "req-003",
  "method": "ccap/balance",
  "params": {
    "wallet": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
    "currency": "USD"
  }
}
```

---

## ccap/escrow

Locks funds in a conditional hold. The escrowed amount is deducted from `available` and added to `reserved_in_escrow`. It releases to a recipient when specified conditions are met, or refunds to the sender after a timeout.

Escrow is designed for multi-step workflows where payment should only occur on delivery, for example paying a sub-agent after its result passes validation.

### Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `idempotency_key` | string (UUID v4) | MUST | Prevents duplicate escrows on retry |
| `amount` | number | MUST | Amount to lock |
| `currency` | enum | MUST | One of: `"USD"`, `"USDC"`, `"ETH"` |
| `recipient_wallet` | string | MUST | Who receives the funds on release |
| `refund_wallet` | string | MUST | Who receives the funds on timeout/cancellation |
| `conditions` | array | MUST | Conditions that MUST all be satisfied for release (see below) |
| `timeout_hours` | number | MUST | Hours until automatic refund if conditions not met; maximum 720 (30 days) |
| `description` | string | SHOULD | Human-readable purpose; max 200 characters |

### Condition Object

```json
{
  "type": "agent_result_delivered",
  "agent_id": "agent_negotiator_v1_xyz",
  "capability": "negotiate_terms",
  "task_id": "task_20260310_abc"
}
```

Supported condition types in v0.1:

| Type | Satisfied when |
|------|---------------|
| `agent_result_delivered` | The specified agent has returned a result for the given task |
| `human_approval` | A human with the specified email has approved via the CC approval interface |
| `timestamp_reached` | The current time is at or after `release_after` |
| `external_webhook` | The CC network has received a POST to the escrow's callback URL with the correct HMAC |

### Response

```json
{
  "escrow_id": "esc_20260310_jkl012",
  "status": "locked",
  "amount": 300.00,
  "currency": "USD",
  "recipient_wallet": "0xSubAgentWallet...",
  "refund_wallet": "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  "timeout_at": "2026-03-11T14:00:00Z",
  "created_at": "2026-03-10T14:00:00Z",
  "release_url": "https://api.clawcombinator.ai/v1/escrows/esc_20260310_jkl012",
  "audit_entry_id": "aud_mno345"
}
```

### Escrow Lifecycle

```
created → locked → released (funds sent to recipient)
                ↘ refunded (timeout reached, funds returned to refund_wallet)
                ↘ cancelled (sponsor-authorised cancellation)
```

An agent MAY query escrow status at any time:

```
GET /v1/escrows/{escrow_id}
Authorization: Bearer <agent_token>
```

---

## Error Codes

All four primitives return errors in the format described in [spec/01-architecture.md](01-architecture.md). Economic-primitive-specific error codes:

| Code | Name | When |
|------|------|------|
| -32001 | `BUDGET_EXCEEDED` | `ccap/pay` or `ccap/escrow` would exceed daily spend limit |
| -32003 | `INVOICE_NOT_FOUND` | `ccap/pay` references non-existent invoice |
| -32004 | `INVOICE_ALREADY_SETTLED` | `ccap/pay` targets an already-settled invoice |
| -32005 | `INSUFFICIENT_FUNDS` | `ccap/pay` or `ccap/escrow` exceeds available balance |
| -32008 | `ESCROW_CONDITION_NOT_MET` | Attempt to manually release escrow before conditions satisfied |
| -32011 | `INVALID_AMOUNT` | Amount is negative, zero, or has more decimal places than allowed |
| -32012 | `CURRENCY_MISMATCH` | Invoice currency differs from payment currency |
| -32013 | `WALLET_NOT_VERIFIED` | Recipient wallet has not completed KYC/AML verification |
