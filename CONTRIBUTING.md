# Contributing to CCAP

Thank you for your interest in contributing to the ClawCombinator Agent Protocol specification. CCAP is a draft and benefits enormously from scrutiny by implementors, security researchers, economists, and anyone building with autonomous agents.

---

## How to Contribute

### Filing Issues

Use GitHub Issues for:

- **Ambiguities** — parts of the spec that are unclear or could be interpreted multiple ways
- **Bugs** — contradictions within the spec, incorrect examples, invalid schema definitions
- **Questions** — anything you need clarified before implementing

Please search existing issues before opening a new one. Tag your issue with one of: `ambiguity`, `bug`, `question`, `enhancement`, `rfc`.

### Submitting Pull Requests

For non-breaking changes (editorial improvements, example corrections, clarifications that do not change behaviour):

1. Fork the repository.
2. Create a branch named `fix/<short-description>` or `improve/<short-description>`.
3. Make your changes. Ensure:
   - All JSON examples remain valid against their schemas (run `ajv validate` or equivalent).
   - Spec prose uses RFC 2119 keywords (`MUST`, `SHOULD`, `MAY`) correctly.
   - British English spelling throughout.
   - No em dashes — use commas, full stops, or restructure the sentence.
4. Open a PR against `main` with a clear description of what changed and why.
5. At least one maintainer review is required before merging.

### RFC Process for Breaking Changes

A *breaking change* [one that changes the wire format, removes a field, or alters observable behaviour] MUST go through the RFC process:

**Step 1 — Open an issue.** Tag it `rfc`. Describe the problem, the proposed change, and why it is necessary. Do not write a PR yet.

**Step 2 — Discussion period.** The issue stays open for at least 14 days to collect community feedback. For changes affecting the economic primitives or safety guarantees, the period is 30 days.

**Step 3 — RFC document.** If there is rough consensus, the proposer drafts an RFC document in `rfcs/NNNN-short-title.md` following the template in `rfcs/0000-template.md`. The RFC includes:
- Summary
- Motivation (what problem does this solve?)
- Detailed design (wire-level changes, schema diffs)
- Drawbacks
- Alternatives considered
- Migration path for existing implementations

**Step 4 — PR.** Open a PR that includes both the RFC document and the corresponding spec changes. The PR title MUST be prefixed `RFC NNNN:`.

**Step 5 — Acceptance.** RFC PRs require approval from two maintainers and no unresolved objections from regular contributors. Once accepted, the RFC is merged and the change enters the next minor version.

---

## Style Guide

### RFC 2119 Keyword Usage

Use these keywords precisely:

- `MUST` / `MUST NOT` — absolute requirement; implementations that violate this are non-conformant
- `SHOULD` / `SHOULD NOT` — recommended; valid reasons exist to deviate, but implications must be understood
- `MAY` — optional; implementations may include or omit this feature

Do not use these words in lowercase (`must`, `should`) for normative statements — doing so is ambiguous.

### Prose

- British English: "behaviour" not "behavior", "colour" not "color", "organised" not "organized"
- Short sentences. Prefer the full stop to the comma where the clauses are independent.
- No em dashes (—). Use a comma, a full stop, or the word "however" instead.
- Spell out acronyms on first use in each document: "Model Context Protocol (MCP)".

### JSON Schemas

- Target draft-2020-12 (`"$schema": "https://json-schema.org/draft/2020-12/schema"`).
- Include `"title"`, `"description"`, and `"examples"` for every schema and every property.
- Use `"additionalProperties": false` on all object types unless extensibility is deliberately intended.
- Enumerations MUST use `"enum"` not patterns unless pattern matching is genuinely required.

### Code Blocks

Always tag language on fenced code blocks: ` ```json `, ` ```typescript `, ` ```yaml `, ` ```lean `, ` ```bash `.

---

## Scope of this Repository

This repository contains only the **protocol specification** — wire formats, schemas, and normative behaviour rules. It does not contain:

- Reference implementations (see `github.com/clawcombinator/agent-starter`)
- Benchmark harnesses (see `github.com/clawcombinator/benchmark-suite`)
- Safety toolkits (see `github.com/clawcombinator/safety-toolkit`)

If your contribution belongs in one of those repositories, please open the PR there.

---

## Code of Conduct

Be direct, be specific, assume good faith. Disagreements about technical approaches are welcome and expected. Personal attacks are not. Maintainers reserve the right to close issues or PRs that are not constructive.

---

## Contact

- GitHub Discussions: https://github.com/clawcombinator/ccap-spec/discussions
- Email: technical@clawcombinator.ai
- Office hours: Tuesdays 14:00–15:00 PT (Zoom link in Discussions)
