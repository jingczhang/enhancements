# KEP-NNNN: Hugepage-Aware Memory Eviction

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Root Cause Analysis](#root-cause-analysis)
  - [Proposed Fix](#proposed-fix)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) within one minor version of promotion to GA
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

The kubelet eviction manager's `memory.available` signal overstates real
available regular memory on nodes that have hugepages configured. This is
because the root cgroup memory limit is set to `MemTotal` (which includes
hugepage-reserved RAM), while the cgroup memory controller does not account for
hugetlb allocations in `WorkingSet`. The result is that
`AvailableBytes = MemoryLimit - WorkingSet` includes "phantom" hugepage memory,
causing eviction to trigger later than intended and risking OOM kills.

Operators today work around this by inflating `evictionHard.memory.available` by
the total hugepage size. However, this inflated threshold feeds into the
`Allocatable[memory]` calculation, causing hugepage memory to be double-subtracted
and reducing scheduling capacity for non-hugepage pods.

This KEP proposes making the kubelet eviction signal hugepage-aware by
subtracting the node's total hugepage capacity from the reported
`AvailableBytes` in the summary stats provider. This fixes eviction timing
without requiring configuration workarounds and eliminates the density loss
caused by double-subtraction.

## Motivation

Hugepages are a Linux kernel feature that provides larger memory pages (e.g.,
2Mi or 1Gi) for applications that benefit from reduced TLB pressure.
Kubernetes manages hugepages as first-class resources: the scheduler accounts
for them, the kubelet advertises them in `Node.Status.Capacity`, and they are
subtracted from `Allocatable[memory]` so the scheduler does not over-commit
regular memory.

However, the kubelet eviction manager is not hugepage-aware. On a node with
hugepages configured, free hugepages appear as "available memory" to the
eviction signal, creating a gap between what the eviction manager considers
available and what is actually usable for regular allocations. This leads to
delayed eviction and potential OOM kills under memory pressure.

The only current workaround—inflating `evictionHard.memory.available` by the
total hugepage size—fixes eviction timing but introduces a secondary problem:
the inflated eviction threshold is included in the `nodeAllocatableReservation`
calculation, causing hugepage memory to be subtracted twice from
`Allocatable[memory]`. This reduces pod scheduling density by exactly the
hugepage reservation amount.

### Goals

* Make the `memory.available` eviction signal accurate on nodes with hugepages
  by subtracting hugepage capacity from reported available bytes.
* Eliminate the need for operators to inflate `evictionHard.memory.available`
  as a workaround for the hugepage phantom availability issue.
* Restore full pod scheduling density on hugepage-enabled nodes by removing
  the double-subtraction of hugepage memory from `Allocatable[memory]`.
* Gate the fix behind a feature gate (`HugepageAwareEviction`) for safe rollout.

### Non-Goals

* Changing how the scheduler handles hugepage resources.
* Modifying the Memory Manager's NUMA-aware hugepage allocation logic.
* Adding hugepage-specific eviction signals (e.g., a dedicated
  `hugepages.available` signal).
* Changing how cAdvisor or the kernel reports memory statistics.

## Proposal

Introduce a new function `adjustForHugePages` in the kubelet summary stats
provider (`pkg/kubelet/server/stats/summary.go`) that subtracts the node's
total hugepage capacity from `AvailableBytes` before the memory stats are
consumed by the eviction manager.

The adjustment is performed at the point where `NodeStats.Memory` is
constructed in both `Get()` and `GetCPUAndMemoryStats()`, ensuring all
consumers of the summary API (including the eviction manager) see corrected
values.

The function reads hugepage capacity from `Node.Status.Capacity` (which is
already available to the summary provider) and performs a simple subtraction,
clamping to zero if hugepage capacity exceeds available bytes.

### User Stories (Optional)

#### Story 1

As a cluster administrator running workloads that require 2Mi hugepages
alongside regular pods on the same nodes, I want eviction to trigger at the
correct real memory threshold without needing to manually inflate
`evictionHard.memory.available`. Currently, I must add the hugepage size to
the eviction threshold, which causes the node to report less allocatable
memory and prevents me from scheduling as many non-hugepage pods as the node
can actually handle.

#### Story 2

As a platform team managing a fleet of Kubernetes nodes with mixed workloads
(some requiring 1Gi hugepages, some not), I want consistent eviction behavior
that works correctly regardless of whether a node has hugepages configured.
The current inconsistency forces per-node-profile tuning of eviction
thresholds, increasing operational complexity.

### Notes/Constraints/Caveats (Optional)

**Why hugepages create phantom availability:**

1. cAdvisor reads `MemTotal` from `/proc/meminfo` and sets the root cgroup's
   `Memory.Limit` to this value. `MemTotal` includes hugepage-reserved RAM.
2. The cgroup memory controller tracks regular memory in `WorkingSet` but
   does **not** track hugetlb allocations (those are managed by the separate
   `hugetlb` cgroup controller).
3. The eviction signal computes `memory.available = Memory.Limit - WorkingSet`.
4. Since hugepage memory is in the limit but not in the working set, the
   result is inflated by exactly the hugepage reservation amount.

**Numerical example:**

| Parameter | Value |
|---|---|
| `MemTotal` | 64 GiB |
| Hugepages (2Mi × 2048) | 4 GiB |
| Regular memory available | 60 GiB |
| `evictionHard.memory.available` | 5754 Mi |

Without the fix, the eviction manager sees ~64 GiB total and subtracts
`WorkingSet` (which excludes hugepage usage). Eviction fires when
`64 GiB - WorkingSet < 5754 Mi`, i.e., when `WorkingSet > ~58.4 GiB`. But
only 60 GiB of regular memory exists, so actual regular memory is exhausted
when `WorkingSet ≈ 60 GiB`—leaving a ~1.6 GiB gap where eviction should have
fired but didn't, risking OOM kills.

**Workaround-induced density loss:**

Operators inflate the threshold to `5754 Mi + 4096 Mi = 9850 Mi`. This fixes
eviction timing, but the inflated 9850 Mi feeds into
`nodeAllocatableReservation`, which is subtracted from `Capacity[memory]`
alongside the explicit hugepage subtraction. The net effect is that 4 GiB of
hugepage memory is subtracted twice, reducing `Allocatable[memory]` by 4 GiB
more than necessary.

### Risks and Mitigations

**Risk:** Subtracting hugepage capacity could make `AvailableBytes` lower than
expected if hugepages are deallocated at runtime while the node still reports
them in `Capacity`.

**Mitigation:** Hugepage reservations are typically static and configured at
boot via kernel parameters. Dynamic hugepage changes are rare and the
adjustment clamps to zero, so this cannot cause negative values or panics.

**Risk:** The fix changes the effective eviction threshold for existing clusters
that have already applied the inflation workaround, potentially causing earlier
eviction.

**Mitigation:** The feature is gated behind `HugepageAwareEviction` (default
disabled). Operators should remove the workaround inflation from their
`evictionHard` configuration when enabling the feature gate. Documentation
will include migration guidance.

## Design Details

### Root Cause Analysis

The kubelet computes the `memory.available` eviction signal as:

```
memory.available = node.stats.memory.availableBytes
                 = root_cgroup.memory.limit - root_cgroup.memory.workingSet
```

- `root_cgroup.memory.limit` is set to `MemTotal` by cAdvisor
  (`vendor/github.com/google/cadvisor/container/raw/handler.go`), which reads
  from `/proc/meminfo`. `MemTotal` includes hugepage-reserved RAM.
- `root_cgroup.memory.workingSet` comes from the memory cgroup controller,
  which does **not** track hugetlb allocations (those use a separate hugetlb
  cgroup controller).

This means hugepage-reserved memory is counted in the limit but never in the
working set, creating "phantom" available bytes equal to the hugepage
reservation.

The eviction manager in `pkg/kubelet/eviction/helpers.go` maps
`SignalMemoryAvailable` directly to `v1.ResourceMemory` and reads
`AvailableBytes` from the node summary stats. It has no awareness of hugepages.

### Proposed Fix

Add a helper function `adjustForHugePages` to
`pkg/kubelet/server/stats/summary.go` that:

1. Sums all hugepage capacities from `Node.Status.Capacity` (keys prefixed
   with `hugepages-`).
2. Subtracts the total from `AvailableBytes`, clamping to zero.
3. Returns a **copy** of the `MemoryStats` (does not mutate the original).

This function is called in both `Get()` and `GetCPUAndMemoryStats()` when
constructing `NodeStats.Memory`:

```go
func adjustForHugePages(memory *statsapi.MemoryStats, node *v1.Node) *statsapi.MemoryStats {
    if memory == nil || memory.AvailableBytes == nil || node == nil {
        return memory
    }

    var totalHugePageBytes uint64
    for name, quantity := range node.Status.Capacity {
        if strings.HasPrefix(string(name), string(v1.ResourceHugePagesPrefix)) {
            totalHugePageBytes += uint64(quantity.Value())
        }
    }
    if totalHugePageBytes == 0 {
        return memory
    }

    adjusted := *memory
    if *adjusted.AvailableBytes > totalHugePageBytes {
        available := *adjusted.AvailableBytes - totalHugePageBytes
        adjusted.AvailableBytes = &available
    } else {
        available := uint64(0)
        adjusted.AvailableBytes = &available
    }
    return &adjusted
}
```

The call sites change from:

```go
Memory: rootStats.Memory,
```

to:

```go
Memory: adjustForHugePages(rootStats.Memory, node),
```

This is guarded by the `HugepageAwareEviction` feature gate.

### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes
necessary to implement this enhancement.

##### Prerequisite testing updates

None. The existing eviction unit tests and e2e tests provide sufficient
baseline coverage.

##### Unit tests

Unit tests for `adjustForHugePages` are added in
`pkg/kubelet/server/stats/summary_hugepages_test.go` (platform-independent
file, no build constraints) covering:

- Nil `MemoryStats` input
- Nil `AvailableBytes`
- Nil `Node`
- Node with no hugepage resources
- Node with a single hugepage size (e.g., `hugepages-2Mi`)
- Node with multiple hugepage sizes (e.g., `hugepages-2Mi` + `hugepages-1Gi`)
- Hugepage capacity exceeding available bytes (clamp to zero)
- Non-mutation of the original `MemoryStats` struct

- `pkg/kubelet/server/stats`: `2026-03-19` - coverage to be measured after merge

##### Integration tests

Integration tests are not specifically needed for this change. The adjustment
is a pure function with no external dependencies. The existing eviction
integration tests in `test/integration/eviction/` exercise the full eviction
pipeline and will validate the end-to-end behavior.

##### e2e tests

For Alpha, we will add a node e2e test that:
1. Configures hugepages on a test node.
2. Verifies that `memory.available` reported by the summary API correctly
   excludes hugepage capacity when the feature gate is enabled.
3. Verifies that eviction fires at the correct real memory threshold.

- `[sig-node] Eviction [Serial] hugepage-aware memory eviction`: to be added

### Graduation Criteria

#### Alpha

- Feature implemented behind the `HugepageAwareEviction` feature gate
- Unit tests for `adjustForHugePages` pass on all platforms
- Initial e2e test added to node e2e suite

#### Beta

- Feature gate enabled by default
- Gather feedback from operators running hugepage workloads
- Confirm no regressions in eviction behavior on nodes without hugepages
- Documentation updated with migration guidance for removing the inflation
  workaround
- e2e tests stable in CI for at least two releases

#### GA

- At least two releases at Beta with no reported issues
- Real-world validation from operators who previously used the inflation
  workaround
- Lock feature gate to true and remove in a subsequent release

### Upgrade / Downgrade Strategy

**Upgrade:** After enabling the `HugepageAwareEviction` feature gate, operators
should remove any hugepage inflation from their `evictionHard.memory.available`
and `reservedMemory` configurations. The eviction signal will now correctly
exclude hugepage memory, so the original (non-inflated) thresholds should be
used. `Allocatable[memory]` will increase by the hugepage amount, allowing more
pods to be scheduled.

**Downgrade:** If the feature gate is disabled (or kubelet is downgraded to a
version without this feature), operators must re-apply the inflation workaround
to `evictionHard.memory.available` to avoid late eviction. No data migration
is needed as the change is purely behavioral.

### Version Skew Strategy

This change is entirely local to the kubelet. There is no coordination needed
with the API server, scheduler, or controller manager. An n-3 kubelet without
this feature behaves exactly as today. An n-1 kubelet with this feature
enabled correctly adjusts its own eviction signal independently.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: `HugepageAwareEviction`
  - Components depending on the feature gate: kubelet

###### Does enabling the feature change any default behavior?

Yes. On nodes with hugepages configured, the `memory.available` eviction signal
will report a lower value (reduced by the total hugepage capacity). This means
eviction may trigger earlier than before on nodes that were relying on the
phantom hugepage availability. This is the correct behavior—without the fix,
eviction was triggering too late.

On nodes without hugepages, there is no behavioral change.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Setting `--feature-gates=HugepageAwareEviction=false` and restarting
kubelet reverts to the previous behavior. No persistent state is affected.
Operators should re-apply the eviction threshold inflation workaround if
disabling.

###### What happens if we reenable the feature if it was previously rolled back?

The hugepage adjustment is applied again. No persistent state is involved, so
re-enablement is safe. Operators should remove the inflation workaround when
re-enabling.

###### Are there any tests for feature enablement/disablement?

Unit tests exercise `adjustForHugePages` directly. Additional tests verifying
the feature gate toggle behavior will be added for Beta.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

A rollout could cause earlier eviction on nodes where the operator has
**both** the inflation workaround and the feature gate enabled. In this case,
the effective eviction threshold would be double-adjusted. This is mitigated
by documentation clearly stating that the inflation workaround must be removed
when the feature gate is enabled.

A rollback (disabling the feature gate) without re-adding the inflation
workaround could lead to late eviction on hugepage nodes. This is the same
behavior as before the feature existed.

###### What specific metrics should inform a rollback?

- Unexpected increase in OOM kills on hugepage nodes (indicates the
  workaround was removed but the feature gate was not enabled).
- Unexpected pod evictions on hugepage nodes (indicates double adjustment—
  both workaround and feature gate active).
- `eviction_signal_memory_available` metric showing unexpected values.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Will be tested as part of Beta graduation. The change is stateless, so
upgrade/downgrade is inherently safe.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

The feature is node-level, not workload-level. Operators can check:
- The kubelet feature gate configuration for `HugepageAwareEviction=true`.
- The `memory.available` value reported in the node summary API—on hugepage
  nodes, it will be lower by the total hugepage capacity compared to before.

###### How can someone using this feature know that it is working for their instance?

- [x] Other (treat as last resort)
  - Details: Compare the `memory.available` value from the kubelet summary API
    (`/stats/summary`) with and without the feature gate. With the gate enabled,
    `memory.available` should be lower by exactly the total hugepage capacity
    shown in `Node.Status.Capacity`.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

The eviction signal should accurately reflect available regular (non-hugepage)
memory within the existing eviction manager polling interval (default 10s).

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [x] Metrics
  - Metric name: `eviction_signal_memory_available` (existing)
  - Components exposing the metric: kubelet

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

A metric exposing the hugepage adjustment amount
(`eviction_hugepage_adjustment_bytes`) would be useful for debugging. This
will be added as part of the implementation.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No. The fix is entirely within the kubelet and relies only on data already
available (`Node.Status.Capacity` and cgroup memory stats).

### Scalability

###### Will enabling / using this feature result in any new API calls?

No. The `Node` object is already fetched by the summary provider.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No. The adjustment is a trivial O(n) iteration over hugepage resource keys in
`Node.Status.Capacity` (typically 1-2 entries) performed once per summary
stats collection cycle.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No. The computation is a simple integer subtraction.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

No impact. The adjustment uses `Node.Status.Capacity` which is cached locally
by the kubelet. API server unavailability does not affect the eviction signal
adjustment.

###### What are other known failure modes?

- Incorrect hugepage capacity in `Node.Status.Capacity`
  - Detection: `memory.available` from summary API does not match expected
    value (compare with `MemAvailable` from `/proc/meminfo` minus hugepage
    reservation).
  - Mitigations: Disable the feature gate and re-apply manual threshold
    inflation.
  - Diagnostics: kubelet logs at `--v=4` will show the summary stats values.
  - Testing: Unit tests verify correct behavior for all input combinations.

###### What steps should be taken if SLOs are not being met to determine the problem?

1. Check if the feature gate is enabled: `kubelet --feature-gates`.
2. Compare `memory.available` from `/stats/summary` endpoint with expected
   value based on `free -m` output and hugepage configuration.
3. Verify hugepage capacity in `kubectl describe node` matches actual
   kernel configuration (`/proc/meminfo` or `/sys/kernel/mm/hugepages/`).
4. If values are inconsistent, disable the feature gate and investigate.

## Implementation History

- 2026-03-19: KEP created (provisional)
- 2026-03-19: Initial implementation in `pkg/kubelet/server/stats/summary.go`
  with unit tests in `pkg/kubelet/server/stats/summary_hugepages_test.go`

## Drawbacks

The fix slightly increases the complexity of the summary stats provider.
However, the change is isolated to a single helper function and does not
affect the stats collection pipeline.

Operators with existing workarounds must update their configuration when
enabling the feature gate. This is a one-time migration cost that is offset
by the elimination of the workaround and improved pod density.

## Alternatives

**Alternative 1: Fix at the cAdvisor level.** Modify cAdvisor to set the root
cgroup memory limit to `MemTotal - HugePages_Total` instead of `MemTotal`.
This was rejected because it would affect all cAdvisor consumers, not just
the kubelet eviction manager, and could break other monitoring tools that
expect the limit to equal `MemTotal`.

**Alternative 2: Fix in the eviction manager.** Modify the eviction signal
observation code in `pkg/kubelet/eviction/helpers.go` to subtract hugepages.
This was rejected because the eviction manager does not currently have access
to `Node.Status.Capacity` and adding this dependency would increase coupling.

**Alternative 3: Add a dedicated `hugepages.available` eviction signal.**
This would allow operators to set separate thresholds for hugepage memory.
This was rejected as over-engineering for this bug fix—the core issue is that
the existing `memory.available` signal is incorrect, not that a new signal is
needed.

**Alternative 4: Kernel/cgroup fix.** Have the kernel include hugetlb
allocations in the memory cgroup's `WorkingSet`. This is outside the scope
of the Kubernetes project and would require kernel changes that may not be
accepted upstream.

## Infrastructure Needed (Optional)

None.
