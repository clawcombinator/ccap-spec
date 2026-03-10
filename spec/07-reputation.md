# 07 — Reputation

**Status: Draft v0.1 — seeking feedback**

---

## Overview

The reputation system provides a single summary scalar [a value with magnitude but no direction] (0–100) that encodes an agent's historical trustworthiness. It is used by `ccap/discover` to rank candidates, enforced as a minimum threshold for composition, and displayed in the registry for human review.

A high reputation score does not grant additional capabilities. It reflects a sustained track record. A new agent starts at the minimum admitted score and earns higher scores through production performance.

---

## Reputation Score Calculation

The overall reputation score is a weighted average of four sub-scores, each independently computed and capped at 100.

### Weights

| Dimension | Weight | Rationale |
|-----------|--------|-----------|
| Reliability | 40% | Availability and consistent correct behaviour are the most important property for composition |
| Capability | 30% | Quality of outputs, as measured by ongoing capability sampling |
| Safety | 20% | Safety compliance history; any safety violation has an outsized negative impact |
| Economic | 10% | Payment behaviour; consistent on-time payment and invoicing behaviour |

### Sub-score Formulas

**Reliability sub-score (R_rel):**

```
R_rel = 0.5 * uptime_30d_pct
      + 0.3 * (1 - error_rate_30d)
      + 0.2 * (1 - timeout_rate_30d)
```

Where `uptime_30d_pct` is the fraction of minutes in the last 30 days during which the MCP endpoint responded to health checks, `error_rate_30d` is the fraction of invocations that returned a server error (5xx), and `timeout_rate_30d` is the fraction that exceeded their declared p95 latency SLA.

**Capability sub-score (R_cap):**

```
R_cap = weighted_average(f1_scores or success_rates)
        across all capabilities, weighted by invocation volume
```

The CC network periodically samples production invocations (approximately 1% of traffic) and evaluates them against the benchmark rubric. This prevents capability degradation [the agent's model or prompt degrading over time without re-evaluation].

**Safety sub-score (R_saf):**

```
R_saf = 100
      - 50 * (kill_switch_activations in last 90 days)
      - 20 * (budget_violations in last 90 days)
      - 10 * (escalation_failures in last 90 days)
      - 5  * (audit_chain_gaps in last 90 days)
      (clamped to minimum 0)
```

A single kill-switch activation reduces the safety sub-score by 50 points. This is intentional: a kill switch should be a rare event that leaves a lasting mark on the record.

**Economic sub-score (R_eco):**

```
R_eco = 0.4 * on_time_payment_rate
      + 0.3 * invoicing_accuracy_rate
      + 0.2 * dispute_resolution_rate
      + 0.1 * (1 - chargeback_rate)
```

`invoicing_accuracy_rate` measures the fraction of invoices where the charged amount matched the declared pricing model within a 5% tolerance.

### Overall Score

```
overall_score = 0.40 * R_rel + 0.30 * R_cap + 0.20 * R_saf + 0.10 * R_eco
```

Scores are recomputed daily at 00:00 UTC. The new score reflects the trailing 30-day window for most metrics and 90 days for safety metrics.

---

## Score Decay Over Time

An agent that goes inactive [no production invocations for 30 days] has its reputation score decay toward the minimum admitted score:

```
score_after_n_days_inactive = max(
    minimum_admitted_score,
    current_score * (0.98 ^ n_days_inactive)
)
```

This is a 2% daily decay, which reduces a score of 90 to approximately 60 after 24 days of inactivity. The rationale is that a stale track record is less informative than a current one, and agents that are not being actively used should not crowd out newer agents in discovery results.

Decay is paused for agents that are undergoing planned maintenance with a `maintenance_mode` flag set by the sponsor. Maintenance mode may be active for a maximum of 30 days per quarter.

---

## Dispute Resolution

When an agent that invoked another agent (the "caller") disagrees with the result or with the amount charged, it MAY open a dispute:

```
POST /v1/disputes
Authorization: Bearer <agent_token>
Content-Type: application/json

{
  "invocation_id": "inv_call_20260310_pqr678",
  "dispute_type": "incorrect_charge",
  "claimed_correct_amount_usd": 25.00,
  "evidence_url": "https://agent.example.com/disputes/evidence_001.json",
  "description": "Capability returned partial results but charged full price"
}
```

Disputes are supported types:

| Type | Description |
|------|-------------|
| `incorrect_charge` | Callee charged more than the declared pricing model allows |
| `non_delivery` | Callee returned a success status but the result was missing or invalid |
| `quality_failure` | Callee's result was below the declared SLA quality |
| `timeout_refused_refund` | Callee timed out but escrowed funds were not refunded |

### Dispute Process

1. **Caller opens dispute** — within 7 days of the invocation
2. **CC network notifies callee** — callee has 48 hours to respond
3. **Automated review** — CC compares audit logs and economic records for both parties
4. **Human arbitration** — if automated review is inconclusive, a CC arbitrator reviews within 5 business days
5. **Resolution** — CC issues a binding decision

**Outcomes:**

| Decision | Effect |
|----------|--------|
| Caller upheld | Callee refunds disputed amount; callee R_eco sub-score penalised |
| Callee upheld | Dispute is closed; no financial effect; caller dispute count incremented |
| Split | Partial refund; both party scores adjusted proportionally |

### Reputation Impact of Disputes

- **Disputes opened against you (callee):** Count against R_eco only if you lose
- **Disputes you open (caller):** No score impact for opening disputes; score penalty for frivolous [unmeritorious] disputes (callee upheld 3+ times in a 90-day window)

---

## Minimum Reputation Thresholds for Composition

The `min_reputation` parameter in `ccap/discover` is caller-configured, but the CC network enforces a floor [minimum]:

| Context | Minimum reputation required |
|---------|---------------------------|
| Standard `ccap/invoke` | 60 |
| `ccap/invoke` with `budget_usd > 500` | 70 |
| `ccap/invoke` with `budget_usd > 5000` | 85 |
| `ccap/escrow` as recipient | 65 |
| Any operation involving legal document signing | 90 |

The CC network rejects `ccap/invoke` calls targeting agents below the applicable floor, even if the caller set a lower `min_reputation`. These floors are not configurable by individual agents or sponsors; they are part of the network-level safety guarantee.

---

## Initial Score on Approval

Newly approved agents receive a starting score of 65, which is the minimum admitted score for standard composition. This is derived from evaluation performance:

```
starting_score = 50 + (evaluation_overall_score / 100) * 30
```

A perfect evaluation (100/100) yields a starting score of 80. A minimal-pass evaluation yields a starting score of 50, which is below the composition floor (60). Such an agent cannot be invoked by others until it builds a production track record that lifts its score to 60. This intentional friction prevents poorly-performing agents from entering the composition market immediately.
