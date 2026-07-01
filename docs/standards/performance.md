# Standard: Performance

## Purpose

Define when and how performance work should be done, so it is
evidence-driven rather than speculative.

## Scope

Applies to any change motivated primarily by performance, and to
performance-sensitive code paths once identified.

## Goals

- Ensure performance work is justified by measurement, not intuition.
- Prevent premature optimization from adding complexity without proven
  benefit.
- Keep performance-sensitive paths protected against regression.

## Rules

1. **Correctness and clarity come first.** Do not trade either away for
   performance without a measured, real need (`CLAUDE.md` §8).
2. **Every performance-motivated change cites a benchmark or profiling
   result** in its commit/PR description — before-and-after numbers, not
   just "this should be faster."
3. **Performance-sensitive paths get a regression test or benchmark**
   checked into `tests/`, so future changes can't silently regress them.
4. **Optimize the measured bottleneck**, not the code that merely looks
   slow.

## Design Decisions

- **No speculative performance work.** Optimization without a measured
  bottleneck is explicitly out of scope per `CLAUDE.md` §4 (YAGNI) and §8.
- **Benchmarks live with the code they measure**, in `tests/`, so they run
  as part of normal CI once CI exists, rather than as one-off scripts that
  rot.

## Best Practices

- Profile before optimizing; guesses about where time is spent are wrong
  often enough to be unreliable.
- Prefer algorithmic improvements over micro-optimizations; a better data
  structure usually beats a faster inner loop.
- Document the expected load/scale a performance-sensitive path is
  designed for, so future changes know what "still acceptable" means.

## Examples

A slow API endpoint: profile it first (identify whether it's a database
query, serialization, or network call), fix the actual bottleneck, and
include before/after latency numbers in the PR description along with a
benchmark test guarding the fix.

## Common Mistakes

- Optimizing code based on assumption rather than a profiler/benchmark.
- Adding caching, memoization, or other complexity for a path that isn't
  actually a bottleneck.
- Sacrificing readability for a performance gain that hasn't been measured
  as meaningful at the system's actual scale.
- Fixing a performance issue without a regression test, allowing a later
  change to silently reintroduce it.

## Future Improvements

- Once `src/` exists, choose and document a benchmarking tool appropriate
  to the chosen language/framework.

## Related Documents

- `CLAUDE.md` §8
- `docs/standards/scalability.md`
- `docs/standards/testing.md`
