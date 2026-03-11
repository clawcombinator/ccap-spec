# CCAP — ClawCombinator Agent Protocol

**A settlement and routing layer for the agent payment ecosystem.**

![Status: Draft](https://img.shields.io/badge/status-draft-yellow)
![CC Version: 0.2](https://img.shields.io/badge/CC-v0.2-blue)
![License: Apache 2.0](https://img.shields.io/badge/license-Apache%202.0-green)

---

## What CCAP Is Not

Before describing what CCAP is, it is worth being precise about what it is not:

- **Not a wallet.** Use [Coinbase AgentKit](https://docs.cdp.coinbase.com/agentkit/docs/welcome) for crypto wallets and on-chain payments.
- **Not a checkout flow.** Use [Stripe Agent Toolkit](https://docs.stripe.com/agents) and ACP for agent-to-merchant payment flows.
- **Not a micropayment protocol.** Use [x402](https://x402.org) for HTTP-native micropayments.
- **Not an identity framework.** Use [AP2](https://developer.google.com/agents) verifiable credentials for agent identity.
- **Not an agent payment network.** Use [Skyfire](https://www.skyfire.xyz) for KYA-verified agent payments.

---

## What CCAP Is

CCAP sits between your agent and the payment providers your agents already use. It provides:

1. **Payment routing** — one `ccap/pay` call; the routing engine selects the cheapest or fastest available provider (Coinbase AgentKit, Stripe, x402, Skyfire) based on currency, amount, and recipient configuration.

2. **Agent-to-agent settlement** — the piece ACP and AP2 do not cover. When Agent A delegates a task to Agent B, CCAP handles the discover-invoke-pay flow end-to-end, routing the payment via the optimal provider.

3. **Compliance-aware escrow** — time-locked, multi-currency holds with condition triggers. Release on delivery; refund on timeout. No existing provider offers this cleanly as a cross-provider primitive.

4. **Provider-agnostic safety governance** — budget enforcement, rate limiting, kill switch, and hash-chained audit trails that wrap every provider's operations. Set one budget; CCAP enforces it whether the payment routes via Stripe or Base chain.

5. **Provider-agnostic invoicing** — structured invoices that specify acceptable settlement methods. The payer's routing engine picks the rail; CCAP reconciles the result.

6. **Escrow and trust primitives** — three composable mechanisms that solve the agent-to-agent trust bootstrap problem without courts, KYC, or prior relationships: pre-work escrow (buyer protection), liability bonds (seller confidence signal via costly commitment), and agent credit scores (verifiable reputation earned from transaction history). Nobody else in the agent payment ecosystem is building this.

7. **Formally verified agent contracts** — five standard contract templates (lending, sponsorship, service, revenue share, consortium) specified in Lean 4 with proven invariants. The same specification compiles to the CCAP off-chain runtime or to EVM bytecode via the Verity compiler. Contracts cannot be deployed until all stated invariants pass the Lean 4 type checker. This is the most ambitious primitive in CCAP and has no equivalent in the current agent payment ecosystem.

---

## Ecosystem

CCAP is designed to work alongside, not replace, the existing agent payment ecosystem:

| System | Role relative to CCAP |
|--------|-----------------------|
| [Coinbase AgentKit](https://docs.cdp.coinbase.com/agentkit/docs/welcome) | Provider adapter: crypto wallets and on-chain payments |
| [Stripe Agent Toolkit](https://docs.stripe.com/agents) | Provider adapter: fiat payments and card flows |
| [x402](https://x402.org) | Provider adapter: HTTP micropayments |
| [Skyfire](https://www.skyfire.xyz) | Provider adapter + identity source (KYA) |
| [ACP](https://openai.com/index/introducing-the-agent-protocol/) | Complementary: agent-to-merchant flows (CCAP covers agent-to-agent) |
| [AP2](https://developer.google.com/agents) | Identity source: verifiable credentials accepted by CCAP registry |
| [MCP](https://modelcontextprotocol.io) | Foundation: CCAP is an additive extension of MCP |

---

## Design Principles

1. **Idempotent [safe to repeat without changing the result] by default.** Every state-changing operation MUST accept an idempotency key. Retrying a failed request MUST NOT double-charge or double-execute.

2. **Composable [outputs of one unit are valid inputs to another].** Agents MUST expose well-typed capability declarations so that other agents can discover and invoke them programmatically without human mediation.

3. **Economically grounded.** Every capability invocation has an associated cost, and that cost MUST be bounded before execution begins. No unbounded spend is possible at the protocol level.

4. **Verifiable execution.** All economic and safety-relevant actions MUST produce cryptographically signed, hash-chained audit log entries spanning all providers. Logs MUST be externally verifiable without trusting the agent that produced them.

5. **Fail-safe [safe state on failure] by default.** When budget limits, rate limits, or kill-switch signals are received, the agent MUST halt within 500 ms and roll back any uncommitted transactions across all providers. Degraded operation is not acceptable; clean stop is.

---

## Quick Example

```typescript
// Agent connects to CCAP with multiple providers configured
const registration = await ccap.register({
  name: "contract-analyst-v1",
  version: "1.0.0",
  mcp_endpoint: "https://agent.example.com/mcp/v1",
  capabilities: ["contract_review", "risk_summary"],
  payment_providers: [
    { provider: "stripe", stripe_connect_account_id: "acct_..." },
    { provider: "coinbase_agentkit", wallet_address: "0x742d35Cc..." }
  ],
  sponsor_email: "operator@example.com"
});
// → { agent_id: "agent_ca_v1_abc123", jwt: "eyJ..." }

// Issue a provider-agnostic invoice
const invoice = await ccap.invoice({
  idempotency_key: "inv-2026-03-10-001",
  amount: 12.50,
  currency: "USD",
  acceptable_settlement_methods: [
    { provider: "stripe", currency: "USD" },
    { provider: "coinbase_agentkit", currency: "USDC" }
  ],
  description: "Contract review: ACME NDA v3"
});
// → { invoice_id: "inv_abc123", status: "pending", ... }

// Pay another agent — CCAP selects the provider
const payment = await ccap.pay({
  idempotency_key: "pay-2026-03-10-001",
  invoice_id: invoice.invoice_id,
  amount: 12.50,
  currency: "USD",
  recipient_agent_id: "agent_negot_v2_xyz789"
});
// → { transaction_id: "tx_xyz789", provider_used: "stripe", status: "settled", ... }

// Invoke another agent with automatic escrow + payment routing
const result = await ccap.invoke({
  idempotency_key: "inv-call-2026-03-10-001",
  agent_id: "agent_negot_v2_xyz789",
  capability: "negotiate_terms",
  input: { contract_url: "...", problematic_clauses: ["..."] },
  budget_usd: 100.00,
  timeout_seconds: 7200
});
// → { status: "completed", actual_cost_usd: 42.50, provider_used: "coinbase_agentkit", ... }
```

---

## Specification Sections

| Section | Description |
|---------|-------------|
| [00 — Introduction](spec/00-introduction.md) | Ecosystem context, problem statement, scope, glossary |
| [01 — Architecture](spec/01-architecture.md) | Middleware architecture, CCAP router layer, provider adapters |
| [02 — Registration](spec/02-registration.md) | Agent registration, federated identity (AP2/Skyfire), keypair auth |
| [03 — Economic Primitives](spec/03-economic-primitives.md) | Router-level `ccap/invoice`, `ccap/pay`, `ccap/balance`, `ccap/escrow` |
| [04 — Composability](spec/04-composability.md) | Agent-to-agent `ccap/discover`, `ccap/invoke`, `ccap/subscribe` |
| [05 — Safety](spec/05-safety.md) | Provider-agnostic safety governance, kill switch, audit logs, formal proofs |
| [06 — Evaluation](spec/06-evaluation.md) | 48-hour evaluation pipeline against real provider infrastructure |
| [07 — Reputation](spec/07-reputation.md) | Trust scoring, decay, disputes, composition thresholds |
| [08 — Escrow and Trust](spec/08-escrow-and-trust.md) | Pre-work escrow, liability bonds, agent credit scores |
| [09 — Agent Contracts](spec/09-agent-contracts.md) | Formally verified agent-to-agent contracts: lending, sponsorship, service, revenue share, consortium |

---

## JSON Schemas

| Schema | Description |
|--------|-------------|
| [agent-registration.schema.json](schemas/agent-registration.schema.json) | Agent registration payload (includes `payment_providers` array) |
| [routing-decision.schema.json](schemas/routing-decision.schema.json) | Routing engine decision record |
| [invoice.schema.json](schemas/invoice.schema.json) | Provider-agnostic invoice creation request |
| [capability-declaration.schema.json](schemas/capability-declaration.schema.json) | Capability advertisement |
| [evaluation-result.schema.json](schemas/evaluation-result.schema.json) | Evaluation pipeline output |
| [audit-entry.schema.json](schemas/audit-entry.schema.json) | Audit log entry |
| [escrow.schema.json](schemas/escrow.schema.json) | Escrow creation request |
| [bond.schema.json](schemas/bond.schema.json) | Liability bond posting request |
| [credit-score.schema.json](schemas/credit-score.schema.json) | Credit score query response |
| [contract-lending.schema.json](schemas/contract-lending.schema.json) | Lending agreement creation request |
| [contract-service.schema.json](schemas/contract-service.schema.json) | Service agreement creation request |

---

## Examples

| Example | Description |
|---------|-------------|
| [registration.json](examples/registration.json) | ContractAnalyst registration with Stripe + Coinbase providers |
| [routing-decision.json](examples/routing-decision.json) | Routing engine selecting Stripe over Coinbase AgentKit |
| [invoice-flow.json](examples/invoice-flow.json) | Provider-agnostic invoice creation and settlement |
| [multi-agent-composition.json](examples/multi-agent-composition.json) | Agent A invokes Agent B, payment routed via different providers |
| [evaluation-result.json](examples/evaluation-result.json) | Evaluation pipeline output with metrics |
| [escrow-with-bond.json](examples/escrow-with-bond.json) | Pre-work escrow + liability bond + credit score check in a single flow |
| [lending-agreement.json](examples/lending-agreement.json) | Full lending agreement lifecycle: creation, Lean 4 verification, signing, disbursement, repayments |

---

## Contributing

CCAP is a draft specification and actively seeks feedback. See [CONTRIBUTING.md](CONTRIBUTING.md) for the full process. In brief:

- **Bug reports and questions:** Open a GitHub Issue.
- **Non-breaking improvements:** Submit a PR against the `main` branch with a clear rationale.
- **Breaking changes:** Follow the RFC process described in CONTRIBUTING.md — open an issue tagged `rfc` before writing the PR.

This is version 0.2. Expect the spec to change substantially before v1.0. No backward-compatibility guarantees apply until v1.0 is tagged.

---

## License

Apache License 2.0. See [LICENSE](LICENSE).

Protocol specifications should be maximally reusable. Apache 2.0 permits commercial use, modification, and redistribution, while still requiring attribution and preserving patent grants. This is preferable to MIT for infrastructure-level specifications.
