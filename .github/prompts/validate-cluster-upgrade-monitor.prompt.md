---
description: "Verify proposed cluster-upgrade recording and alert rules with actual-expression missing-data tests and seven-day historical characterization using alert-tester."
agent: "agent"
---

Validate the candidate cluster-upgrade monitoring policy described in
[the Phase 2 contract](../../docs/alerts/cluster-upgrade.md). This is
read-only validation, not deployment or approval to alter incidents.

## Establish inputs

- Identify the candidate commit and exact recording/alert files.
- Obtain approved Grafana environment URL, region/datasource scope and UTC
  analysis dates from the user. Do not silently use the previous calendar week.
- Confirm Phase 1 deployment/coverage start. Require seven usable observation
  days after deployment plus appropriate warm-up/pre-roll; if unavailable,
  report the historical track as blocked.
- Use the approved identity and network path. Never expose tokens or move raw
  telemetry/caches outside their authorized processing boundary.
- Save ad-hoc artifacts only in the workspace's designated gitignored scratch
  location. Do not commit raw observations, caches or incident/customer details.

## Track A: deterministic rule verification

Run the repository's existing generator/promtool test entry points and the
actual-expression regressions in
[upgrade_monitor_regression_test.go](../../tooling/prometheus-rules/internal/upgrade_monitor_regression_test.go).
Use the real candidate expressions, not an AI approximation.

Verify at least:

1. Clear and firing values survive missing observations without a state change.
2. Missing/stale samples pause completion/lifecycle confirmation; contrary
   valid evidence resets only that reason's count.
3. Third valid convergence/lifecycle observation clears, with current evidence.
4. Partial target rows, missing counts and all target/count/convergence loss do
   not hide unfinished version telemetry from another source.
5. Partial/malformed inventory does not imply removal; a confirmed zero is
   published before series retirement.
6. Evaluator gaps beyond five minutes recover the latest 0/1/reset within the
   supported bound without resurrecting older failures.
7. Rollback, target/phase/UUID changes, initial-install exclusion and detector
   Pending/identity behavior match the contract.

Keep input gaps in the fixtures. Do not replace missing input with forward-
filled/default healthy values. If extracting historical data for a fixture,
label injected changes as synthetic, preserve sample timestamps/staleness and
verify fixture conversion semantics before claiming exact replay.

## Track B: seven-day historical characterization

Inspect the current
[alert-tester documentation](https://github.com/mmazur/alert-tester/blob/main/README.md)
before running it. Verify available CLI options with the installed binary.
The ARO skill's default recursive recording-rule inlining must NOT be applied
to this self-referencing state machine.

- If actual candidate recordings have been deployed in an approved non-paging
  scope, query those outputs directly and evaluate the detector's filtered
  `== 1` condition, planned one-minute cadence and five-minute Pending.
- Otherwise characterize raw observations/simple predicates only. Exact
  stateful backtesting is blocked until a verified chronological evaluator
  executes the recording group with proper initialization and missing-data
  semantics. Do not pretend `atest` implements that evaluator.
- Use the services datasources, expected resource scope and correlation labels
  from the contract. Do not group by version/phase or blindly default to a
  service-cluster-only incident identity.
- Assess presence, freshness, target/inventory consistency, exporter epoch
  churn and coverage before interpreting quiet results.
- Report firing episodes and precise start/end boundaries; correlate observed
  completion/teardown events with independent source evidence.
- Explicitly separate true gaps, filter-false absence and query/fetch failures.
  `atest` absence and resolution-delay merging do not prove unknown-state
  preservation or the real Azure/IcM mitigation chain.
- Treat incident counts as correlation-key estimates, not actual ticket counts.
  Inspect details: aggregate replay equality does not prove equal timings.
- Seven days without an edge case makes that scenario unobserved, not passed.
  Use Track A rather than production mutations to cover those edges.

## Result

Report the exact candidate/tools/window/scope and, per scenario:

| Scenario | Evidence type | Expected | Observed | Result |
| --- | --- | --- | --- | --- |
| Example | Synthetic or production | Contract | Measured output | Passed, failed, unobserved or blocked |

Keep deterministic and production conclusions separate. Name missing raw
metrics, recording deployment, history initialization or replay tooling as
blockers. Link code/test evidence and authorized query links, not credentials.
Do not claim SDP deployment, Action Group delivery or IcM auto-mitigation from
local or historical evaluator results.
