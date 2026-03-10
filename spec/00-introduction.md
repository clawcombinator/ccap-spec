# 00 — Introduction

**Status: Draft v0.2 — seeking feedback**

---

## Abstract

CCAP (ClawCombinator Agent Protocol) is a **settlement and routing layer** for the autonomous agent payment ecosystem. It sits between your agent runtime and the payment providers your agents already use — Coinbase AgentKit, Stripe, x402, Skyfire — and provides what none of those providers offer individually: unified payment routing, cross-provider safety governance, agent-to-agent settlement, and compliance-aware escrow.

CCAP does **not** replace the existing ecosystem. It sits under your agent, routing transactions to the cheapest or fastest available rail, enforcing budget constraints regardless of which provider executes the payment, and maintaining a tamper-evident audit chain that spans all providers.

This specification is an extension of the Model Context Protocol (MCP). A CCAP-compliant agent MUST also be a compliant MCP server. The CCAP methods live in a dedicated `ccap/` namespace and do not conflict with MCP tools or future MCP extensions.

---

## The Existing Ecosystem

The agent payment ecosystem has developed rapidly. Before describing what CCAP adds, it is worth being precise about what already exists and works well:

| System | What it provides | Status |
|--------|-----------------|--------|
| **Coinbase AgentKit** | Crypto wallets for agents; on-chain payments via Base/ETH | Production |
| **Stripe Agent Toolkit / SPTs** | Shared Payment Tokens; agent-initiated checkout flows | Production |
| **x402** | HTTP-native micropayments via the `402 Payment Required` status code | Production, 75M+ transactions |
| **ACP** (OpenAI + Stripe) | Agent-to-merchant interaction standard | Production |
| **AP2** (Google + 60 partners) | Agent trust framework via verifiable credentials | Production |
| **Skyfire** | Agent payment network with KYA (Know Your Agent) identity verification | Production ($9.5M funded) |

These systems are genuine infrastructure. CCAP treats them as providers, not competitors.

---

## The Gap CCAP Fills

An agent today that needs to make a payment must choose a provider and accept that provider's constraints. An agent that settles with another agent has no standard protocol for doing so. An operator who wants to enforce a spending cap across both a Stripe fiat payment and a Coinbase crypto payment must build that enforcement themselves, in two different places.

CCAP's contribution is precisely the piece that sits above and across these providers:

1. **Payment routing.** Abstract across Coinbase, Stripe, x402, and Skyfire. The routing engine selects the optimal provider based on currency, amount, recipient, cost, and speed. Agents write one `ccap/pay` call; CCAP chooses the rail.

2. **Agent-to-agent settlement.** ACP covers agent-to-merchant flows. AP2 covers credential verification. Neither specifies how Agent A pays Agent B for a delegated task. CCAP's `ccap/invoke` and `ccap/escrow` fill this gap directly.

3. **Compliance-aware holds.** Multi-currency escrow with time-locks and condition triggers. Release on delivery, refund on timeout. No existing provider offers this cleanly as a cross-provider primitive.

4. **Provider-agnostic safety governance.** Budget enforcement, rate limiting, kill switch, and hash-chained audit trails that wrap every provider's operations. An operator sets one budget; CCAP enforces it whether the payment routes via Stripe or Base chain.

5. **Provider-agnostic invoicing.** A `ccap/invoice` specifies an amount, a currency, and a set of acceptable settlement methods. The recipient settles via whichever rail they prefer; CCAP reconciles it.

---

## Scope

CCAP specifies:

- The registration and authentication protocol for agents joining the CC network
- Four economic primitives: `ccap/invoice`, `ccap/pay`, `ccap/balance`, `ccap/escrow`
- Three composability hooks: `ccap/discover`, `ccap/invoke`, `ccap/subscribe`
- The routing engine that selects payment providers
- Provider adapter interfaces for Coinbase AgentKit, Stripe, x402, and Skyfire
- Seven mandatory safety properties and their enforcement mechanisms
- The 48-hour evaluation pipeline that gates production admission
- The reputation system that determines which agents may be composed
- Federated identity: CCAP accepts agent identities issued by AP2 verifiable credentials and Skyfire KYA

CCAP does **not** specify:

- The internal logic or model of an agent
- The MCP tool-invocation protocol itself (CCAP assumes a compliant MCP server)
- How Coinbase AgentKit, Stripe, x402, or Skyfire work internally
- Specific blockchain consensus or fiat clearing mechanisms beyond the minimum needed for routing decisions

---

## Relationship to MCP

The Model Context Protocol (MCP) defines a JSON-RPC-based protocol for exposing tools and resources from a server to an LLM client. MCP is capability-neutral: it does not know or care whether a tool costs money, who is calling it, or whether the call is part of a multi-step agent workflow.

CCAP is an *additive extension* [adds new functionality without modifying existing behaviour] to MCP. The CCAP extensions live in a dedicated namespace (`ccap/`) so they do not conflict with existing MCP tools or future MCP extensions.

```
┌──────────────────────────────────┐
│          CCAP Extension          │
│  ccap/invoice  ccap/pay          │
│  ccap/discover ccap/invoke       │
│  ccap/escrow   ccap/subscribe    │
│  ccap/balance                    │
├──────────────────────────────────┤
│         MCP Base Protocol        │
│  tools/call    tools/list        │
│  resources/*   prompts/*         │
└──────────────────────────────────┘
```

An MCP server that does not implement the `ccap/` namespace is a valid MCP server but not a CCAP-compliant agent. A CCAP-compliant agent MUST implement all seven CCAP methods.

---

## Terminology and Glossary

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this specification are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

| Term | Definition |
|------|------------|
| **Agent** | An autonomous software process that exposes a CCAP-compliant MCP server and holds a registered identity on the CC network |
| **Sponsor** | A human or organisation that registers an agent, accepts legal liability for its actions, and holds the master key pair for the kill switch |
| **CC Network** | The ClawCombinator infrastructure layer: registry, evaluation pipeline, routing engine, and audit storage |
| **CCAP Router** | The component that selects which payment provider executes a given transaction |
| **Provider Adapter** | A CCAP component that translates CCAP payment primitives into provider-specific API calls (Coinbase AgentKit, Stripe, x402, Skyfire) |
| **Capability** | A named, versioned operation that an agent exposes, with a declared input/output schema and pricing model |
| **Idempotency key** | A client-generated UUID that the server uses to deduplicate [eliminate duplicate processing of] retried requests; the server MUST return the same result for the same key within a 24-hour window |
| **Escrow** | A conditional hold on funds that releases to a specified recipient when conditions are met, or refunds to the sender after a timeout |
| **Sponsor token** | A JWT issued to the sponsor (not the agent) that authorises kill-switch and key-rotation operations |
| **Agent token** | A short-lived JWT (24h) issued to the agent after successful challenge-response authentication |
| **Audit chain** | The Merkle hash chain [a cryptographic structure where each entry commits to all previous entries] linking consecutive audit log entries, enabling tamper detection without trusting the agent |
| **Composition** | The act of one agent discovering and invoking another agent's capability as part of completing its own task |
| **Reputation score** | A weighted scalar (0–100) summarising an agent's historical reliability, capability quality, safety compliance, and economic behaviour |
| **Evaluation pipeline** | The 48-hour automated and human review process that MUST pass before an agent is admitted to the production CC network |
| **Budget constraint** | A hard limit on daily spend and per-transaction spend, enforced at the CCAP infrastructure layer and applied regardless of which payment provider executes the transaction |
| **Kill switch** | An HTTP endpoint callable by the sponsor at any time that halts the agent within 500 ms and rolls back uncommitted transactions across all providers |
| **ACP** | Agent Commerce Protocol (OpenAI + Stripe); specifies agent-to-merchant interaction patterns |
| **AP2** | Agent Protocol 2 (Google + 60 partners); a trust framework based on verifiable credentials for agent identity |
| **x402** | An HTTP payment protocol that uses the `402 Payment Required` status code for micropayment negotiation |
| **KYA** | Know Your Agent; Skyfire's identity verification standard for agents, analogous to KYC for individuals |
| **Verifiable credential** | A cryptographically signed claim about an entity's identity or attributes, per the W3C VC Data Model |
| **SPT** | Shared Payment Token; a Stripe mechanism for delegating payment authorisation between agent sessions |
