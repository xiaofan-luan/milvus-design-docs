# MEP: Reduce bitmap memory in indexed scalar filters

- **Created:** 2026-10-20
- **Author(s):** @xiaofan-luan
- **Status:** Draft
- **Component:** QueryNode, Proxy
- **Related work:** Milvus PR #52138 (ARRAY contains predicate merging)
- **Released:** Not released

## Summary

Reduce the number of full-segment bitmaps retained by indexed scalar filter
expressions through two independent changes:

1. Read immutable validity data from sealed indexes instead of cloning a
   full-segment validity bitmap for every expression leaf.
2. Merge compatible ARRAY contains predicates during compilation. An OR of
   `ARRAY_CONTAINS` predicates becomes `ARRAY_CONTAINS_ANY`; an AND becomes
   `ARRAY_CONTAINS_ALL`.

The second optimization already exists on master in PR #52138. Its 2.6
adaptation must preserve the executor's type conversions and avoid repeatedly
flattening and merging long logical chains. This proposal documents the combined
memory model and the ownership requirements for the first optimization.

No public API, protobuf, persisted index format, or query result changes are
required. Both changes must remain useful independently.

## Motivation

An indexed leaf commonly computes a dense hit bitmap once for all rows, caches
it, and returns slices as the expression engine advances through batches. It
also retains an owned validity bitmap. For `L` evaluated leaves and `N` rows in
one segment, these two caches require approximately `2 * L * ceil(N / 8)` bytes,
before allocator overhead and batch results. Concurrent segment executions
multiply this cost even when CPU utilization is below its limit.

The hit bitmap depends on the predicate's values. A sealed index's validity
bitmap does not: many leaves can read the same immutable validity data. Similarly,
thousands of same-column contains predicates connected by OR do not require
thousands of cached hit bitmaps when the index already accepts a set of terms.

The intended improvement is lower simultaneously retained filter memory. This
does not claim that every native-memory peak or OOM has this cause, or that
lower allocation volume automatically improves end-to-end query latency.

## Design Details

### Immutable validity capability

Add an optional internal index capability that returns an immutable validity
handle. The handle represents one of:

- A known row count and an all-valid marker, without an allocated bitmap.
- A known row count and an owning `shared_ptr<const TargetBitmap>`.

An unsupported capability is distinct from the all-valid marker. In particular,
an empty bitmap used internally as an all-valid sentinel must not be interpreted
as an index containing zero rows.

The existing `IsNotNull()` API retains its owned, writable-result contract.
Callers that invert or otherwise modify its result must remain correct.

Sealed inverted indexes expose their finalized validity storage through the
new capability. Growing indexes use the existing snapshot behavior because
their row counts and null sets can change. Derived JSON indexes must opt in
only where their validity semantics match the exposed bitmap: ARRAY validity,
typed comparable-value masks, missing JSON paths, and JSON-flat executors are
not interchangeable. Unsupported derived implementations keep the old path.

### Expression consumption and ownership

An indexed expression first asks for the immutable capability. If available,
it retains the handle and copies only the validity slice needed by the current
batch or offset selection. An all-valid handle fills only the requested output
range. It must not call full-segment `IsNotNull()` and then slice as a fallback
on every batch.

The output batch is independently owned and writable. AND, OR, NOT, and final
predicate-to-exclusion-mask conversion must never mutate shared index storage.
The handle owns the bitmap; it cannot be a bare view into a temporary JSON
executor. Existing index pinning and the immutable owner must make lifetime
requirements explicit.

Unsupported readers retain one owned validity snapshot per leaf, with existing
cross-batch reuse. Arbitrary offset evaluation does not imply that the last
sequential batch has been consumed.

This change does not require a broad conversion of all bitset APIs to borrowed
views, nor a change to the existing batch scheduler.

### ARRAY contains compilation

For a homogeneous logical chain, group compatible predicates by the complete
column identity and execution semantics. Examples:

```text
ARRAY_CONTAINS(labels, "a") OR ARRAY_CONTAINS(labels, "b")
    -> ARRAY_CONTAINS_ANY(labels, ["a", "b"])

ARRAY_CONTAINS(labels, "a") AND ARRAY_CONTAINS(labels, "b")
    -> ARRAY_CONTAINS_ALL(labels, ["a", "b"])
```

The eligible physical field type is ARRAY. JSON fields are deliberately left
unchanged. The rewrite must not cross a NOT node or change the grouping of
mixed AND/OR expressions. Compatible pre-existing ANY or ALL predicates may
join their corresponding OR or AND group.

