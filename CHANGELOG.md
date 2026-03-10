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

This is a community review draft. Nothing in v0.1.0 is stable. All sections are open for feedback. No backward-compatibility guarantees apply until v1.0.0 is tagged.

[Unreleased]: https://github.com/clawcombinator/ccap-spec/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/clawcombinator/ccap-spec/releases/tag/v0.1.0
