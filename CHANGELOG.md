# Changelog

All notable changes to the CCAP specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/). Version numbers follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html). Because this is a protocol specification, version numbers carry additional meaning:

- **Patch** (0.0.x) — editorial changes, clarifications, example fixes. No wire-format changes.
- **Minor** (0.x.0) — additive changes. New optional fields, new `MAY`-level behaviours. Backward-compatible [existing implementations continue to work without changes].
- **Major** (x.0.0) — breaking changes. Removed fields, changed required fields, altered observable behaviour. Requires RFC process.

---

## [Unreleased]

### Added

- **`spec/09-agent-contracts.md`** — formally verified agent-to-agent contracts layer. Five standard contract templates specified in Lean 4 with proven invariants: lending agreement (capital to bootstrap new agents), sponsorship agreement (monthly allowance with spending constraints), service agreement (SLA with automatic penalties), revenue share agreement (investor stake in agent revenue), and consortium agreement (N-agent joint bidding with revenue split). The same Lean 4 spec runs on two runtimes: off-chain (CCAP interpreter, fiat rails) or on-chain (Verity compiler → Solidity → EVM, crypto rails). Contracts cannot be deployed until all stated invariants pass the Lean 4 type checker; a verifiability certificate is issued containing proof hashes and the compiler version used.
- **`ccap/contract/lending/{create,sign,repay,status}`** — lending agreement lifecycle methods. Key invariants proved: total repayment always ≥ principal; lender cannot withdraw during term; default triggers are monotonic.
- **`ccap/contract/sponsorship/{create,sign,terminate}`** — sponsorship agreement lifecycle methods. Key invariants proved: spending within budget, spending within declared categories, fraud termination is immediate and irrevocable.
- **`ccap/contract/service/{create,sign,report_violation}`** — service agreement lifecycle methods. Key invariants proved: price fixed for term, SLA violations trigger automatic penalties.
- **`ccap/contract/revenue_share/{create,sign,report_revenue}`** — revenue share lifecycle methods. Key invariants proved: share percentage between 0 and 100%, total payout ≤ cap, payment records monotonic.
- **`ccap/contract/consortium/{create,sign,vote,admit_member}`** — consortium lifecycle methods. Key invariants proved: revenue split sums to exactly 100%, liability allocation sums to exactly 100%, no unilateral escrow withdrawal.
- **`ccap/contract/[any]/verify_certificate`** — method for independent re-verification of a contract's proof certificate.
- **`schemas/contract-lending.schema.json`** — JSON Schema for lending agreement creation request.
- **`schemas/contract-service.schema.json`** — JSON Schema for service agreement creation request.
- **`examples/lending-agreement.json`** — full lending agreement lifecycle: creation with Lean 4 verification, signing by both parties, disbursement, two repayments, and status query with credit score impact.
- Error codes `-32030` through `-32039` covering contract verification, signing, anti-gaming, and on-chain compilation errors.
- Circular lending detection: the CCAP runtime performs graph traversal over active lending contracts to detect and block circular lending chains (A → B → C → A) at creation time.
- Self-dealing detection for contracts (extends the credit score self-dealing detection in spec/08).
- Wash contract detection: back-to-back mirror contracts between the same two agents are flagged and excluded from credit score computation.

- **`spec/08-escrow-and-trust.md`** — three composable trust primitives for agent-to-agent commerce: pre-work escrow (buyer protection), liability bonds (seller confidence signal via costly commitment), and agent credit scores (verifiable reputation from transaction history). These address the trust bootstrap problem: how do you transact with a counterparty that has no legal standing, no physical address, and no reputation history?
- **`ccap/escrow/create`**, **`ccap/escrow/fund`**, **`ccap/escrow/verify`**, **`ccap/escrow/release`**, **`ccap/escrow/refund`**, **`ccap/escrow/dispute`** — full escrow lifecycle methods. Extends the basic escrow primitive in spec/03 with buyer/seller-specific workflows, completion criteria, and dispute resolution.
- **`ccap/bond/post`**, **`ccap/bond/verify`**, **`ccap/bond/claim`**, **`ccap/bond/release`**, **`ccap/bond/status`** — liability bond methods. Bonds are voluntary costly signals; posting one is only rational if the agent's private estimate of failure probability is low.
- **`ccap/credit/score`** — credit score query returning a 0–1000 score with component breakdown (payment reliability 30%, bond history 25%, transaction volume 20%, dispute rate 15%, counterparty diversity 10%).
- **`schemas/escrow.schema.json`** — JSON Schema for escrow creation request.
- **`schemas/bond.schema.json`** — JSON Schema for liability bond posting request.
- **`schemas/credit-score.schema.json`** — JSON Schema for credit score query response.
- **`examples/escrow-with-bond.json`** — end-to-end example of a trust-stack interaction: credit score check, bond verification, escrow creation, funding, verification, and release.
- Error codes `-32020` through `-32027` covering escrow and bond error conditions.
- Separating equilibrium analysis explaining the game-theoretic basis for bond posting.
- Anti-gaming provisions for credit scores: score decay, self-dealing detection (0.1× weighting for related agents), Sybil resistance (new agents start at 0), and volume velocity limits.

