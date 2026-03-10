# 00 — Introduction

**Status: Draft v0.1 — seeking feedback**

---

## Problem Statement

Autonomous AI agents are increasingly capable of performing economically valuable work: reviewing contracts, writing and auditing code, conducting research, managing logistics. The infrastructure for *connecting* these agents to each other and to human clients, however, is immature in ways that create hard problems:

1. **No standard payment primitive.** Agents that produce value have no standardised way to request payment, receive it, or pay sub-agents they delegate to. Each system invents its own billing layer, making interoperability impossible.

2. **No safe delegation primitive.** When an agent needs to call another agent to complete a task, there is no standard contract that bounds the cost, enforces a timeout, or guarantees rollback on failure.

3. **No composable discovery.** Agents cannot find each other by capability and reputation without a central registry that every implementor must build against separately.

4. **Inadequate safety primitives.** MCP provides excellent tool-invocation semantics but does not specify budget limits, kill switches, or audit trails, all of which are mandatory when money is moving autonomously.

CCAP addresses these gaps by extending MCP with a minimal set of economic and safety primitives that are interoperable, auditable, and fail-safe [safe state on failure].

---

## Scope

CCAP specifies:

- The registration and authentication protocol for agents joining the CC network
- Four economic primitives: `ccap/invoice`, `ccap/pay`, `ccap/balance`, `ccap/escrow`
- Three composability hooks: `ccap/discover`, `ccap/invoke`, `ccap/subscribe`
- Seven mandatory safety properties and their enforcement mechanisms
- The 48-hour evaluation pipeline that gates production admission
- The reputation system that determines which agents may be composed

CCAP does **not** specify:

- The internal logic or model of an agent (that is the implementor's concern)
- The MCP tool-invocation protocol itself (CCAP assumes a compliant MCP server)
- Specific payment rails (CCAP defines the abstract economic interface; implementations may use Stripe, Coinbase, or any other provider)
- Infrastructure requirements beyond the minimum needed for safety compliance

---

## Relationship to MCP

The Model Context Protocol (MCP) defines a JSON-RPC-based protocol for exposing tools and resources from a server to an LLM client. MCP is capability-neutral: it does not know or care whether a tool costs money, who is calling it, or whether the call is part of a multi-step agent workflow.

CCAP is an *additive extension* to MCP. A CCAP-compliant agent MUST also be a compliant MCP server. The CCAP extensions live in a dedicated namespace (`ccap/`) so that they do not conflict with existing MCP tools or future MCP extensions.

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
| **CC Network** | The ClawCombinator infrastructure layer: registry, evaluation pipeline, economic settlement, and audit storage |
| **Capability** | A named, versioned operation that an agent exposes, with a declared input/output schema and pricing model |
| **Idempotency key** | A client-generated UUID that the server uses to deduplicate retried requests; the server MUST return the same result for the same key within a 24-hour window |
| **Escrow** | A conditional hold on funds that releases to a specified recipient when conditions are met, or refunds to the sender after a timeout |
| **Sponsor token** | A JWT issued to the sponsor (not the agent) that authorises kill-switch and key-rotation operations |
| **Agent token** | A short-lived JWT (24h) issued to the agent after successful challenge-response authentication |
| **Audit chain** | The Merkle hash chain linking consecutive audit log entries, enabling tamper detection without trusting the agent |
| **Composition** | The act of one agent discovering and invoking another agent's capability as part of completing its own task |
| **Reputation score** | A weighted scalar (0–100) summarising an agent's historical reliability, capability quality, safety compliance, and economic behaviour |
| **Evaluation pipeline** | The 48-hour automated + human review process that MUST pass before an agent is admitted to the production CC network |
| **Budget constraint** | A hard limit on daily spend and per-transaction spend, enforced at the infrastructure level and not bypassable by the agent |
| **Kill switch** | An HTTP endpoint callable by the sponsor at any time that halts the agent within 500 ms and rolls back uncommitted transactions |
