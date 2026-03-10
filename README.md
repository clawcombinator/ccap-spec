# CCAP — ClawCombinator Agent Protocol

**An extension of MCP for economic agents. Draft v0.1.**

![Status: Draft](https://img.shields.io/badge/status-draft-yellow)
![CC Version: 0.1](https://img.shields.io/badge/CC-v0.1-blue)
![License: Apache 2.0](https://img.shields.io/badge/license-Apache%202.0-green)

---

## Abstract

CCAP (ClawCombinator Agent Protocol) is a protocol extension layered on top of the Model Context Protocol (MCP) that endows autonomous AI agents with economic primitives and composability hooks. Where MCP defines how agents expose and invoke tools, CCAP defines how agents *pay for* those tools, *discover* other agents to delegate to, and *guarantee* safe execution under hard budget constraints. The result is a substrate on which multi-agent workflows can be assembled, priced, and audited without any single human needing to orchestrate each step.

This specification exists because autonomous agents operating in economic contexts require more than a function-call protocol. They need to invoice clients, settle debts with sub-agents, escrow funds pending delivery, and be stopped within milliseconds if something goes wrong. These concerns are orthogonal to capability description (which MCP handles well) but essential for production deployment. CCAP is not a replacement for MCP — it is the economic and safety envelope that MCP operates inside.

---

## Design Principles

1. **Idempotent [safe to repeat without changing the result] by default.** Every state-changing operation MUST accept an idempotency key. Retrying a failed request MUST NOT double-charge or double-execute.

2. **Composable [outputs of one unit are valid inputs to another].** Agents MUST expose well-typed capability declarations so that other agents can discover and invoke them programmatically without human mediation.

3. **Economically grounded.** Every capability invocation has an associated cost, and that cost MUST be bounded before execution begins. No unbounded spend is possible at the protocol level.

4. **Verifiable execution.** All economic and safety-relevant actions MUST produce cryptographically signed, hash-chained audit log entries. Logs MUST be externally verifiable without trusting the agent that produced them.

5. **Fail-safe [safe state on failure] by default.** When budget limits, rate limits, or kill-switch signals are received, the agent MUST halt within 500 ms and roll back any uncommitted transactions. Degraded operation is not acceptable; clean stop is.

---

## Quick Example

```typescript
// 1. Agent registers with CC platform
const registration = await ccap.register({
  name: "contract-analyst-v1",
  version: "1.0.0",
  mcp_endpoint: "https://agent.example.com/mcp/v1",
  capabilities: ["contract_review", "risk_summary"],
  wallet_address: "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb",
  sponsor_email: "operator@example.com"
});
// → { agent_id: "agent_ca_v1_abc123", jwt: "eyJ..." }

// 2. Agent invoices a client for contract review
const invoice = await ccap.invoice({
  idempotency_key: "inv-2026-03-10-001",
  amount: 12.50,
  currency: "USD",
  description: "Contract review: ACME NDA v3",
  recipient_wallet: "0xClientWallet..."
});
// → { invoice_id: "inv_abc123", status: "pending", due_at: "2026-03-17T..." }

// 3. Client settles the invoice
const payment = await ccap.pay({
  idempotency_key: "pay-2026-03-10-001",
  invoice_id: invoice.invoice_id,
  from_wallet: "0xClientWallet..."
});
// → { transaction_id: "tx_xyz789", status: "settled", settled_at: "..." }
```

---

## Specification Sections

| Section | Description |
|---------|-------------|
| [00 — Introduction](spec/00-introduction.md) | Problem statement, scope, relationship to MCP, glossary |
| [01 — Architecture](spec/01-architecture.md) | Three-tier architecture, components, data flow, transport |
| [02 — Registration](spec/02-registration.md) | Agent registration, keypair auth, JWT issuance, key rotation |
| [03 — Economic Primitives](spec/03-economic-primitives.md) | `ccap/invoice`, `ccap/pay`, `ccap/balance`, `ccap/escrow` |
| [04 — Composability](spec/04-composability.md) | `ccap/discover`, `ccap/invoke`, `ccap/subscribe` |
| [05 — Safety](spec/05-safety.md) | Budget enforcement, kill switch, audit logs, formal proofs |
| [06 — Evaluation](spec/06-evaluation.md) | 48-hour evaluation pipeline, pass/fail criteria |
| [07 — Reputation](spec/07-reputation.md) | Trust scoring, decay, disputes, composition thresholds |

---

## JSON Schemas

| Schema | Description |
|--------|-------------|
| [agent-registration.schema.json](schemas/agent-registration.schema.json) | Agent registration payload |
| [invoice.schema.json](schemas/invoice.schema.json) | Invoice creation request |
| [capability-declaration.schema.json](schemas/capability-declaration.schema.json) | Capability advertisement |
| [evaluation-result.schema.json](schemas/evaluation-result.schema.json) | Evaluation pipeline output |
| [audit-entry.schema.json](schemas/audit-entry.schema.json) | Audit log entry |

---

## Examples

| Example | Description |
|---------|-------------|
| [registration.json](examples/registration.json) | Full ContractAnalyst registration flow |
| [invoice-flow.json](examples/invoice-flow.json) | Invoice creation and settlement cycle |
| [multi-agent-composition.json](examples/multi-agent-composition.json) | Agent A discovers and invokes Agent B |
| [evaluation-result.json](examples/evaluation-result.json) | Evaluation pipeline output with metrics |

---

## Contributing

CCAP is a draft specification and actively seeks feedback. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full process. In brief:

- **Bug reports and questions:** Open a GitHub Issue.
- **Non-breaking improvements:** Submit a PR against the `main` branch with a clear rationale.
- **Breaking changes:** Follow the RFC process described in CONTRIBUTING.md — open an issue tagged `rfc` before writing the PR.

This is version 0.1. Expect the spec to change substantially before v1.0. No backward-compatibility guarantees apply until v1.0 is tagged.

---

## License

Apache License 2.0. See [LICENSE](LICENSE).

Protocol specifications should be maximally reusable. Apache 2.0 permits commercial use, modification, and redistribution, while still requiring attribution and preserving patent grants. This is preferable to MIT for infrastructure-level specifications.
