# MEP: Resource-Group Balance Epoch

- **Created:** 2026-07-12
- **Author(s):** @xiaofanluan
- **Status:** Draft
- **Component:** Coordinator
- **Related Issues:** [milvus-io/milvus#51244](https://github.com/milvus-io/milvus/issues/51244)
- **Related Pull Requests:** [milvus-io/milvus#49861](https://github.com/milvus-io/milvus/pull/49861), [milvus-io/milvus#50774](https://github.com/milvus-io/milvus/pull/50774)
- **Released:** Not released

## Summary

This MEP introduces a resource-group-scoped balance epoch for QueryCoord.

Today, QueryCoord periodically generates balance plans collection by collection and submits them directly to the task scheduler. A later checker iteration can plan again while an earlier set of Grow and Reduce actions is still being executed or has only partially appeared in QueryNode distribution. Scheduler task deltas reduce this problem, but they do not provide a consistent planning snapshot, a bounded planning wave, or a clear boundary between acting and observing.

The proposed control loop is:

```text
Observe -> Plan -> Admit -> Execute -> Reconcile -> Observe again
```

Each Resource Group (RG) is an independent balance group and may have at most one current normal-balance planning generation. An epoch:

1. captures a versioned placement snapshot for the RG;
2. generates a bounded wave of channel and segment plans;
3. admits tasks while revalidating their preconditions;
4. observes the authoritative QueryNode distribution and classifies task outcomes;
5. reconciles completed work and carries unresolved safe-settle work forward as locked pending work;
6. ends as completed, degraded, superseded, or timed out; and
7. allows the next epoch to plan unrelated objects from a newly reconciled snapshot.

Different RGs can run epochs concurrently. Stopping balance and failure recovery have higher priority and may invalidate or preempt a normal epoch.

This MEP changes the orchestration and correctness boundary of balancing. It does not define a new resource-aware scoring function, shard placement policy, or migration-cost model. Those policies can be implemented separately on top of the snapshot and epoch interfaces defined here.

## Motivation

### Current behavior

As of Milvus master commit `158b1dc38a962837dfed1a36925e3d00910342be`:

- `BalanceChecker` periodically selects collections and replicas, invokes the configured balancer, converts plans to tasks, and calls `Scheduler.Add` directly.
- Stopping balance has priority over normal balance.
- The default normal policy is `ChannelLevelScoreBalancer`.
- Channel planning is performed before segment planning. If channel plans are generated, segment planning is deferred to a later checker iteration.
- A segment move is represented as ordered `Grow(target)` and `Reduce(source)` actions.
- RPC completion is not authoritative task completion. QueryCoord waits for QueryNode distribution to contain the target before Reduce, and waits for the source copy to disappear before completing the task.
- Pending segment-task deltas are distribution-aware after PR #49861, preventing Grow and Reduce effects that are already reflected in distribution from being counted twice.

These mechanisms are necessary but do not form a complete closed-loop balance protocol.

### Problems

#### Planning may overlap execution feedback

QueryNode distribution is pulled asynchronously. A checker iteration can observe a mixture of:

- actions that have not started;
- Grow RPCs that returned but are not visible in distribution;
- target copies that are visible while source copies still exist;
- actions with RPC errors, RPC deadlines, or external cancellation;
- node or shard-leader changes; and
- target or replica metadata from a different logical point in time.

Task deltas project some in-flight effects, but a new planning round still has no explicit dependency on the completion or reconciliation of the previous planning round.

#### Independently valid plans may conflict as a batch

Plans generated for different collections or replicas in the same RG compete for the same QueryNode capacity. If they are generated independently from the same pre-move state, multiple plans may select the same apparently underloaded node. Their combined result may provide little or no net redistribution even when each individual decision appeared beneficial.

#### Generated work is not the same as admitted work

The current checker counts generated task objects before scheduler admission. `Scheduler.Add` may reject a task because of deduplication, a stale source, a missing leader, or another state change. Admission errors are currently ignored by `BalanceChecker`, so control-loop accounting may advance even when no work was accepted.

#### Existing batch limits are not hard wave boundaries

Batch limits are checked before processing a collection. One collection with multiple replicas, shards, or outbound nodes can generate more tasks than the nominal checker-wide limit. There is no RG-wide movement or task reservation shared by all plans in the round.

#### Failure handling lacks an epoch-level reconciliation boundary

A failed Grow is normally safe because the source remains. A failed Reduce leaves a safe but redundant copy. A node failure, RG membership change, target change, or shard-leader change can invalidate the assumptions of many plans at once. These cases require different handling, but the current periodic checker has no explicit epoch state describing whether it is still executing, reconciling, or invalidated.

### Goals

This MEP has the following goals:

1. Establish one current closed-loop normal-balance planning generation per RG.
2. Prevent a new normal planning wave from ignoring unresolved results from the previous wave.
3. Build each wave from one immutable, versioned placement snapshot.
4. Count and reserve only successfully admitted tasks.
5. Enforce hard per-epoch task limits and provide an extension point for byte/resource budgets.
6. Define deterministic epoch behavior for task failure, ambiguous RPC outcomes, node failure, target changes, and RG topology changes.
7. Preserve the existing Grow-before-Reduce availability invariant.
8. Allow different RGs to balance independently and concurrently.
9. Allow stopping balance and recovery to preempt normal balancing.
10. Prevent one permanently slow or bad object from blocking unrelated objects in the same RG.
11. Provide metrics that distinguish planning, admission, execution, distribution waiting, reconciliation, and convergence.

### Non-goals

This MEP does not:

- replace `ChannelLevelScoreBalancer` or define a new score;
- add per-segment memory or disk estimation;
- define small-cluster versus shard-local placement;
- optimize migration byte cost;
- make a balance wave transactional or roll back all successful moves after one task fails;
- persist an in-progress epoch across QueryCoord restart; or
- serialize unrelated RGs behind a cluster-global balance lock.

## Terminology

### Balance group

A balance group is one Resource Group. QueryNode capacity and replica placement are isolated by RG, so normal placement decisions in different RGs can proceed independently.

### Placement snapshot

An immutable, versioned view of the RG state used to produce one epoch plan. It contains actual distribution, desired target state, topology, pending tasks, and node eligibility.

### Balance epoch

A versioned planning and admission generation for one RG. An epoch is not a transaction and is not an all-task completion barrier. Tasks from an older epoch may continue to a safe point after that epoch is superseded, but their objects and resource reservations remain visible and locked in later snapshots.

### Wave

The set of channel and segment move tasks admitted by one epoch. Plans are simulated together before admission and share one task/resource budget.

### Reconciliation

The process of reading authoritative distribution and scheduler state after task execution, failure, timeout, or invalidation. Reconciliation classifies each object as resolved or carried forward so the next epoch can plan without reusing uncertain capacity or moving the same object again.

## Design Details

### Architecture overview

```text
                         +----------------------+
                         | Balance Trigger      |
                         | periodic / manual    |
                         +----------+-----------+
                                    |
                                    v
                    +---------------+----------------+
                    | BalanceEpochManager             |
                    | active epoch keyed by RG        |
                    +---------------+----------------+
                                    |
              +---------------------+---------------------+
              |                                           |
              v                                           v
  +-----------+------------+                 +------------+-----------+
  | PlacementSnapshotBuilder|                 | Epoch Event Handler    |
  | dist/target/topology/    |                 | node/target/RG changes |
  | pending task versions    |                 | task outcomes/deadline|
  +-----------+-------------+                 +------------+-----------+
              |                                           |
              v                                           |
  +-----------+-------------+                             |
  | BalancePlanner          |                             |
  | channel/segment plans   |                             |
  | simulated as one wave   |                             |
  +-----------+-------------+                             |
              |                                           |
              v                                           |
  +-----------+-------------+                             |
  | Epoch Admission         |<----------------------------+
  | revalidate + reserve    |
  +-----------+-------------+
              |
              v
  +-----------+-------------+
  | Existing Task Scheduler |
  | Grow -> dist -> Reduce   |
  +-----------+-------------+
              |
              v
  +-----------+-------------+
  | Distribution Feedback   |
  +-------------------------+
```

### Epoch scope and concurrency

The balance-group key is the RG name. Each RG has at most one current normal planning generation:

```text
RG-A -> epoch 101, Executing; epoch 100 has one safe-settle task
RG-B -> epoch 38, Reconciling
RG-C -> no active epoch, converged
```

The RG boundary is chosen because all replicas assigned to an RG compete for the same QueryNodes. Planning only per collection cannot reserve shared destination capacity. A cluster-wide epoch is unnecessarily coarse because independent RGs do not share QueryNodes.

A collection can have replicas in different RGs. These replicas participate in the epoch of their assigned RG and do not force epochs in different RGs to serialize.

Moving a node between RGs or moving a replica to another RG is a topology operation, not a normal segment-balance action. It invalidates epochs in the affected RGs. Those RGs reconcile independently after the topology operation.

### Epoch identity

An epoch ID is scoped to the RG and the current QueryCoord leader term:

```go
type BalanceEpochID struct {
    ResourceGroup string
    LeaderTerm    uint64
    Sequence      uint64
}
```

`LeaderTerm` may be implemented as the QueryCoord leadership term or a unique process boot ID. It prevents a delayed completion callback from an old QueryCoord leader from mutating a new leader's epoch. The sequence is used for task attribution, stale-event rejection, logs, and metrics. Neither field replaces distribution, target, or topology versions.

### Placement snapshot

The planner consumes one immutable snapshot:

```go
type PlacementSnapshot struct {
    ResourceGroup string
    LeaderTerm    uint64

    RGSegmentRevision  uint64
    RGChannelRevision  uint64
    TargetVersions     map[int64]int64
    ReplicaVersions    map[int64]int64
    RGVersion          uint64

    Nodes       map[int64]NodeSnapshot
    Replicas    map[int64]ReplicaSnapshot
    Segments    map[SegmentObjectKey]SegmentPlacement
    Channels    map[ChannelObjectKey]ChannelPlacement
    PendingWork PendingWorkSnapshot
}

type SegmentObjectKey struct {
    ReplicaID int64
    SegmentID int64
    Scope     DataScope
}

type ChannelObjectKey struct {
    ReplicaID   int64
    ChannelName string
}
```

Segment and channel distribution revisions remain separate. The current score cache may use their monotonic sum as a coarse change detector, but an epoch needs the separate values to identify which scoped subsystem changed, validate snapshot capture, and provide useful diagnostics.

The revisions must be scoped to the RG, or the snapshot builder must compute a stable digest over the distribution records copied into this RG snapshot. Using the current cluster-global distribution version directly would allow unrelated activity in another RG to invalidate this RG repeatedly.

The first implementation may construct a snapshot using optimistic version validation:

1. read all RG-scoped revision and metadata-version tokens;
2. copy the required distribution, target, replica, RG, node, and pending-task data;
3. read all version tokens again; and
4. accept the snapshot only if every scoped token is unchanged.

If a required manager does not expose a scoped revision, it must add one, provide a digest of the copied records, or participate in a higher-level snapshot lock. The snapshot builder retries when concurrent relevant updates are detected. It must copy manager-owned slices and maps before returning them to the planner.

Revision equality is used to validate snapshot capture, not as a rule that aborts an executing epoch whenever distribution changes. The epoch's own Grow and Reduce actions are expected to advance distribution. Runtime invalidation is driven by the scoped event matrix below.

### Pending work

The snapshot includes all admitted tasks whose effects are not fully reflected in distribution. This includes tasks created outside normal balance when they affect the same segments, channels, or nodes.

Pending effects remain distribution-aware:

- a Grow effect is pending until the target distribution contains the resource;
- a Reduce effect is pending until the source distribution no longer contains the resource; and
- an ambiguous RPC remains pending until reconciliation resolves the actual distribution.

Current segment task deltas already filter action effects that distribution has absorbed. Current channel task deltas do not: they retain the full Grow/Reduce count until the whole task is removed. The first epoch implementation must either extend `ChannelTaskDelta` with action records equivalent to segment deltas or have `PlacementSnapshotBuilder` independently recompute channel effects against distribution. A snapshot must not double-count a channel Grow that is already visible.

Pending work is not required to belong to the current epoch. A task carried over from an older generation keeps:

- an epoch-owned object lock keyed by `SegmentObjectKey` or `ChannelObjectKey`;
- its destination-capacity reservation until distribution resolves the Grow outcome; and
- its source/target projected effects in later snapshots.

This allows the current planning generation to work on unrelated objects without pretending that old work disappeared.

The object lock and reservation are owned by `BalanceEpochManager`, not by the current scheduler task record. Today, an RPC error can make the scheduler fail and remove a task, which also removes its scheduler dedup index and task delta. Epoch safety therefore cannot depend on that scheduler entry surviving. The epoch-owned record remains until distribution resolves the outcome, even if the underlying scheduler task has already been removed.

In the first implementation, a capacity reservation consists of scheduler task slots plus the row/count workload effects understood by the configured current policy. This MEP does not claim exact byte-level admission. A future resource-aware planner may extend the same structure with memory, disk, GPU, and loading-byte reservations.

### Planning one wave

The planner receives the placement snapshot and one RG-wide budget. Balance policy evaluation must use the immutable snapshot view; it must not re-read live distribution managers while planning. Existing policy behavior can be preserved through snapshot-backed adapters, but the existing `BalanceReplica(ctx, replica)` interface is insufficient as the epoch planner contract because its implementations read mutable global managers internally.

The epoch-facing interface is conceptually:

```go
type BalancePlanner interface {
    Plan(snapshot *PlacementSnapshot, budget BalanceWaveBudget) BalanceWave
}
```

The first implementation adapts the configured current policy to this interface and retains its scoring semantics. All returned plans are collected and simulated against one projected placement before admission.

```go
type BalanceWaveBudget struct {
    MaxSegmentTasks int
    MaxChannelTasks int
    MaxTasksPerNode int
    MaxTasksPerCollection int
}
```

The budget is a hard maximum. Plan generation stops or trims deterministically when the remaining budget is exhausted. A future MEP may add `MaxLoadBytes`, `MaxMemoryReservation`, and `MaxDiskReservation` without changing epoch lifecycle semantics.

`MaxTasksPerNode` counts placement actions by the node whose placement changes: a move consumes one Grow slot on the target and one Reduce slot on the source. It does not count the shard leader used as the RPC routing hop. Existing executor concurrency remains a separate runtime limit.

Carry-over and non-balance pending work is deducted from relevant node/object capacity before calculating the new wave budget. Within a wave:

1. later plans see the projected effects of earlier accepted plans;
2. the same replica-scoped segment or channel object cannot appear in multiple plans;
3. destination eligibility is checked against projected state and reservations;
4. plans have deterministic ordering; and
5. the planner records why each plan was selected or skipped.

This MEP does not require segment and channel plans to execute simultaneously. The planner may preserve channel-first policy semantics, but both plan types are accounted for by the same epoch and budget. Repeated channel waves therefore cannot cause an unrelated, overlapping normal epoch to start before reconciliation.

### Admission

Planning does not change scheduler state. A task belongs to the epoch only after scheduler admission succeeds.

For every plan, admission revalidates at least:

- the active epoch ID for the RG;
- RG membership of source and target nodes;
- replica membership and version;
- target version;
- segment or channel identity and expected source;
- shard/domain eligibility required by the configured policy;
- source presence for a move;
- target node state and resource-exhaustion penalty;
- scheduler deduplication; and
- remaining epoch budget.

All normal-balance and stopping-balance task producers must use one admission gateway. A checker must not bypass the gateway and call `Scheduler.Add` directly, because bypassing it would make RG budgets and reservations advisory rather than hard constraints.

The proposed commit interface is conceptually:

```go
type AdmissionToken struct {
    EpochID            BalanceEpochID
    RGVersion          uint64
    TargetVersion      int64
    ReplicaVersion     int64
    SegmentKey         *SegmentObjectKey
    ChannelKey         *ChannelObjectKey
    ExpectedSourceNode int64
}

type AdmissionReason int

const (
    AdmissionAccepted AdmissionReason = iota
    AdmissionDuplicate
    AdmissionSourceGone
    AdmissionLeaderMissing
    AdmissionReplicaChanged
    AdmissionRGChanged
    AdmissionTargetChanged
    AdmissionNodeIneligible
    AdmissionBudgetExhausted
)

type AdmissionResult struct {
    TaskID int64
    Reason AdmissionReason
}

TryAdmitBalanceTask(task Task, token AdmissionToken) AdmissionResult
```

`TryAdmitBalanceTask` is the linearization point. The task must not become visible to dispatcher queues until the token, scheduler deduplication key, epoch-owned reservation, and hard budget have been validated and installed. Topology or target changes after this point are handled as scoped invalidation events. Existing scheduler resource deduplication remains authoritative inside this operation.

The implementation must close the validation-to-enqueue TOCTOU window. It may either acquire version-owner read fences in a documented lock order, or use optimistic two-phase admission:

1. create a held, non-dispatchable task and reservation;
2. validate all token versions;
3. re-read the versions after registration;
4. atomically commit the task to dispatcher-visible queues if unchanged; otherwise roll back the held task and reservation.

In both implementations, no QueryNode action can be dispatched between token validation and admission commit.

Admission errors must be typed. Epoch state transitions must not parse error strings to distinguish a duplicate from a stale source or changed topology.

The epoch tracks separate collections of plans and tasks:

```text
planned  = planner output
admitted = AdmissionReason == AdmissionAccepted
rejected = any other typed AdmissionReason
```

Only admitted tasks consume the new-wave budget and appear in epoch completion accounting. Carry-over work is accounted separately before the budget is calculated. If a rejection indicates a stale snapshot, admission stops for plans in the invalid scope and enters reconciliation. A local rejection such as a duplicate task may be recorded while other independent plans continue, provided their preconditions remain valid.

Tasks admitted by an epoch carry metadata identifying the RG and epoch ID. This metadata is diagnostic and does not replace the scheduler's existing resource deduplication keys.

### Execution invariants

The epoch layer does not execute QueryNode RPCs directly. Admitted tasks continue through the existing scheduler and executor.

The following invariants must be preserved:

1. A move performs Grow before Reduce.
2. Reduce is not dispatched until authoritative distribution confirms the target copy.
3. For a channel move, the Grow-to-Reduce transition additionally waits until the new target is observed as the replica's shard leader/delegator.
4. RPC success alone does not complete an action.
5. An action with an ambiguous RPC outcome remains locked until distribution reconciliation.
6. The epoch planner never mutates distribution or manufactures pending-task deltas.

### Epoch state machine

```text
Idle
  |
  v
Planning
  | snapshot accepted and plans generated
  v
Admitting
  | at least one task admitted
  v
Executing / Observing
  | wave reaches quiescence, no-progress deadline,
  | or scoped invalidation occurs
  v
Reconciling
  | resolved work is committed to observed state;
  | unresolved safe-settle work is carried forward
  +---------------------+----------------------+-------------------+
  |                     |                      |                   |
  v                     v                      v                   v
Completed             Degraded             Superseded           TimedOut
```

If planning produces no improving plan and the snapshot remains valid, the epoch may transition directly from `Planning` to `Completed` with the reason `Converged`.

Planning and admission edge cases are explicit:

- Plans generated but every admission is `Duplicate`: reconcile existing pending work; finish `Completed` only if observed placement is already satisfied, otherwise `Degraded`.
- A stale-snapshot rejection before any admission: transition to `Superseded` through reconciliation.
- A scoped stale rejection after partial admission: retain admitted tasks, discard unadmitted plans in that scope, and reconcile before publishing the next generation.
- Deadline during `Planning` or `Admitting`: stop generation/admission and transition through reconciliation to `TimedOut`.
- An RG-wide invalidation in any non-terminal state: stop admission and transition to `Superseded`; already dispatched actions follow safe-settle rules.

Terminal state definitions:

| State | Meaning |
|---|---|
| `Completed` | The wave reached its desired observed state. Every admitted object is either complete or observed satisfied. |
| `Degraded` | Some objects failed or were quarantined, but resolved placement and carried-forward work are known, so unrelated work may be replanned. |
| `Superseded` | A newer generation is required because a planning assumption changed. Unstarted plans are discarded and unresolved running objects are carried forward. |
| `TimedOut` | The no-progress or epoch deadline expired. Unresolved objects are carried forward or quarantined before the next generation. |

All terminal states permit a later epoch. None permits the next epoch to reuse the old projected placement.

### Quiescence and generation handoff

An empty task set is not sufficient to declare an epoch complete. Before a terminal transition or handoff to a newer generation, the controller must verify:

1. every admitted object is terminal, observed satisfied, quarantined, or explicitly classified as carried-forward safe-settle work;
2. every ambiguous action retains its object lock and conservative capacity reservation;
3. distribution has advanced or been refreshed after the last relevant task event;
4. pending reservations have been removed or rebuilt from scheduler state;
5. actual placement is compared with the wave's successful outcomes; and
6. the next snapshot includes all unresolved work before planning unrelated objects.

The result determines `Completed`, `Degraded`, `Superseded`, or `TimedOut`. A stuck object does not block the whole RG indefinitely, but it cannot be moved again or have its reserved capacity reused while its outcome is uncertain.

### Epoch deadline

The epoch owns an explicit deadline. It does not depend on the currently ineffective timeout argument accepted by balance task constructors.

When the deadline expires:

1. stop admitting new plans;
2. cancel tasks that have not started when cancellation is safe;
3. do not assume that already dispatched RPCs were cancelled;
4. reconcile every ambiguous Grow or Reduce against distribution;
5. release or rebuild reservations; and
6. carry unresolved safe-settle tasks and their locks into the next snapshot; and
7. finish the epoch as `TimedOut`.

An in-flight Grow is not followed by Reduce unless target presence is confirmed. A completed Grow with a failed or cancelled Reduce is handled as a redundant-copy repair in the next epoch. The next epoch may plan unrelated objects while this repair remains locked.

### Scoped invalidation matrix

Runtime changes do not all invalidate the same scope:

| Event | Handling |
|---|---|
| RG node add/remove, RW-to-RO transition, or RG capacity/quota change | Hard-supersede the current RG generation because shared capacity assumptions changed. |
| Replica moved into or out of the RG | Supersede the affected RG generations. |
| Target version changed for one collection | Discard unadmitted plans for that collection and lock/reconcile its admitted objects; unrelated collections may continue or enter the next generation. |
| Channel-exclusive mapping changed | Invalidate the affected replica/shards, not unrelated replicas. |
| Expected distribution change caused by this epoch's task | Do not invalidate; reconcile it as expected feedback. |
| Recovery task conflicts with the same segment/channel | Recovery supersedes the normal task for that object using the recovery admission class and safe cancellation rules. |
| Unrelated collection distribution changed | Refresh projected state if needed, but do not automatically abort the RG generation. |
| QueryNode resource exhausted | Update node penalty/reservation assumptions and stop further admission to that node; replan affected work. |
| RPC timeout or temporary unavailable | Keep the object locked and reconcile/retry within its bounded policy; do not automatically invalidate unrelated plans. |
| Source resource disappeared | Mark the plan stale and reconcile that object. |
| Shard leader changed | Invalidate tasks and plans whose execution path depends on that leader; unrelated shards remain eligible. |
| QueryCoord leader term changed | Supersede all old epochs and ignore old-term callbacks. |

In particular, the implementation must not abort an epoch merely because a cluster-global distribution version changed. The epoch's own tasks and unrelated RGs both advance those versions.

### Failure handling

Epochs are not transactions. A successfully completed move is not rolled back because another move failed.

#### Grow/load failure

If Grow fails and the target is absent from distribution:

- keep the source copy;
- mark the task failed;
- release its destination reservation;
- record the destination's typed failure, including resource exhaustion when available; and
- allow independent tasks in the epoch to continue.

The next epoch may choose a different target. Repeated failures trigger the existing node resource-exhaustion penalty when applicable and the quarantine policy below.

#### No-progress handling and quarantine

The epoch layer must prevent one permanently failing object from monopolizing the RG control loop.

- The first implementation does not add in-place scheduler action retries. An RPC failure terminates the scheduler task according to current behavior.
- Epoch reconciliation classifies the result and a later generation may admit a newly planned task after backoff if the preconditions still make sense.
- Each object has a bounded number of consecutive epoch-level retries before quarantine.
- An object that reaches the no-progress deadline without an ambiguous in-flight RPC is quarantined for a configured backoff interval.
- An object with an ambiguous in-flight RPC remains locked and reserved rather than retried.
- Quarantine is cleared when its backoff expires or when a relevant topology, target, replica, leader, or node-penalty change provides new evidence that retry may succeed.
- Quarantined objects are reported as degraded placement, but they do not block unrelated collections, replicas, or shards.

Quarantine is a control-loop decision, not a declaration that the segment may be dropped. Existing availability and target checkers remain responsible for required copies.

#### Ambiguous Grow outcome

If the Grow RPC times out or loses its response, the task must not immediately retry or replan the segment. Reconciliation checks target distribution:

- target present: treat Grow as successful and continue or repair Reduce;
- target absent: treat Grow as failed and release the reservation.

#### Reduce failure

If target presence was confirmed but Reduce fails, the segment may exist on both source and target. This is availability-safe but consumes extra resources.

The epoch records a partial completion and ends as `Degraded` after reconciliation. The next epoch decides which copy to keep based on current target, topology, and policy. It must not automatically generate a reverse move merely because Reduce failed.

#### Node failure

A node failure invalidates the RG snapshot and immediately stops normal admission:

1. mark the current generation `Superseding`;
2. cancel not-started normal-balance tasks when safe;
3. leave dispatched actions to distribution-based reconciliation;
4. run or allow higher-priority recovery for missing channels and segments;
5. rebuild authoritative RG state; and
6. end the normal epoch as `Superseded`.

Recovery does not wait for the normal epoch to reach its original deadline.

#### Shard leader change

A leader change invalidates tasks whose execution path depends on the old leader. The controller stops admission for the affected shard/replica and enters reconciliation for those objects. Existing scheduler safety checks continue to reject Reduce through an unexpected leader. Unrelated shards in the RG are not automatically invalidated.

#### Target, replica, or RG membership change

A target version change invalidates plans for the affected collection. A replica topology change invalidates the affected replica. An RG membership or capacity change supersedes the whole RG planning generation because every destination-capacity assumption may have changed. Unadmitted plans in the invalid scope are discarded. Admitted tasks are reconciled according to their actual stage; recovery and target-consistency checkers retain higher priority.

### Normal versus recovery priority

Normal balance is opportunistic. Stopping balance and recovery protect availability and topology correctness. They share RG admission capacity but use separate priority lanes and object-level conflict scopes.

This behavior cannot be implemented using current `TaskPriority` values alone because normal and stopping tasks of the same resource type may have equal priority, while scheduler replacement currently requires strictly higher priority. The epoch design therefore introduces an admission class independent of execution priority:

```text
AdmissionClassRecovery > AdmissionClassNormal
```

For the same replica-scoped object key, a recovery admission may cancel a not-started normal task or supersede its epoch-owned lock. A running normal action is not forcibly killed; it reaches a safe point and is reconciled. Execution-pool priority may continue to use existing task priority after admission.

Priority rules:

1. A normal epoch never blocks node-down recovery or stopping balance.
2. Recovery reserves admission capacity before normal balance and may replace a lower-admission-class task on the same scheduler deduplication key.
3. An RG-wide capacity/topology event supersedes the current normal generation.
4. A single missing segment, leader change, or collection target change freezes only its conflicting object/replica scope; unrelated normal work may continue when the RG remains serviceable.
5. Recovery may bypass normal cooldown and movement-benefit thresholds.
6. Recovery still preserves Grow-before-Reduce and destination eligibility.
7. When the RG is not serviceable, recovery may consume the full admission budget.

The initial implementation may keep existing stopping and recovery checkers as plan producers. `BalanceEpochManager` coordinates their priority and invalidation effects without requiring those checkers to share the normal scoring policy.

### QueryCoord restart

Balance epochs are in-memory control-loop state and are not persisted.

After QueryCoord restart:

1. recover target, replica, RG, and QueryNode distribution using existing recovery paths;
2. establish a new leader term and ignore delayed in-process events carrying an older term;
3. discard old scheduler-task, epoch, object-lock, reservation, and projected-placement state because it was not persisted;
4. treat any redundant or missing copies visible in recovered distribution as fresh repair input; and
5. create new epochs only after initial distribution recovery completes.

This avoids treating an old projected plan as durable desired state. Target metadata and observed QueryNode distribution remain the sources of truth.

An old leader's already-dispatched RPC may still take effect after the new leader starts; `LeaderTerm` cannot prevent that external side effect. A later distribution pull observes it, and the new leader's recovery/redundancy logic reconciles the resulting actual placement.

### Proposed components

#### BalanceEpochManager

Responsibilities:

- owns the current planning generation and carry-over work map keyed by RG;
- serializes planning and admission within one RG;
- permits epochs in different RGs to proceed concurrently;
- receives topology and task events;
- manages deadlines, supersession, quarantine, and terminal transitions; and
- exposes epoch status and metrics.

#### PlacementSnapshotBuilder

Responsibilities:

- captures immutable distribution, target, replica, RG, node, and pending-task state;
- validates the version tuple before publishing a snapshot; and
- retries or reports a transient failure when a consistent snapshot cannot be obtained.

#### BalancePlanner

Responsibilities:

- consumes only immutable snapshot data;
- invokes a snapshot-backed adapter for the configured channel/segment policy;
- simulates all accepted plans in one projected placement;
- enforces the hard wave budget; and
- returns deterministic plans and explanations.

#### EpochReconciler

Responsibilities:

- classifies task outcomes using scheduler and distribution state;
- resolves ambiguous RPC outcomes;
- rebuilds reservations;
- determines the terminal epoch state; and
- produces the authoritative input boundary for the next epoch.

### Integration with BalanceChecker

`BalanceChecker` remains a trigger and eligibility component but no longer submits normal balance tasks directly.

The proposed flow is:

```text
BalanceChecker tick/manual trigger
    -> identify RGs eligible for normal balance
    -> BalanceEpochManager.TryStart(resourceGroup)
    -> snapshot / plan / admission / execution / reconciliation
```

If an RG already has a current normal planning generation, repeated periodic triggers are coalesced. They do not create another concurrent planner for the RG. A trigger received during reconciliation may request the next generation after resolved and carry-over state has been published.

Stopping balance can continue to run on its more frequent trigger. Before submitting recovery work, it reserves recovery-lane capacity and supersedes conflicting normal objects. Node removal or another RG-wide capacity change supersedes the whole current planning generation.

### Observability

The implementation should expose at least:

- active epoch ID and state per RG;
- epoch duration and reconciliation duration;
- snapshot retries and invalidation reasons;
- planned, admitted, rejected, running, waiting-distribution, completed, and failed task counts;
- admission rejection reasons;
- epoch completion state and reason;
- number of normal epochs preempted by stopping/recovery;
- count and age of tasks carried over from older generations;
- quarantined object and node counts with reasons;
- number and age of ambiguous RPC outcomes;
- number of redundant copies left after partial moves; and
- convergence age per RG.

Logs for each plan should include RG, epoch ID, collection, replica, shard, segment/channel, source, target, admission result, and terminal outcome.

## Public Interfaces

This proposal does not change the Milvus user-facing API.

Internal interfaces will be added or extended to support:

- snapshot version tuples;
- RG and epoch metadata on balance tasks;
- task terminal/admission notifications or equivalent polling APIs;
- typed admission results and recovery-versus-normal admission classes;
- cancellation of not-started tasks by epoch/object scope;
- epoch invalidation events for RG, replica, target, node, and leader changes; and
- epoch status in QueryCoord diagnostics.

Configuration should include:

```text
queryCoord.balanceEpoch.enabled
queryCoord.balanceEpoch.deadline
queryCoord.balanceEpoch.noProgressDeadline
queryCoord.balanceEpoch.maxSegmentTasks
queryCoord.balanceEpoch.maxChannelTasks
queryCoord.balanceEpoch.maxTasksPerNode
queryCoord.balanceEpoch.maxTasksPerCollection
queryCoord.balanceEpoch.maxObjectRetries
queryCoord.balanceEpoch.quarantineBackoff
```

Existing balance batch-size settings can be used as initial defaults for the epoch limits. The epoch layer treats them as hard limits.

## Compatibility, Deprecation, and Migration Plan

The feature is internal and does not change collection, replica, or search APIs.

The minimum deliverable is deliberately limited to the normal-balance closed loop:

1. RG-scoped immutable snapshot capture;
2. snapshot-backed adaptation of the existing configured policy;
3. one current normal planning generation per RG;
4. typed, atomic admission with hard task budgets;
5. epoch-owned replica-scoped object locks and pending reservations;
6. distribution-based reconciliation, deadline, and generation handoff; and
7. planned-versus-admitted observability.

Recovery admission classes, epoch-scoped cancellation, and fine-grained recovery integration are required by the final design but are delivered in the later recovery-integration phase. Resource vectors, migration byte budgets, and new placement objectives remain separate MEPs.

Rollout is divided into phases:

1. **Instrumentation:** add epoch-compatible task attribution and distinguish planned from admitted work without changing scheduling behavior.
2. **Shadow planning:** build RG snapshots and waves, but compare plans and budgets without admitting epoch-generated tasks.
3. **Normal-balance epoch:** route normal balance through one epoch per RG while preserving the configured existing balance policy.
4. **Recovery integration:** allow stopping and recovery events to preempt/invalidate normal epochs through the common manager.
5. **Default enablement:** enable RG epochs by default after upgrade, failure, and scale tests demonstrate no convergence regression.

During rollout, disabling `queryCoord.balanceEpoch.enabled` stops creation of new epochs. The current generation first publishes reconciled and carry-over state; only then does QueryCoord restore the existing periodic normal-balance submission path. In-flight scheduler tasks continue to use existing safety and deduplication semantics. Epoch task metadata must be ignored safely by older QueryNodes because epoch orchestration remains inside QueryCoord.

Mixed-version QueryNode deployments are supported because the first version of the epoch protocol relies on existing distribution fields. Future resource-vector extensions require their own compatibility design.

## Test Plan

### Unit tests

- Snapshot construction retries when any member of the version tuple changes.
- Returned snapshot maps and slices are isolated from distribution-manager mutation.
- One RG cannot run two planning/admission generations concurrently.
- Different RGs can plan and execute concurrently.
- Old-generation safe-settle tasks remain locked and reserved while the current generation plans unrelated objects.
- Hard task limits cannot be exceeded by multiple collections, replicas, shards, or outbound nodes.
- Only successfully admitted tasks count toward epoch completion and budget.
- Deterministic plan ordering produces the same wave for the same snapshot.
- Duplicate and stale plans are rejected without corrupting reservations.
- Epoch deadline transitions through reconciliation before `TimedOut`.

### State-machine tests

- No-plan snapshot finishes as `Completed/Converged`.
- All tasks succeed and distribution matches: `Completed`.
- Grow fails before target presence: source remains and epoch becomes `Degraded`.
- Grow RPC times out but target later appears: reconciliation recognizes success.
- Reduce fails after target presence: redundant copy remains and epoch becomes `Degraded`.
- Node failure and RG membership change supersede the RG generation.
- A collection target change or leader change invalidates only its collection/replica/shard scope.
- Stopping balance preempts normal balance without violating Grow-before-Reduce.
- The epoch's own distribution updates do not self-invalidate the generation.
- Unrelated RG distribution updates do not invalidate the snapshot scope.
- QueryCoord restart discards projected epoch state and replans after distribution recovery.

### Integration tests

- Multiple collections in one RG select destinations under one shared wave budget.
- Multiple replicas and channels cannot exceed RG-wide task limits.
- A slow load does not allow repeated normal checker ticks to generate overlapping waves.
- A permanently stuck segment does not prevent unrelated collections in the same RG from progressing after the no-progress deadline.
- Scheduler admission rejection is visible in epoch accounting and does not suppress future planning indefinitely.
- Distribution pull delay between Grow and Reduce does not create a reverse move for the same segment.
- One failed task does not roll back unrelated successful moves.
- One failed RG epoch does not block epochs in other RGs.
- Recovery supersedes conflicting normal objects without cancelling already-running actions unsafely.

### Long-running and failure-injection tests

- Repeated QueryNode restarts and shard-leader changes while normal balance is active.
- RG scale-out and scale-in near balance trigger boundaries.
- Object-storage load failures and resource-exhaustion responses.
- Lost RPC responses where QueryNode applies the operation successfully.
- A production-scale distribution modeled after issue #51244, verifying bounded in-flight work and the absence of repeated reverse moves under a static target.

## Rejected Alternatives

### Cluster-global epoch

A single epoch for the entire QueryCoord would provide one global barrier but would couple unrelated RGs. One slow or failed load could block normal balancing for all tenants and resource pools. RG-level epochs preserve the required shared-capacity boundary without introducing cluster-wide head-of-line blocking.

### Per-collection epoch

Collections in the same RG compete for the same destination nodes. Independent collection epochs can reserve the same apparent headroom and reproduce conflicting plans. The balance group therefore must include all collections and replicas assigned to the RG.

### Per-replica or per-shard epoch

This granularity increases parallelism but does not provide shared RG capacity reservation. It may be added later as internal wave scheduling under one RG epoch, provided all sub-waves use the same snapshot and reservation ledger.

### Continue using task deltas without an epoch

Distribution-aware task deltas correctly project individual in-flight actions, but they do not define a consistent snapshot, hard wave boundary, admission accounting, failure reconciliation, or topology invalidation protocol.

### Transactional rollback

Rolling back every successful move after one task fails would create additional loads and releases, increasing cost and availability risk. Grow-before-Reduce makes most partial outcomes safe. The design keeps successful moves, reconciles actual placement, and replans from reality.

### Persist in-progress epochs

Persisting projected placement and action state introduces recovery complexity and can conflict with the authoritative QueryNode distribution after failover. The first version rebuilds from target, topology, scheduler state, and recovered distribution instead.

## References

- [Issue #51244: ScoreBasedBalancer never converges](https://github.com/milvus-io/milvus/issues/51244)
- [PR #49861: make balance workload delta dist aware](https://github.com/milvus-io/milvus/pull/49861)
- [PR #50774: 2.6 backport](https://github.com/milvus-io/milvus/pull/50774)
- `internal/querycoordv2/checkers/balance_checker.go`
- `internal/querycoordv2/balance/channel_level_score_balancer.go`
- `internal/querycoordv2/task/scheduler.go`
- `internal/querycoordv2/task/executor.go`
- `internal/querycoordv2/dist/dist_handler.go`
- `internal/querycoordv2/meta/segment_dist_manager.go`
- `internal/querycoordv2/meta/channel_dist_manager.go`
