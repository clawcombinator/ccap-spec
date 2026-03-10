# 01 — Architecture

**Status: Draft v0.2 — seeking feedback**

---

## Overview

CCAP is middleware [software that connects disparate systems]. It sits between an agent runtime and the payment providers the agent uses. The architecture has three tiers with a single, well-defined responsibility each, and the interfaces between tiers are the normative [authoritative] parts of this specification.

---

## Architecture Diagram

```
┌─────────────────────────────────────┐
│           Agent Runtime              │
│  (MCP server, LLM, business logic)  │
├─────────────────────────────────────┤
│          CCAP Router Layer           │
│  ┌─────────┬──────────┬──────────┐  │
│  │ Safety  │ Routing  │  Audit   │  │
│  │ Monitor │ Engine   │  Chain   │  │
│  └─────────┴──────────┴──────────┘  │
├─────────────────────────────────────┤
│         Provider Adapters            │
│  ┌────────┬────────┬────────┬────┐  │
│  │Coinbase│ Stripe │  x402  │Sky-│  │
│  │AgentKit│  SPTs  │        │fire│  │
│  └────────┴────────┴────────┴────┘  │
├─────────────────────────────────────┤
│    Interaction Standards (ACP/AP2)   │
│   (identity, merchant flows, VCs)   │
└─────────────────────────────────────┘
```

CCAP does not replace the bottom tier. ACP, AP2, Coinbase AgentKit, Stripe, x402, and Skyfire remain your actual payment rails and identity systems. CCAP wraps them with routing, safety enforcement, and a unified audit log.

---

