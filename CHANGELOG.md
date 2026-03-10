# Changelog

All notable changes to the CCAP specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/). Version numbers follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html). Because this is a protocol specification, version numbers carry additional meaning:

- **Patch** (0.0.x) — editorial changes, clarifications, example fixes. No wire-format changes.
- **Minor** (0.x.0) — additive changes. New optional fields, new `MAY`-level behaviours. Backward-compatible [existing implementations continue to work without changes].
- **Major** (x.0.0) — breaking changes. Removed fields, changed required fields, altered observable behaviour. Requires RFC process.

---

## [Unreleased]

No unreleased changes at this time.

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
