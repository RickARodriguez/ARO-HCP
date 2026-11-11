# Proposed cluster-level upgrade monitor (Phase 2)

Tracks [ARO-30519](https://issues.redhat.com/browse/ARO-30519) and the approved
cutover follow-up [ARO-30662](https://issues.redhat.com/browse/ARO-30662).
Requires the backend observations and dispatch safeguards in
[Phase 1](../cluster-upgrade-observations.md), published separately as
[Azure/ARO-HCP#8057](https://github.com/Azure/ARO-HCP/pull/8057).

This branch preserves a monitoring proposal for later review. It adds
recordings and replaces the two version-specific journey alerts with one
cluster-level detector. It must not be deployed as an observation-only change:
deploying its alert artifact changes the production consumer.

## Separation from Phase 1

| Phase 1 | Phase 2 |
| --- | --- |
| Teardown-aware RP dispatch guard | Duration/eligibility and monitor decisions |
| Coherent raw version/target/lifecycle observations | Completeness checks, retained state and clearing policy |
| Independent primary inventory | Additive recording outputs and cluster-level detector |
| Backend/collector tests | Recording-state, alert and historical verification |
| Existing recordings/alerts/routing unchanged | Reviewed replacement of the legacy alert consumer |

There is no legacy/shadow/active configuration, extra alert module, or generator
enable parameter in this proposal. The existing RP rule group is reused;
other RP alert groups and the shared Action Group are unchanged.

## Monitor contract

`hosted_control_plane_upgrade:stuck:bool` has a stable key of service cluster,
ARM resource ID and subscription. Version, phase and changing UUID observations
do not create new alert identities. Per-version diagnostics remain available.

| Term | Definition |
| --- | --- |
| Current targets | Newest observed active version plus resolved Desired if different |
| Historical target | Older retained history; not independently actionable as a stuck upgrade |
| Converged | Resolved Desired equals newest current and that entry is Completed |
| Unknown | Required observations are missing, stale or incomplete; not proof of health |
| Upgrade eligible | Existing requirement: two distinct reported versions and at least one Completed baseline; excludes initial installs |

Prefer independent `HostedCluster.status.controlPlaneVersion.history` when
available; otherwise retain the existing legacy history behavior. ARM operation
success and worker health are separate contracts.

The existing export cap of three active-history entries remains unchanged.
Verify Completed-baseline visibility for deeper Partial histories; absence of
an alert is not proof of health.

### Detection and clearing

| Observation or transition | Monitor/alert behavior |
| --- | --- |
| Eligible Desired remains unstarted for more than 20 minutes | Monitor 1; enter five-minute Pending |
| Target remains incomplete for more than 30 minutes from earliest eligible Desired/Partial observation | Monitor 1; enter five-minute Pending |
| Within budget with known observations | Monitor 0 |
| Threshold remains exceeded through Pending | Fire; establish the persistent cluster latch |
| A phase transition puts a transient stall back within budget during Pending | Cancel Pending |
| Phase or target changes after firing, including a younger incomplete target | Retain the same firing cluster condition |
| Desired=current+Completed has three valid observations | Emit 0; detector no longer matches |
| Deleting/uninstalling or authoritative removal has three valid observations | Emit 0 for lifecycle exclusion, not upgrade success |
| Missing/incomplete/stale observations | Hold prior 0/1 and pause confirmation progress |
| Valid contrary observation | Reset that reason's confirmation count |

Completion time includes Desired time: Desired at 00:00 and Partial at 00:10
exceeds the completion budget just after 00:30, not 00:40.

Completion and lifecycle counters are independent, bounded at three and
evaluated once per minute. Fresh observations must be less than 60 seconds old
and pass coverage/coherence checks. Missing evaluations pause progress; three
observations are not three uninterrupted wall-clock minutes.

Clearing requires current evidence as well as a stored confirmation count.
Old counts alone cannot newly clear when current observations are unavailable.
Dispatch stops immediately on confirmed teardown; it does not wait for these
alert-clearing confirmations.

## Observation completeness

The target count, rows, convergence and version-state timestamps must agree.
Version telemetry independently discovers the source set, so a source losing
all target/count/convergence families contributes an incomplete zero instead
of disappearing from validation. A complete replica cannot hide another
source's unfinished/unknown state.

Inventory completeness likewise requires expected counts and matching scrape
timestamps. Malformed entries remain in the expected total while valid rows
and unrelated backend metrics remain available. A partial inventory cannot
prove removal; positive lifecycle evidence on a valid row can still be used.

Retire a removed resource's monitor series only after publishing a confirmed
clear zero. The primary inventory is an informer snapshot, not an independent
live Cosmos read; initial synchronization does not establish ongoing watcher
freshness.

## Evaluator outages

The latch, prior monitor value and confirmation counters recover their latest
stored values through an explicit 24-hour lookup. A newer 0/reset wins over
an older 1/progress value; this is not a historical maximum latch.

After restoration, reconcile through the same normal completion/lifecycle
checks. There is no separate recovery timer or counter. A younger incomplete
target retains firing; unknown evidence holds state.

This requires retained recording history in the same store and supports gaps
shorter than 24 hours. Monitoring-store data loss or longer outages cannot be
claimed as confirmed state recovery. Measure query cost before cutover.

## Existing alerts and compatibility

All five existing recording names and expressions are preserved. Thirteen
additional outputs calculate source completeness and the new cluster monitor;
the existing per-version recordings remain diagnostic and legacy inputs.

The deployed legacy baseline includes the deleting-state exclusion from
[Azure/ARO-HCP#8004](https://github.com/Azure/ARO-HCP/pull/8004). The proposed
detector uses the new lifecycle evidence and three-observation contract instead.

The existing RP group identity is retained but its two upgrade detectors and
version-specific correlation keys are replaced. Existing incidents, silences,
old SLO labels and correlation migration need an explicit cutover decision.
Disabling/replacing the old rules does not prove that their incidents close.

## Verification before a Phase 2 PR/cutover

1. Deploy Phase 1 through approved SDP pipelines. Confirm raw metric coverage,
   snapshot consistency, lifecycle visibility, polling load and series churn.
2. Collect seven usable days after deployment and required warm-up. Missing
   regions/scenarios must be reported as unavailable or unobserved, not passed.
3. Run the actual-rule Go/promtool regressions. Inject missing scrapes, stale
   samples, partial/all target-family loss, malformed inventory, evaluator gaps,
   counter pause/reset, rollback and retirement cases.
4. Use `alert-tester` for historical characterization. Do not recursively inline
   self-referencing recordings or describe raw observations alone as an exact
   replay of an undeployed state machine.
5. To evaluate stateful recordings on historical input, use an explicitly
   verified chronological replay harness with initialization and stale-sample
   semantics, or run the approved recordings in a non-paging test deployment
   and query their actual output history. `atest` alone does not provide that
   harness. Do not stop or alter production telemetry to create test gaps.
6. Validate the candidate detector without incident delivery using an approved
   test scope/process. No mode-switch scaffold is supplied by this branch.
7. Review recordings, detector, correlation migration and receiver behavior
   together; only then approve SDP cutover and verify real 1-to-0 mitigation.

`atest` models query-result firings; its missing-sample behavior and
resolution-delay grouping are not a proof of this monitor's unknown-state
contract or actual IcM lifecycle. Its incident count is distinct correlation
keys, not actual ticket creation/closure. Inspect exact firing boundaries,
not only aggregate/replay counts.

The on-demand [validation prompt](../../.github/prompts/validate-cluster-upgrade-monitor.prompt.md)
orchestrates these checks and distinguishes deterministic tests, observed
production evidence and blocked/unobserved scenarios.

## Test ownership and limits

- Recording tests verify emitted 0/1, count and coverage state.
- `tooling/prometheus-rules/internal/upgrade_monitor_regression_test.go` reads
  actual production expressions and exercises missing recording evaluations.
- Native alert fixtures verify Pending/Firing behavior and stable labels.
- No local test proves Azure receiver delivery, production rollout or
  downstream IcM mitigation latency.

No broader cluster-health gate, generic failed-state suppression, automatic
remediation or relabeling historical Partial attempts as successful is proposed.