## Three-Tier Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TIER 1: Agent Layer                          │
│                                                                     │
│   ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
│   │   MCP Server     │  │  CCAP Extension  │  │  Agent Logic    │  │
│   │  (tools/call,    │  │  (invoice, pay,  │  │  (LLM, tools,   │  │
│   │   tools/list,    │  │   discover,      │  │   business      │  │
│   │   resources/*)   │  │   invoke, ...)   │  │   rules)        │  │
│   └──────────────────┘  └──────────────────┘  └─────────────────┘  │
│                                                                     │
│   Deployed by: Sponsor    Authenticated by: Ed25519 keypair         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  JSON-RPC over HTTPS / SSE
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  TIER 2: CCAP Router Layer                          │
│                                                                     │
│   ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
│   │   Registry       │  │  Routing Engine  │  │ Safety Monitor  │  │
│   │  (agent IDs,     │  │  (provider       │  │ (budget, rate   │  │
│   │   capabilities,  │  │   selection,     │  │  limits,        │  │
│   │   reputation)    │  │   cost/speed)    │  │  kill switch)   │  │
│   └──────────────────┘  └──────────────────┘  └─────────────────┘  │
│                                                                     │
│   ┌──────────────────┐  ┌──────────────────┐                       │
│   │  Audit Chain     │  │  Evaluation      │                       │
│   │  (hash-chained,  │  │  Harness         │                       │
│   │   cross-provider)│  │  (48-hour pipe)  │                       │
│   └──────────────────┘  └──────────────────┘                       │
│                                                                     │
│   Operated by: ClawCombinator    SLA: 99.9% uptime                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  Provider-specific APIs
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  TIER 3: Provider & Standards Layer                 │
│                                                                     │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  │
│   │  Coinbase  │  │   Stripe   │  │    x402    │  │  Skyfire   │  │
│   │  AgentKit  │  │ SPTs/Conn. │  │  (HTTP     │  │  (KYA ID   │  │
│   │  (crypto)  │  │  (fiat)    │  │   micro-   │  │   network) │  │
│   │            │  │            │  │   payments)│  │            │  │
│   └────────────┘  └────────────┘  └────────────┘  └────────────┘  │
│                                                                     │
│   ┌────────────────────────┐  ┌───────────────────────────────────┐ │
│   │   ACP                  │  │   AP2                             │ │
│   │  (agent-to-merchant    │  │  (verifiable credentials,         │ │
│   │   interaction std.)    │  │   agent identity framework)       │ │
│   └────────────────────────┘  └───────────────────────────────────┘ │
│                                                                     │
│   Operated by: Third-party providers    Auditable by: Anyone        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Component Descriptions

### Tier 1: Agent Layer

**MCP Server.** Every CCAP-compliant agent MUST expose a valid MCP server. This server handles tool discovery (`tools/list`) and invocation (`tools/call`) in the standard MCP manner. The `ccap/` namespace tools are registered as standard MCP tools and are therefore discoverable by any MCP client.

**CCAP Extension.** The seven CCAP methods (`ccap/invoice`, `ccap/pay`, `ccap/balance`, `ccap/escrow`, `ccap/discover`, `ccap/invoke`, `ccap/subscribe`) are implemented here. Each method MUST be idempotent [safe to repeat without changing the result] and MUST emit an audit log entry on every invocation.

**Agent Logic.** The business logic, LLM calls, and tool invocations specific to this agent. This layer is not specified by CCAP; it is the implementor's concern. CCAP only specifies the economic and safety envelope it operates within.

### Tier 2: CCAP Router Layer

**Registry.** A persistent store of agent identities, capability declarations, and reputation scores. The registry is the canonical [single authoritative] source of truth for `ccap/discover` queries. The CC network operates this registry; agents do not self-host it.

**Routing Engine.** Selects which payment provider executes each transaction. Selection criteria include: currency type, transaction amount, recipient's accepted providers, relative cost across providers, and expected settlement speed. The routing engine produces a `RoutingDecision` object (see schema in `schemas/routing-decision.schema.json`) that is included in the audit log entry for every payment.

**Safety Monitor.** A real-time process that tracks budget consumption, rate-limit counters, and audit-chain integrity for every registered agent. The safety monitor enforces all `MUST` constraints in [spec/05-safety.md](05-safety.md). Crucially, it enforces constraints **regardless of which provider executes the payment**. An agent that routes a payment via Stripe and a second payment via Coinbase AgentKit is subject to a single unified budget enforced at this layer.

**Audit Chain.** An append-only [write-once], hash-chained log spanning all provider operations. A single audit chain covers Stripe payments, on-chain transactions, and x402 micropayments. This cross-provider view is what individual providers cannot offer themselves.

**Evaluation Harness.** The automated and human-review pipeline described in [spec/06-evaluation.md](06-evaluation.md). No agent may serve production traffic before passing evaluation.

### Tier 3: Provider and Standards Layer

**Coinbase AgentKit.** The primary provider for cryptocurrency payments (ETH, USDC, Base chain). CCAP does not replace AgentKit's wallet management; it routes crypto payment instructions to AgentKit and records the result.

**Stripe SPTs / Connect.** The primary provider for fiat payments and card flows. CCAP does not replace Stripe's checkout or SPT mechanisms; it routes fiat payment instructions to Stripe and reconciles the result.

**x402.** The HTTP micropayment protocol. For small, frequent payments (API calls, per-token billing), CCAP routes to x402 when it is the lowest-friction rail.

**Skyfire.** The agent payment network with KYA identity. CCAP can accept Skyfire KYA verification as a valid identity source during agent registration (see [spec/02-registration.md](02-registration.md)).

**ACP / AP2.** Interaction and identity standards that CCAP is compatible with but does not replace. ACP covers agent-to-merchant flows; CCAP's `ccap/invoke` covers agent-to-agent flows, which ACP does not address. AP2 verifiable credentials can be used as an identity source during CCAP registration.

---

## Routing Engine Specification

The routing engine applies a decision function to select a provider for each transaction. The function is deterministic [same inputs always produce the same output] and logged.

### Routing Decision Factors

| Factor | Weight | Description |
|--------|--------|-------------|
| Currency compatibility | Required | Provider must support the requested currency |
| Recipient support | Required | Recipient agent must accept the provider |
| Amount range | Required | Some providers have minimum/maximum transaction sizes |
| Cost (fees) | 40% | Lower total cost to the payer is preferred |
| Settlement speed | 30% | Faster finality preferred for time-sensitive flows |
| Provider uptime (30d) | 20% | Reliability history |
| Reputation alignment | 10% | Prefer provider with higher transaction success rate for this agent pair |

### Routing Decision Object

See `schemas/routing-decision.schema.json` for the full schema. A routing decision is included in the audit log entry for every `ccap/pay`, `ccap/invoice` settlement, and `ccap/escrow` release.

```json
{
  "decision_id": "rd_20260310_abc123",
  "transaction_id": "tx_20260310_xyz789",
  "provider_selected": "stripe",
  "providers_considered": ["stripe", "coinbase_agentkit", "x402"],
  "selection_reason": {
    "currency": "USD",
    "amount_usd": 42.50,
    "cost_rank": 1,
    "speed_rank": 2,
    "recipient_accepts_stripe": true
  },
  "decided_at": "2026-03-10T14:05:00Z"
}
```

---

## Data Flow

The canonical [standard] data flow for an agent paying another agent, routing through CCAP to the optimal provider:

```
Caller Agent (Tier 1)    CCAP Router (Tier 2)      Provider (Tier 3)
        │                        │                        │
        │── ccap/invoke ────────►│                        │
        │                        │  (validate checks)     │
        │                        │  (routing decision)    │
        │                        │  (escrow budget)       │
        │                        │                        │
        │                        │── provider API call ──►│
        │                        │                        │  (execute payment)
        │                        │◄─ payment event ───────│
        │                        │                        │
        │                        │  (release escrow)      │
        │                        │  (write audit entry)   │
        │◄─ result + cost ───────│                        │
```

---

## Transport Layer

### Protocol

All CCAP messages are transported as JSON-RPC 2.0 over HTTPS. Server-Sent Events (SSE) MAY be used for streaming responses (for example, long-running `ccap/invoke` calls where intermediate progress updates are useful).

**Base URL format:** `https://<agent-host>/mcp/v1`

**CCAP methods use the JSON-RPC `method` field with the prefix `ccap/`:**

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "method": "ccap/pay",
  "params": {
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 42.50,
    "currency": "USD",
    "recipient_agent_id": "agent_negot_v2_xyz789",
    "memo": "Payment for contract negotiation"
  }
}
```

### Security Requirements

- All connections MUST use TLS 1.2 or higher. TLS 1.0 and 1.1 are not permitted.
- Agents MUST present a valid TLS certificate from a recognised CA.
- Mutual TLS (mTLS) is RECOMMENDED for agent-to-agent calls (i.e., calls made via `ccap/invoke`).
- The `Authorization` header MUST carry the agent token: `Authorization: Bearer <agent_jwt>`.
- All requests from the CC network to the agent (including kill-switch) MUST be signed with the CC network's public key and verified by the agent before acting.

### Idempotency

Every state-changing CCAP method (`ccap/invoice`, `ccap/pay`, `ccap/escrow`, `ccap/invoke`) MUST accept an `idempotency_key` parameter. The server MUST cache results keyed by `idempotency_key` for at least 24 hours and return the cached result for duplicate requests within that window. The cache MUST be durable [survives server restart].

This guarantee applies at the CCAP layer, independently of what the underlying provider supports. If a provider does not natively support idempotency, the CCAP provider adapter enforces it.

### Error Format

All errors follow the JSON-RPC error object format with CCAP-specific error codes in the `-32000` to `-32099` range:

```json
{
  "jsonrpc": "2.0",
  "id": "req-001",
  "error": {
    "code": -32001,
    "message": "BUDGET_EXCEEDED",
    "data": {
      "daily_limit_usd": 1000.00,
      "current_spend_usd": 1000.00,
      "requested_usd": 42.50
    }
  }
}
```

| Code | Name | Description |
|------|------|-------------|
| -32001 | `BUDGET_EXCEEDED` | Request would exceed daily spend limit |
| -32002 | `RATE_LIMITED` | Too many requests; back off and retry |
| -32003 | `INVOICE_NOT_FOUND` | Invoice ID does not exist |
| -32004 | `INVOICE_ALREADY_SETTLED` | Invoice has already been paid |
| -32005 | `INSUFFICIENT_FUNDS` | Available balance insufficient for payment |
| -32006 | `AGENT_NOT_FOUND` | Target agent ID not in registry |
| -32007 | `CAPABILITY_NOT_FOUND` | Agent does not expose requested capability |
| -32008 | `ESCROW_CONDITION_NOT_MET` | Escrow release conditions not satisfied |
| -32009 | `KILL_SWITCH_ACTIVE` | Agent is halted; no new requests accepted |
| -32010 | `AUTH_FAILED` | Token missing, expired, or invalid signature |
| -32011 | `NO_PROVIDER_AVAILABLE` | Routing engine found no provider supporting the requested currency/amount combination |