Preserve the first occurrence's position and stable value order. Deduplicate
only according to the target ARRAY element type and the executor's actual
numeric conversions; for example, FLOAT and DOUBLE do not have identical
conversion rules. Invalid, unsupported, unresolved, or non-scalar values must
not be silently discarded by the rewrite. Template substitution must happen
before a rewrite that depends on concrete values.

Flatten each same-operator chain once and merge once. Recursively merging an
increasing term list at every node of a left-deep chain creates quadratic work.
Rebuild a stack-safe logical tree while preserving unrelated rewrite behavior
and wrapper metadata.

`ContainsAny` uses one multi-term index query and retains one hit bitmap.
`ContainsAll` can retain an accumulator and combine one temporary term result
at a time. The latter still performs multiple term queries; this proposal does
not claim that it necessarily reduces index-query CPU.

### NULL and empty-value semantics

Rewriting must preserve three-valued logic. A NULL array, an empty array,
an empty candidate list, and an unsupported or mismatched value type have
different behavior and require explicit differential coverage. The merge
eligibility and empty-list behavior follow the verified executor semantics,
rather than a blanket assumption that every contains operation is two-valued.

The validity optimization must also preserve the distinction between missing
JSON paths, non-comparable values, and field-level NULLs in every fallback.

## Compatibility, Deprecation, and Migration Plan

The compiler optimization uses the existing expression optimizer enablement
and produces already-supported plan operators. No cluster-wide rollout order
or index rebuild is needed for the plan representation.

The immutable validity capability is internal and optional. Existing index
implementations continue to work without implementing it. Growing and
semantically different JSON readers preserve their previous behavior until
they have explicit implementations and tests.

The initial implementation targets the 2.6 incident path with a narrow patch.
Master's index-reader interfaces differ from 2.6 and require an explicit port;
the 2.6 class hierarchy must not be copied wholesale into master.

## Test Plan

### Correctness and lifetime

- Compare rewritten and original predicates for OR/AND, nested mixed operators,
  NOT boundaries, different columns, different ARRAY element types, templates,
  empty candidates, duplicate values, NULL arrays, and empty arrays.
- Exercise numeric conversion boundaries, non-finite values, and cases where
  the rewrite must decline optimization.
- Verify that a large synthetic same-column chain reduces to one contains leaf
  without losing unrelated predicates or metadata.
- Compare shared-validity and legacy results across multiple sequential
  batches, final partial batches, offset selections, and short-circuit skips.
- Mutate one returned batch and verify that the shared index bitmap and a
  second expression's result are unchanged.
- Retain a validity handle while releasing its producer and prove that the
  handle remains valid. Check all-valid, zero-row, nullable, JSON-derived, and
  growing fallback cases.

### Measurable benefit

- Measure compilation time and allocations for long synthetic contains chains,
  including more than two thousand terms, with optimization enabled/disabled.
- Use independent leaves over a realistic row count to verify that the shared
  path does not allocate one full-segment validity copy per leaf. Distinguish
  final batch output storage from retained full-segment caches.
- Run focused native index/expression tests and the parser/rewriter suite using
  the repository's required test flags. Record the exact source revision and
  any build or integration limitations.
- Before production rollout, compare original and rewritten queries on indexed
  and scan paths, and measure native peak memory, latency, and throughput under
  matched segment/request concurrency. Unit and microbenchmark results alone
  are not production memory or latency measurements.

## Rejected Alternatives

- **Drop each leaf cache after every batch:** repeats full-segment index work
  and can substantially increase CPU and latency.
- **Only increase batch size:** does not by itself transfer ownership or
  release the leaf caches.
- **Return a writable alias from `IsNotNull()`:** permits existing callers to
  corrupt shared index validity and other requests.
- **Merge arbitrary JSON contains expressions:** typed JSON and path validity
  require additional semantics beyond homogeneous ARRAY predicates.
- **Copy a customer's expression into tests or design documents:** synthetic
  shapes and value distributions are sufficient for correctness and scaling
  tests; production literals are unnecessary.

## References

- [ARRAY contains compilation merge](https://github.com/milvus-io/milvus/pull/52138)
- [Whole-segment expression execution](https://github.com/milvus-io/milvus/pull/48049)
- [Index reader interface restructuring](https://github.com/milvus-io/milvus/pull/53585)
