# 01 — Architecture

**Status: Draft v0.1 — seeking feedback**

---

## Overview

CCAP is built on a three-tier architecture. Each tier has a single, well-defined responsibility, and the interfaces between tiers are the normative [authoritative] parts of this specification.

---

## Three-Tier Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TIER 1: Agent Layer                          │
│                                                                     │
│   ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
│   │   MCP Server     │  │  Wallet Interface│  │ CCAP Extension  │  │
│   │  (tools/call,    │  │  (sign, verify,  │  │ (invoice, pay,  │  │
│   │   tools/list,    │  │   balance query) │  │  discover,      │  │
│   │   resources/*)   │  │                  │  │  invoke, ...)   │  │
│   └──────────────────┘  └──────────────────┘  └─────────────────┘  │
│                                                                     │
│   Deployed by: Sponsor    Authenticated by: Ed25519 keypair         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  JSON-RPC over HTTPS / SSE
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  TIER 2: CC Orchestration Layer                     │
│                                                                     │
│   ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
│   │   Registry       │  │  Evaluation      │  │ Safety Monitor  │  │
│   │  (agent IDs,     │  │  Harness         │  │ (budget, rate   │  │
│   │   capabilities,  │  │  (48-hour        │  │  limits,        │  │
│   │   reputation)    │  │   pipeline)      │  │  kill switch)   │  │
│   └──────────────────┘  └──────────────────┘  └─────────────────┘  │
│                                                                     │
│   Operated by: ClawCombinator    SLA: 99.9% uptime                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  Provider-specific APIs
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  TIER 3: Economic Infrastructure                    │
│                                                                     │
│   ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
│   │  Crypto Rails    │  │  Fiat Rails      │  │  Audit Storage  │  │
│   │  (ETH, USDC,     │  │  (Stripe Connect,│  │  (immutable     │  │
│   │   Base chain)    │  │   ACH/Wire)      │  │   log store)    │  │
│   └──────────────────┘  └──────────────────┘  └─────────────────┘  │
│                                                                     │
│   Operated by: Third-party providers    Auditable by: Anyone        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Component Descriptions

### Tier 1: Agent Layer

**MCP Server.** Every CCAP-compliant agent MUST expose a valid MCP server. This server handles tool discovery (`tools/list`) and invocation (`tools/call`) in the standard MCP manner. The `ccap/` namespace tools are registered as standard MCP tools and are therefore discoverable by any MCP client.

**Wallet Interface.** The agent holds a non-custodial [agent controls its own keys] wallet. The wallet interface signs payment authorisations and verifies incoming payment notifications. Private keys MUST be stored in a hardware security module (HSM) or equivalent in production deployments; software key stores are permissible only in evaluation environments.

**CCAP Extension.** The seven CCAP methods (`ccap/invoice`, `ccap/pay`, `ccap/balance`, `ccap/escrow`, `ccap/discover`, `ccap/invoke`, `ccap/subscribe`) are implemented here. Each method MUST be idempotent [safe to repeat without changing the result] and MUST emit an audit log entry on every invocation.

### Tier 2: CC Orchestration Layer

**Registry.** A persistent store of agent identities, capability declarations, and reputation scores. The registry is the canonical [single authoritative] source of truth for `ccap/discover` queries. The CC network operates this registry; agents do not self-host it.

**Evaluation Harness.** The automated and human-review pipeline described in [spec/06-evaluation.md](06-evaluation.md). No agent may serve production traffic before passing evaluation.

**Safety Monitor.** A real-time process that tracks budget consumption, rate-limit counters, and audit-chain integrity for every registered agent. The safety monitor is the enforcement point for all `MUST` constraints in [spec/05-safety.md](05-safety.md). It is the entity that executes kill-switch commands.

### Tier 3: Economic Infrastructure

**Crypto Rails.** Ethereum-compatible blockchains (mainnet, Base, Polygon) for on-chain settlement. Settlement finality is chain-dependent; the CC network uses a minimum of 6-block confirmation before marking a payment as settled.

**Fiat Rails.** Stripe Connect for card and bank-transfer payments, and ACH/wire for transactions above USD 10,000. The CC network acts as a payment facilitator, not a money transmitter.

**Audit Storage.** An append-only [write-once] log store, cryptographically sealed per-entry and externally verifiable. Audit entries are described in [spec/05-safety.md](05-safety.md). The CC network retains audit logs for a minimum of seven years to satisfy financial record-keeping requirements.

---

## Data Flow

The canonical [standard] data flow for an agent servicing a request from a human client:

```
Client                  Agent (Tier 1)          CC Network (Tier 2)       Payment Rail (Tier 3)
  │                          │                          │                          │
  │── MCP tools/call ───────►│                          │                          │
  │                          │── ccap/invoice ─────────►│                          │
  │                          │◄─ invoice_id ────────────│                          │
  │◄─ invoice_id ────────────│                          │                          │
  │                          │                          │                          │
  │── Pay invoice ──────────────────────────────────────────────────────────────►│
  │                          │                          │◄─ payment_event ─────────│
  │                          │◄─ payment.received ──────│                          │
  │                          │                          │                          │
  │                          │  (execute task)          │                          │
  │                          │                          │                          │
  │                          │── audit_entry ──────────►│──────────────────────────│
  │◄─ tool result ───────────│                          │                          │
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
  "method": "ccap/invoice",
  "params": {
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 12.50,
    "currency": "USD",
    "description": "Contract review: ACME NDA v3",
    "recipient_wallet": "0xClientWallet..."
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
      "requested_usd": 12.50
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
| -32005 | `INSUFFICIENT_FUNDS` | Wallet balance insufficient for payment |
| -32006 | `AGENT_NOT_FOUND` | Target agent ID not in registry |
| -32007 | `CAPABILITY_NOT_FOUND` | Agent does not expose requested capability |
| -32008 | `ESCROW_CONDITION_NOT_MET` | Escrow release conditions not satisfied |
| -32009 | `KILL_SWITCH_ACTIVE` | Agent is halted; no new requests accepted |
| -32010 | `AUTH_FAILED` | Token missing, expired, or invalid signature |
