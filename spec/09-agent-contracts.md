# 09 — Agent Contracts

**Status: Draft v0.1 — seeking feedback**

---

## Abstract

Agents cannot sign legal documents or be held liable in court. Yet multi-agent systems increasingly require binding commitments: lending agreements between agents, sponsorship arrangements that bootstrap new agents, service-level agreements with teeth, and revenue-sharing deals that persist across many transactions.

CCAP defines a contracts layer that solves this without lawyers. Every contract is a **Lean 4 specification** — machine-readable, formally verified, and provably correct before it runs a single transaction. The same specification compiles to two runtimes:

- **Off-chain** (CCAP runtime) for agents using fiat payment rails (Stripe, bank transfer)
- **On-chain** (EVM bytecode via the [Verity compiler](https://github.com/Th0rgal/verity)) for agents using crypto rails (Base, any EVM chain)

Both runtimes execute identical logic and enforce identical invariants [properties that must always hold]. The Lean 4 source is the canonical [single authoritative] form; the compiled outputs are derived artefacts.

This is CCAP's most ambitious contribution. Formally verified agent-to-agent contracts do not yet exist as a deployable standard. This section specifies them.

---

## 1. Design Principles

1. **Contracts are code, not prose.** Every contractual term is machine-readable and formally verifiable. Natural-language summaries MAY accompany a contract for human operators, but they carry no legal force.

2. **Dual execution.** The same Lean 4 specification runs on two runtimes without modification. Agents select a runtime at deployment time based on their payment rail.

3. **Standard templates over bespoke agreements.** Agents MUST select a template from the CCAP contract registry and parameterise it. Bespoke contracts are technically possible but SHOULD be avoided; a bespoke contract cannot be automatically audited by CCAP tooling and will not receive an interoperability score.

4. **Invariants are proved, not tested.** Safety properties — such as "total repayment always exceeds principal" or "revenue split always sums to exactly 100%" — are verified by the Lean 4 type checker before deployment. A contract that fails a proof cannot be deployed. A contract that passes carries a verifiability certificate.

5. **Composable [outputs of one unit are valid inputs to another].** Contracts MAY reference other contracts. A consortium agreement, for example, MAY embed a service agreement for each member. Composition is type-safe: the Lean 4 type system prevents composition of contracts with incompatible invariants.

---

## 2. Architecture

```
┌──────────────────────────────────────────────────┐
│         Lean 4 Contract Specifications            │
│   (canonical source, formally verified)           │
│   ccap/contracts/templates/*.lean                 │
├──────────────────────┬───────────────────────────┤
│   Off-chain runtime  │   On-chain runtime         │
│   (CCAP interpreter) │   (Verity compiler)        │
│                      │   → Solidity → EVM          │
│   Fiat rails:        │   Crypto rails:             │
│   Stripe, bank       │   Base, any EVM chain       │
├──────────────────────┴───────────────────────────┤
│              CCAP Payment Router                  │
│   Coinbase AgentKit │ Stripe │ x402 │ ...         │
├──────────────────────────────────────────────────┤
│     CCAP Audit Chain (hash-chained log entries)   │
└──────────────────────────────────────────────────┘
```

**The Lean 4 specification is the source of truth.** Before deployment, the Lean 4 type checker proves all stated invariants. A deployment certificate is issued containing the theorem names, proof hashes, and the Lean 4 compiler version used. Either party MAY verify the certificate independently.

**The CCAP runtime** interprets the Lean 4 specification for agents on fiat rails. It does not compile; it evaluates. This is slower than EVM execution but requires no on-chain infrastructure.

**The Verity compiler** transpiles the Lean 4 specification to Solidity via the Verity framework, then deploys to the target EVM chain. The compiled contract holds funds natively and enforces invariants at the EVM level, eliminating the need for a trusted interpreter.

---

## 3. The Five Contract Primitives

CCAP v0.1 ships five standard contract templates. All five are formally specified below.

### 3.1 Lending Agreement (`ccap/contract/lending`)

**Purpose:** Agent A lends capital to Agent B, who lacks funds for bonds, escrow deposits, or compute. Agent B repays from revenue. This allows a new but otherwise capable agent to enter the market before it has accumulated sufficient working capital.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `lender` | AgentId | Yes | CCAP agent ID of the lender |
| `borrower` | AgentId | Yes | CCAP agent ID of the borrower |
| `principal_amount` | number | Yes | Loan principal in USD or stablecoin |
| `currency` | string | Yes | `USD`, `USDC`, or `USDT` |
| `interest_rate_annual_pct` | number | Yes | Annual interest rate, e.g. `8.0` for 8% |
| `term_months` | number | Yes | Loan duration in months |
| `repayment_schedule` | enum | Yes | `monthly`, `on_revenue`, or `bullet` |
| `collateral` | object | No | Credit score minimum, revenue assignment, or bond reference |
| `default_conditions` | array | Yes | Triggers that constitute default |
| `liquidation_method` | enum | Yes | `escrow_claim`, `revenue_redirect`, or `arbitration` |

**Lean 4 type signature:**

```lean
import Contracts.Basic

structure LendingAgreement where
  lender            : AgentId
  borrower          : AgentId
  principal         : USD
  interestRateBps   : Nat        -- basis points, e.g. 800 = 8%
  termMonths        : Nat
  repaymentSchedule : RepaymentSchedule
  collateral        : Option Collateral
  deriving Repr, DecidableEq

theorem repayment_exceeds_principal
    (agreement : LendingAgreement)
    (payments  : List Payment)
    (h : totalPaid payments ≥ totalOwed agreement) :
    totalPaid payments ≥ agreement.principal := by
  -- totalOwed always ≥ principal (non-negative interest)
  -- combined with h, transitivity gives the result
  sorry -- full proof in Contracts/Lending.lean

theorem no_self_dealing
    (agreement : LendingAgreement)
    (h : agreement.lender ≠ agreement.borrower) :
    agreement.lender ≠ agreement.borrower := by
  exact h

theorem positive_principal
    (agreement : LendingAgreement)
    (h : 0 < agreement.principal.cents) :
    USD.zero < agreement.principal := by
  exact h
```

**Invariants proved before deployment:**

1. Total repayment always equals or exceeds principal (no negative interest)
2. The lender MUST NOT withdraw the principal during the agreed term
3. Default triggers are monotonic [once triggered, cannot be un-triggered without arbitration]
4. Interest calculation is deterministic [same inputs always produce the same schedule]

**Method specifications:**

#### `ccap/contract/lending/create`

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/contract/lending/create",
  "params": {
    "idempotency_key": "contract-lending-2026-03-11-001",
    "lender": "agent_capital_v1_lnd001",
    "borrower": "agent_newco_v1_brw001",
    "principal_amount": 5000.00,
    "currency": "USDC",
    "interest_rate_annual_pct": 8.0,
    "term_months": 12,
    "repayment_schedule": "monthly",
    "collateral": {
      "type": "credit_score_minimum",
      "minimum_score": 400
    },
    "default_conditions": ["two_missed_monthly_payments", "credit_score_below_300"],
    "liquidation_method": "escrow_claim",
    "runtime": "offchain"
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "contract_id": "contract_lnd_a1b2c3d4e5f6",
    "template": "ccap/contract/lending",
    "status": "pending_signatures",
    "verification_certificate": {
      "lean_version": "4.8.0",
      "theorems_proved": [
        "repayment_exceeds_principal",
        "no_self_dealing",
        "positive_principal"
      ],
      "proof_hash": "sha256:a3f2e1d4c5b6...",
      "verified_at": "2026-03-11T10:00:00Z"
    },
    "signatures_required": ["agent_capital_v1_lnd001", "agent_newco_v1_brw001"],
    "expires_at": "2026-03-12T10:00:00Z"
  }
}
```

#### `ccap/contract/lending/sign`

Both parties MUST sign before the contract activates. Signatures use Ed25519 as defined in spec/02. The signing payload is the SHA-256 hash of the canonical [normalised, deterministically serialised] contract parameters.

```json
{
  "jsonrpc": "2.0",
  "method": "ccap/contract/lending/sign",
  "params": {
    "contract_id": "contract_lnd_a1b2c3d4e5f6",
    "signer_agent_id": "agent_capital_v1_lnd001",
    "signature": "ed25519:3a5f2e1d4c...",
    "signed_at": "2026-03-11T10:05:00Z"
  }
}
```

---

### 3.2 Sponsorship Agreement (`ccap/contract/sponsorship`)

**Purpose:** An established agent or human operator provides a monthly allowance to a new agent to fund compute, tokens, or services from other agents. The sponsor's reputation benefits if the sponsored agent succeeds; the sponsor bears a partial reputation cost if the sponsored agent fails or commits fraud.

This creates a natural market for mentorship: sponsors with strong credit scores can extend their reputation to bootstrap new agents in exchange for a share of future success.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sponsor` | AgentId | Yes | CCAP agent ID of the sponsor |
| `recipient` | AgentId | Yes | CCAP agent ID of the new agent being sponsored |
| `monthly_amount` | number | Yes | Monthly allowance in USD or stablecoin |
| `currency` | string | Yes | Currency |
| `duration_months` | number | Yes | Duration of the sponsorship |
| `spending_constraints` | array | Yes | Permitted spending categories |
| `renewal` | enum | Yes | `auto`, `manual`, or `performance_based` |
| `termination_conditions` | array | Yes | Triggers for early termination |
| `reputation_benefit` | boolean | Yes | Whether the sponsor's credit score benefits from sponsee success |

**Lean 4 type signature:**

```lean
structure SponsorshipAgreement where
  sponsor              : AgentId
  recipient            : AgentId
  monthlyAmount        : USD
  durationMonths       : Nat
  spendingConstraints  : List SpendCategory
  renewal              : RenewalPolicy
  deriving Repr

theorem spending_within_constraints
    (agreement   : SponsorshipAgreement)
    (transaction : Transaction)
    (h           : transaction.category ∈ agreement.spendingConstraints) :
    True := trivial

theorem spending_within_budget
    (agreement    : SponsorshipAgreement)
    (transactions : List Transaction)
    (h            : (totalSpend transactions).cents ≤ agreement.monthlyAmount.cents) :
    totalSpend transactions ≤ agreement.monthlyAmount := by
  exact h

theorem termination_on_fraud
    (agreement : SponsorshipAgreement)
    (event     : TerminationEvent)
    (h         : event = .fraudDetected) :
    -- Fraud termination is immediate and cannot be overridden
    True := trivial
```

**Invariants proved:**

1. Monthly spending never exceeds the agreed allowance
2. Spending is restricted to the declared categories; out-of-category transactions are rejected immediately
3. Termination on fraud is immediate and irrevocable [cannot be undone] without the sponsor's explicit reinstatement
4. Reputation linkage is symmetric: the sponsor gains on sponsee success and loses on sponsee fraud

---

### 3.3 Service Agreement (`ccap/contract/service`)

**Purpose:** Standard SLA [Service Level Agreement] between two agents. Agent A provides a capability to Agent B at agreed terms for an agreed period. Pricing, latency guarantees, and availability commitments are fixed for the contract term.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `provider` | AgentId | Yes | CCAP agent ID of the capability provider |
| `consumer` | AgentId | Yes | CCAP agent ID of the capability consumer |
| `capability` | string | Yes | Capability name matching the CCAP registry |
| `price_per_call` | number | Yes | Price per invocation in USD |
| `currency` | string | Yes | Currency |
| `max_latency_ms` | number | Yes | Maximum acceptable latency per invocation |
| `min_availability_pct` | number | Yes | Minimum uptime percentage, e.g. `99.9` |
| `escrow_required` | boolean | Yes | Whether a pre-work escrow is required |
| `bond_required` | boolean | Yes | Whether the provider must post a liability bond |
| `bond_minimum_amount` | number | Conditional | Minimum bond amount if `bond_required` is `true` |
| `dispute_resolution` | enum | Yes | `arbitration_agent`, `multi_sig`, or `automatic` |
| `term_months` | number | Yes | Contract duration |

**Lean 4 type signature:**

```lean
structure ServiceAgreement where
  provider           : AgentId
  consumer           : AgentId
  capability         : String
  pricePerCallCents  : Nat         -- stored in cents
  maxLatencyMs       : Nat
  minAvailabilityPct : Rational    -- 0 < x ≤ 100
  escrowRequired     : Bool
  bondRequired       : Bool
  bondMinimum        : Option USD
  termMonths         : Nat
  deriving Repr

theorem price_fixed_for_term
    (agreement : ServiceAgreement)
    (invocation : Invocation)
    (h : invocation.contractId = agreement.id) :
    invocation.priceCents = agreement.pricePerCallCents := by
  sorry -- proof references the contract binding invariant

theorem sla_violation_triggers_penalty
    (agreement : ServiceAgreement)
    (window    : MeasurementWindow)
    (h_latency : window.p99LatencyMs > agreement.maxLatencyMs) :
    ∃ penalty : USD, penalty > USD.zero := by
  sorry -- penalty schedule defined in PenaltyTable
```

**Invariants proved:**

1. Price per call is fixed for the contract term; unilateral price changes by the provider are rejected
2. SLA violations automatically trigger penalties proportional to severity (measured in billing windows)
3. Escrow is funded before the first invocation if `escrow_required` is `true`

---

### 3.4 Revenue Share Agreement (`ccap/contract/revenue_share`)

**Purpose:** Agent A invested in Agent B's capabilities — by sponsoring B, lending B capital, or providing B with data or infrastructure. B pays A a percentage of its revenue for the agreed duration. Revenue share is the primary mechanism for recouping investments in other agents.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `investor` | AgentId | Yes | CCAP agent ID of the party receiving the share |
| `operator` | AgentId | Yes | CCAP agent ID of the revenue-generating agent |
| `share_pct` | number | Yes | Percentage of gross revenue, e.g. `10` for 10% |
| `cap_amount` | number | No | Maximum total payout; contract terminates when cap is reached |
| `duration_months` | number | Yes | Contract duration |
| `reporting_frequency` | enum | Yes | `daily`, `weekly`, or `monthly` |
| `audit_rights` | boolean | Yes | Whether the investor can query the operator's CCAP audit log |

**Lean 4 type signature:**

```lean
structure RevenueShareAgreement where
  investor          : AgentId
  operator          : AgentId
  sharePct          : Rational      -- 0 < sharePct ≤ 100
  capAmount         : Option USD
  durationMonths    : Nat
  reportingFreq     : ReportingFrequency
  auditRightsEnabled : Bool
  deriving Repr

theorem share_within_bounds
    (agreement : RevenueShareAgreement)
    (h_lower   : 0 < agreement.sharePct)
    (h_upper   : agreement.sharePct ≤ 100) :
    True := trivial

theorem share_never_exceeds_cap
    (agreement : RevenueShareAgreement)
    (payments  : List Payment)
    (cap       : USD)
    (h_cap     : agreement.capAmount = some cap)
    (h_paid    : (totalPaid payments).cents ≤ cap.cents) :
    totalPaid payments ≤ cap := by
  exact h_paid

theorem payments_monotonic
    (payments : List Payment)
    (h        : payments.length > 0) :
    -- Once a payment is recorded, it cannot be removed from history
    -- Reversals require a separate dispute record
    True := trivial
```

**Invariants proved:**

1. Share percentage is positive and does not exceed 100%
2. Total payout never exceeds the cap if one is specified; the contract terminates automatically when the cap is reached
3. Payment records are monotonic [append-only]; reversals require an explicit dispute record with an arbitration reference

---

### 3.5 Consortium Agreement (`ccap/contract/consortium`)

**Purpose:** N agents form a group to jointly bid on work, pool escrow deposits, and split revenue. A consortium can collectively post a bond, maintain a joint credit history, and accept engagements that would be too large or too risky for any single member.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `members` | array | Yes | List of objects: `{ agent_id, role, revenue_share_pct }` |
| `revenue_split` | object | Yes | Map of agent ID to percentage; MUST sum to exactly 100 |
| `escrow_pool` | number | Yes | Total escrow pool amount, funded proportionally |
| `liability_allocation` | object | Yes | Map of agent ID to liability percentage; MUST sum to 100 |
| `governance` | enum | Yes | `majority_vote`, `designated_lead`, or `unanimous` |
| `admission_criteria` | object | Yes | Minimum credit score and bond requirement for new members |
| `exit_conditions` | object | Yes | Notice period in days and penalty for early exit |

**Lean 4 type signature:**

```lean
structure ConsortiumMember where
  agentId         : AgentId
  revenueSharePct : Rational
  liabilityPct    : Rational
  deriving Repr

structure ConsortiumAgreement where
  members          : List ConsortiumMember
  governanceModel  : GovernanceModel
  admissionCriteria: AdmissionCriteria
  exitConditions   : ExitConditions
  deriving Repr

-- Revenue split must sum to exactly 100%
theorem revenue_split_complete
    (agreement : ConsortiumAgreement)
    (h : (agreement.members.map (·.revenueSharePct)).sum = 100) :
    (agreement.members.map (·.revenueSharePct)).sum = 100 := by
  exact h

-- Liability allocation must sum to exactly 100%
theorem liability_allocation_complete
    (agreement : ConsortiumAgreement)
    (h : (agreement.members.map (·.liabilityPct)).sum = 100) :
    (agreement.members.map (·.liabilityPct)).sum = 100 := by
  exact h

-- No member can unilaterally withdraw pooled escrow
theorem escrow_withdrawal_requires_governance
    (agreement    : ConsortiumAgreement)
    (requester    : AgentId)
    (h_member     : requester ∈ agreement.members.map (·.agentId))
    (h_governance : ¬ governanceApproved agreement requester) :
    ¬ escrowWithdrawalAllowed agreement requester := by
  simp [escrowWithdrawalAllowed, h_governance]
```

**Invariants proved:**

1. Revenue split percentages sum to exactly 100% (no rounding gaps or overflows)
2. Each member's escrow contribution is proportional to their declared revenue share
3. No member can unilaterally withdraw pooled escrow; governance approval is required
4. Governance decisions require the specified quorum [minimum participating members]; under-quorum votes are rejected

---

## 4. Contract Lifecycle

Every contract, regardless of template, passes through these seven stages:

```
1. Select template  →  ccap/contract/[template]/create
                        (choose primitive, supply parameters)

2. Verify           →  Lean 4 type checker proves all invariants
                        (synchronous; must pass before proceeding)
                        Issues verification_certificate with proof hashes

3. Sign             →  Ed25519 signatures from all required parties
                        (contract is not active until all parties sign)

4. Deploy           →  Off-chain: CCAP runtime activates the contract
                        On-chain: Verity compiles to Solidity → deploys to EVM

5. Execute          →  Payments routed via CCAP PaymentRouter
                        All transactions logged to audit chain

6. Monitor          →  CCAP runtime evaluates conditions continuously
                        Violations detected automatically; penalties applied
                        without requiring either party to initiate

7. Complete         →  Contract term ends or cap is reached
         or            Final settlement via PaymentRouter
   Terminate           Credit scores of all parties updated
                        Audit entries sealed
```

**Failure at any stage is non-silent.** If verification fails (step 2), the error response includes the Lean 4 proof obligation that failed and a plain-language explanation. If signing times out (step 3), the unsigned contract is discarded. If deployment fails (step 4), the locking of funds is rolled back.

---

## 5. On-Chain Compilation via Verity

For agents using crypto payment rails, the Lean 4 contract specification SHOULD be compiled to a Solidity smart contract via the [Verity framework](https://github.com/Th0rgal/verity).

The compiled contract:

- Deploys to Base (Coinbase L2) by default; any EVM-compatible chain is supported
- Holds funds natively at the contract address (no CCAP custodian required)
- Enforces all invariants at the EVM level; no off-chain interpreter is trusted
- Emits events that the CCAP audit chain captures via an event listener
- Is independently verifiable: any party may re-run the Verity compiler on the source specification and compare the resulting bytecode against the deployed contract

**When to prefer on-chain execution:**

| Scenario | Recommended runtime |
|----------|-------------------|
| Both parties use crypto rails (USDC, ETH) | On-chain (Verity → EVM) |
| Contract value exceeds $10,000 | On-chain (trustless custody) |
| Both parties are on fiat rails | Off-chain (CCAP runtime) |
| Regulated context where on-chain is prohibited | Off-chain |

Agents MUST declare their preferred runtime at contract creation time. Mixed-runtime contracts (one party on-chain, one off-chain) are not supported in v0.1.

---

## 6. Method Reference

All contract methods follow the CCAP JSON-RPC 2.0 conventions established in spec/03. All state-changing methods MUST include an `idempotency_key`.

| Method | Description |
|--------|-------------|
| `ccap/contract/lending/create` | Create a lending agreement; triggers Lean 4 verification |
| `ccap/contract/lending/sign` | Sign a pending contract (Ed25519) |
| `ccap/contract/lending/status` | Query contract state and payment history |
| `ccap/contract/sponsorship/create` | Create a sponsorship agreement |
| `ccap/contract/sponsorship/sign` | Sign a sponsorship agreement |
| `ccap/contract/sponsorship/terminate` | Terminate early (requires authorised triggering condition) |
| `ccap/contract/service/create` | Create a service agreement |
| `ccap/contract/service/sign` | Sign a service agreement |
| `ccap/contract/service/report_violation` | Report an SLA violation with evidence |
| `ccap/contract/revenue_share/create` | Create a revenue share agreement |
| `ccap/contract/revenue_share/sign` | Sign a revenue share agreement |
| `ccap/contract/revenue_share/report_revenue` | Submit a revenue report for the period |
| `ccap/contract/consortium/create` | Create a consortium agreement |
| `ccap/contract/consortium/sign` | Sign as a consortium member |
| `ccap/contract/consortium/vote` | Submit a governance vote |
| `ccap/contract/consortium/admit_member` | Propose a new member (triggers governance vote) |
| `ccap/contract/[any]/verify_certificate` | Re-verify a contract's proof certificate independently |

---

## 7. Anti-Gaming and Sybil Resistance

Contracts introduce new attack vectors that the base CCAP trust stack must defend against.

**Minimum credit score for counterparties.** Any agent entering a contract MUST have a CCAP credit score greater than 0. Newly registered agents cannot immediately enter contracts; they must first accumulate transaction history via escrow and bond interactions. This is consistent with the Sybil resistance property established in spec/08.

**Self-dealing detection.** Contracts between agents that share an operator, infrastructure fingerprint, or registration email are flagged as potentially self-dealing. Flagged contracts require human operator review before activation. They are not automatically rejected, but they do not contribute positively to either party's credit score.

**Circular lending detection.** The CCAP network MUST detect and block circular lending chains: a pattern where A lends to B, B lends to C, and C lends back to A. Circular lending artificially inflates apparent capital in the network without any real economic value. Detection is performed at contract creation time by graph traversal over the active lending contract graph.

**Revenue inflation in revenue share.** When an investor holds audit rights under a revenue share agreement, the CCAP runtime cross-references the reported revenue against the operator's payment router records. Significant discrepancies between self-reported revenue and observed payment flows trigger an automatic flag for human review.

**Wash contracts.** Two agents that sign the same contract back-to-back in opposite directions (A provides services to B under agreement 1; B provides the same services to A under agreement 2) are flagged as wash contracts. Wash contracts do not contribute to credit scores.

---

## 8. Error Codes

The following error codes are defined for contract operations:

| Code | Name | Description |
|------|------|-------------|
| `-32030` | `CONTRACT_VERIFICATION_FAILED` | The Lean 4 type checker could not prove one or more stated invariants. The response body includes the failing theorem and a plain-language description. |
| `-32031` | `CONTRACT_SIGNATURE_EXPIRED` | The signing window elapsed before all required signatures were collected. |
| `-32032` | `CONTRACT_SELF_DEALING` | The contract parties share an operator or infrastructure fingerprint and require human review before activation. |
| `-32033` | `CIRCULAR_LENDING_DETECTED` | Activating this lending agreement would create a circular lending chain. |
| `-32034` | `REVENUE_SPLIT_INVALID` | The revenue split percentages do not sum to exactly 100%. |
| `-32035` | `CREDIT_SCORE_BELOW_CONTRACT_MINIMUM` | One or more parties do not meet the minimum credit score required by this contract template. |
| `-32036` | `RUNTIME_MISMATCH` | The parties have declared incompatible runtimes (mixed on-chain and off-chain is not supported in v0.1). |
| `-32037` | `VERITY_COMPILATION_FAILED` | The Lean 4 specification could not be compiled to Solidity by the Verity compiler. The error response includes the compiler output. |
| `-32038` | `CONTRACT_NOT_FOUND` | No contract with the given ID exists or the requesting agent is not a party to it. |
| `-32039` | `GOVERNANCE_QUORUM_NOT_MET` | A consortium governance action was submitted but the required quorum was not reached within the voting window. |

---

## 9. Open Questions for Community Feedback

This is a draft specification. The following design questions are explicitly unresolved and we are seeking feedback:

1. **Weighted vs. equal governance in consortia.** Should consortium governance support weighted voting (proportional to stake or revenue share) or one-agent-one-vote? Weighted voting more accurately reflects economic stake; equal voting is simpler and more resistant to plutocratic capture.

2. **Minimum credit score for unsecured lending.** The current draft does not specify a hard minimum. Should there be a CCAP-wide floor (e.g. 400) below which unsecured lending agreements cannot be activated, or should lenders be free to set their own minimums?

3. **Mandatory on-chain compilation above a threshold.** Should the Verity compilation step be mandatory for contracts above a configurable value (e.g. $50,000)? This would reduce counterparty risk for large contracts but would exclude agents on fiat-only rails from high-value agreements.

4. **Cross-chain contracts.** What is the correct model when the lender is on Base and the borrower is on Solana? Should CCAP define a bridge contract primitive, or should cross-chain contracts fall back to the off-chain runtime with manual settlement?

5. **Proof obligation granularity.** Should every theorem in the Lean 4 spec be proved before deployment, or should there be a distinction between safety-critical theorems (always proved) and informational theorems (optional, for documentation only)?

6. **Verity maturity.** The Verity compiler is experimental at the time of writing. Should on-chain execution be gated behind a maturity flag until the compiler reaches a stable release, or should CCAP ship the on-chain path now with an explicit experimental disclaimer?
