# MEP: Prefilter-aware FMIndex regex routing

- **Created:** 2026-10-13
- **Author(s):** @xiaofan-luan
- **Status:** Draft
- **Component:** QueryNode, Index
- **Related Issues:** [milvus#54191](https://github.com/milvus-io/milvus/issues/54191)

## Summary

Choose between FMIndex candidate generation and the ordinary masked regex scan
using the selected literal occurrence counts and the current upstream bitmap.
Prepare index-local statistics once per physical expression, defer Locate until
the bitmap is available, and reuse the resulting candidate bitmap. A routing
estimate affects the execution strategy only; canonical RE2 evaluation and exact
bitmap intersection continue to determine results.

## Motivation

[milvus#53960](https://github.com/milvus-io/milvus/pull/53960) obtains necessary
literal conditions through the public RE2 API and guards FMIndex lookup using
occurrence count and segment text size. Its initial decision does not include
the upstream filter. Scanning a small surviving row set can be cheaper than
locating occurrences throughout the segment, even when the regex literal is
selective across the complete segment. Rejected lookups must retain the existing
scan batch size and avoid repeated preparation work.

## Design Details

### Immutable preparation

The executor retains the eligibility checks for sealed scalar string fields
with available raw data. For an eligible regex, it postpones the final route
until evaluation receives the current batch's input bitmap.

Preparation uses the expression's compiled RE2 matcher and its cached public
necessary-literal analysis. It counts each distinct literal once and preserves
the existing selection policy: choose the least-cost child of an AND and retain
all children of an OR. The prepared plan belongs to one expression and one
immutable segment index. It cannot share occurrence counts across segments.

An unusable condition means scan. A finite zero count of the selected necessary
condition proves no matches; it does not require Locate. TRUE, an unsupported
condition, or a preparation failure must never be interpreted as zero matches.
If a condition is unusable, or even the full-active window cannot amortize
preparation, the executor can establish the scan route before Count and retain
eager raw-data prefetch. Useful conditions still defer Count until evaluation.

### Cost estimate before Locate

For total index rows `N`, selected occurrence cost `C`, remaining physical rows
`U`, batch size `M`, and active input bits `B`, use:

```text
p = B / M
estimated candidate rows = min(C, N)
estimated active rows remaining = U * p
estimated candidates after prefilter = min(C, N) * (U / N) * p
```

Count measures occurrences, not distinct rows. Clamping the count is a cheap
initial estimate, and the intersection estimate assumes independence. Repeated
text and correlated predicates can make it inaccurate. These assumptions may
change routing performance but cannot remove a candidate or alter correctness.

The FM estimate includes full-segment Locate work, suffix-array sampling rate,
full-segment bitmap construction and Boolean operations, batch bitmap slicing,
and the remaining candidate reads and RE2 checks. Only reads and checks are
scaled by the upstream density; Locate and full-segment bitmap work are not.

The initial soft comparison prices Locate as
`0.25 * C * max(1, SA) / ratio`, independently of the unscaled inherited
ceiling below. The coefficient is a coarse relative weight chosen after
paired warm native diagnostics showed that unit Locate pricing rejected
useful medium-frequency and OR lookups. Those mixed query timings are not
a pure Locate profile. Bitmap work and candidate rechecks retain separate
terms, and broader corpora must validate this initial calibration.

The existing full-segment Locate ceiling, `C * SA < ratio * text_bytes`, remains
an additional upper bound. The masked cost comparison can reject further plans
within that bound. This keeps the initial scan weights from broadening the old
dense acceptance range, especially for short rows. A finite zero-count proof
remains exempt because it performs no Locate.

The scan estimate includes both active string bytes and fixed batch work. The
current sequential string scan constructs string views for each visited batch
before skipping inactive rows. Upstream AND evaluation can skip entirely empty
batches, so the initial estimator accounts for that possibility separately.
Coefficients are relative routing weights and should be calibrated using paired
end-to-end measurements, rather than presented as accurate latency forecasts.

A cheap preparation floor rejects scan windows too small to amortize literal
analysis and Count. An absent bitmap means all rows are active; an explicit
all-false bitmap means zero active rows. Empty indexes and exhausted windows are
handled without division by zero.

### Batch transitions and fallback

A cost rejection for a sparse batch scans only that batch and leaves the plan
available for a later, denser batch. Once accepted, Locate creates one exact
candidate bitmap for the segment. Later batches reuse it, intersect the current
input bitmap, and apply the original RE2 matcher to surviving raw values.

There is also a safe permanent rejection: if full-segment Locate and bitmap
construction alone exceed scanning the remaining window with every row active,
no later, smaller window can amortize those fixed costs. The expression then
uses the ordinary scan directly. This avoids repeated bitmap counts and cost
calculations for common literals.

Deferred FM regex expressions keep normal batch execution rather than claiming
they can execute all at once. Scan fallback advances the data cursor once and
synchronizes the index cursor; index execution and parent short-circuiting keep
both cursors aligned. NULL, negation, offsets, chunk boundaries, and short final
batches retain their existing semantics.

When expression caching is enabled, the deferred regex evaluator explicitly
allows lookup of an exact cached result. This prevents initially index-eligible
expressions from losing existing scan cache hits after fallback. Cache admission
still requires complete, unmasked, sequential canonical results. A phase-one
candidate bitmap is never stored as the final predicate result.

## Compatibility, Deprecation, and Migration Plan

No query syntax, index format, RE2 package, or index build changes are required.
The implementation uses the current pattern-reader contract and the FM
reader's existing construction-time cost ratio. The current loader uses its
default ratio of 0.001; connecting that loader policy to runtime configuration
is a separate existing concern. Other scalar
operators and index types retain their routing. Cold-storage prefetch overlap
and concurrent throughput need separate measurement; warm-data routing results
do not establish those properties.

## Test Plan

Unit tests cover Count reuse, complete OR coverage, cheapest-AND selection,
zero-count proofs, invalid cost inputs, preparation deferral, permanent rejection,
and sparse-to-dense reconsideration. Executor tests compare exact result and
validity bitmaps against ordinary scans across 0%, 1%, 10%, and 100% masks,
changing densities, multiple chunks, a short final batch, NULL, negation,
offsets, and parent AND short-circuiting. Test-only index spies verify that
rejected plans do not Locate and accepted plans execute at most once. Cache
tests exercise capture and cache hits on the deferred path.

Paired benchmarks compare the exact parent commit and the candidate using the
same compiler flags, corpus, queries, index settings, and host. They include
ordinary masked scan, previous FM routing, and the new routing. Report every
group, p50 and p95, fresh-expression preparation, correctness parity, observed
routes, and run-to-run noise. The initial warm fallback targets are at most 3%
p50 and 5% p95 regression; results outside those targets require investigation.
Each fresh-expression sample includes canonical RE2 construction and its
first necessary-literal analysis; warming data/index does not share the
matcher between expressions. Report cold I/O separately. Use rebar and a
larger line-oriented grep corpus rather than relying only on synthetic
literal lookup timings.

Include a real parent AND where an indexed integer prefix matches only the
first one or few physical batches. This verifies skipped batches rather
than manually feeding empty masks into the regex leaf. With only a local
batch bitmap available, extrapolating its density can overestimate future
scan work for clustered matches; report such route misestimates explicitly.

## References

- [Milvus enhancement issue #54191](https://github.com/milvus-io/milvus/issues/54191)
- [FMIndex regex acceleration PR #53960](https://github.com/milvus-io/milvus/pull/53960)
- [Regex filtering design](20260409-regex_filter.md)