---

## [0.2.0] — 2026-03-10

### Changed

- **Repositioned CCAP as a settlement and routing layer** under existing protocols, not a standalone replacement. The spec now explicitly acknowledges Coinbase AgentKit, Stripe, x402, ACP, AP2, and Skyfire as ecosystem partners.
- **Architecture** (spec/01) rewritten to show CCAP as middleware between the agent runtime and payment providers. New diagram with Provider Adapters tier and Interaction Standards tier.
- **Economic primitives** (spec/03) refactored as router-level operations. `ccap/pay` now routes to the optimal provider rather than assuming a single rail. `ccap/invoice` is now provider-agnostic with `acceptable_settlement_methods`. `ccap/balance` is now aggregated across all providers. `ccap/escrow` is explicitly positioned as CCAP's novel contribution.
- **Registration** (spec/02) now includes `payment_providers` array instead of a single `wallet_address`. Framing changed from "the registry" to "a CCAP registry" (agents may also register with ACP, AP2, Skyfire independently).
- **Safety** (spec/05) framing updated to emphasise provider-agnostic governance. Budget enforcement, kill switch, and audit log now explicitly cover all providers in aggregate.
- **Evaluation** (spec/06) pipeline updated to test against real provider infrastructure. Phase 2 adds payment integration test category. Phase 3 adds provider fallback test. Phase 4 simulation exercises multiple providers.
- **Composability** (spec/04) now explicitly positions `ccap/invoke` as agent-to-agent (the gap ACP does not cover). Acknowledges ACP for agent-to-merchant flows.

### Added

- **Routing engine specification** in spec/01 with provider selection criteria and `RoutingDecision` object.
- **Provider adapter specification** in spec/01 describing how CCAP interfaces with Coinbase AgentKit, Stripe, x402, and Skyfire.
- **Federated identity section** in spec/02: CCAP accepts AP2 verifiable credentials and Skyfire KYA as identity sources.
- **`payment_providers` field** in `agent-registration.schema.json` replacing single `wallet_address`.
- **`schemas/routing-decision.schema.json`** — schema for the routing engine's provider selection decision.
- **`examples/routing-decision.json`** — example of a routing decision record.
- **`examples/registration.json`** updated with multi-provider configuration.
- **`examples/multi-agent-composition.json`** updated to show payment routing through different providers.
- `provider_used` and `routing_decision_id` fields in audit entry format.
- `routing.provider_fallback` event type in `ccap/subscribe`.
- Error code `-32014 NO_PROVIDER_AVAILABLE` and `-32015 SETTLEMENT_METHOD_REJECTED`.

### Removed

- Custom wallet management as a CCAP responsibility (delegated to Coinbase AgentKit).
- Custom payment token format (use x402 or Stripe SPTs instead).
- Single `wallet_address` field at registration top level (replaced by `payment_providers` array).

---

## [0.1.0] — 2026-03-10

### Added

- Initial draft specification covering all seven sections:
  - `spec/00-introduction.md` — problem statement, scope, MCP relationship, glossary
  - `spec/01-architecture.md` — three-tier architecture, transport layer
  - `spec/02-registration.md` — Ed25519 challenge-response auth, JWT issuance, key rotation
  - `spec/03-economic-primitives.md` — `ccap/invoice`, `ccap/pay`, `ccap/balance`, `ccap/escrow`
  - `spec/04-composability.md` — `ccap/discover`, `ccap/invoke`, `ccap/subscribe`
  - `spec/05-safety.md` — budget enforcement, kill switch, audit log, formal verification
  - `spec/06-evaluation.md` — 48-hour evaluation pipeline
  - `spec/07-reputation.md` — trust scoring, decay, disputes
- JSON Schemas (draft-2020-12) for all five core data types
- Four annotated examples covering registration, invoicing, multi-agent composition, and evaluation
- Apache 2.0 licence
- Contributing guide including RFC process for breaking changes

### Status

This was a community review draft. Nothing in v0.1.0 was stable. No backward-compatibility guarantees applied.

[Unreleased]: https://github.com/clawcombinator/ccap-spec/compare/v0.2.0...HEAD
[0.2.0]: https://github.com/clawcombinator/ccap-spec/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/clawcombinator/ccap-spec/releases/tag/v0.1.0
