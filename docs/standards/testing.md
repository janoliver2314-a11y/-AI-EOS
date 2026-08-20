# Standard: Testing

## Purpose

Define what must be tested, how tests should be structured, and what
"adequately tested" means for a change to be considered Done.

## Scope

Applies to all behavior changes in `src/`. Documentation-only or pure
configuration changes are exempt unless they affect runtime behavior.

## Goals

- Ensure every behavior change is protected against regression.
- Keep tests fast, deterministic, and meaningful rather than a coverage
  metric to satisfy.
- Make test intent obvious from its name and structure alone.

## Test Types

| Type | Purpose | Location |
|---|---|---|
| Unit | A single function/module in isolation | `tests/unit/`, mirroring `src/` |
| Integration | Multiple components working together | `tests/integration/` |
| End-to-end | Full user-facing flow | `tests/e2e/` |
| Regression | Reproduces a specific fixed bug | Co-located with the relevant unit/integration test, referencing the `memory/lessons-learned.md` entry |

## Rules

1. **New behavior ships with a test that fails without the change** and
   passes with it — this is the definition of "tested," not merely "a test
   exists somewhere."
2. **Bug fixes ship with a regression test** that reproduces the original
   failure, per `CLAUDE.md` §9.
3. **Test names describe behavior**, not implementation: `rejects a request
   with an invalid signature`, not `test1` or `testHandler`.
4. **No skipped or pending tests without a linked tracking issue** and an
   inline comment explaining why.
5. **Tests are deterministic** — no reliance on real wall-clock time,
   network calls to live services, or unseeded randomness in unit tests.
6. **Flaky tests are fixed or removed**, never silently ignored/retried
   into passing.
7. **An encoder is verified against its consumer's grammar, never against
   your own decoder.** A round-trip (`decode(encode(x)) === x`) pairs an
   encoder with a decoder that is lenient about precisely the characters the
   encoder got wrong, so it passes on output the real consumer will truncate
   or reject. Assert the literal encoded value against the spec's grammar,
   add a negative assertion for the delimiter or metacharacter that must
   never survive raw, and assert that characters the spec *does* permit are
   left alone so the fix does not over-escape. A prefix or `includes()`
   check on encoder output is not coverage. Applies to MIME headers, URLs,
   CSV, shell quoting, SQL identifiers, and escaping into HTML or JSON
   (`memory/lessons-learned.md` LL-0042).
8. **An inconclusive verification result is a finding, not a gap** —
   attributing it to the tooling is a hypothesis that needs evidence like
   any other. Where an observation fits both a harness artifact and a
   product defect, hold the procedure fixed and vary the implementation
   (A/B two configurations): differing outcomes convict the product. When a
   suite provably *cannot* observe a behavior, state the proxies used and
   what remains unproven rather than letting a green run imply coverage.
9. **When the code under test is randomized, force the outcome rather than
   trusting a red run.** Rule 5 bans unseeded randomness *in* a test, but
   says nothing about testing a component that is random by design
   (sampling, shuffling, bucketing, eviction, retry). There, both TDD's
   failing step and a mutation check return a *probabilistic* verdict: a
   real gate can pass against the broken implementation, and the miss reads
   exactly like flake. Before trusting a single red run, compute the odds
   your assertion passes against the broken code. If they are not
   negligible, shrink the population until the correct behavior has no
   freedom — prefer set equality on a small pool over disjointness on a
   large one — then run the red state several times, not once, and record
   the residual odds in a comment so the next reader can judge the gate.
   Never buy the pass with a retry (rule 6). (`memory/lessons-learned.md`
   LL-0077.)

10. **A skipped test must not be reportable as a passing one.** A suite that
    cannot run a scenario — an unmet precondition, an environment without the
    dependency, a version pair that does not exercise the path — must say so
    in a way a human *and* a wrapper can both see: a distinct non-zero exit
    code, and a marker on the same stream as the pass/fail counts. Two
    failures of this shape were found in one project: a destructive drill
    whose skip path printed a banner to stderr and then exited 0 with
    "12 run, 0 failed" on stdout, and the same drill's sibling, which had
    silently self-skipped its most important scenario on *every* run for
    months because the condition it needed never held. A failure must always
    outrank a skip, including when an allow-skip override is set. Conversely,
    do not solve a spurious red by converting a genuine failure into a skip
    unless you can gate the skip on evidence independent of the failing
    signal — where "could not test" and "tested and broke" look identical,
    defaulting to skip converts a real defect into a wave-through.
    (`memory/lessons-learned.md` LL-0087, LL-0089.)

11. **A test must not reset the state whose carry-over it is testing.** Where
    one operation restores what another sets — arm/disarm, open/close,
    set/clear, acquire/release, begin/rollback, cache set/invalidate — at
    least one case must invoke the setter **twice with nothing in between**.
    With the restorer in the middle, it is the restorer and not the code
    under test that makes the assertion pass, and the mutant implementing the
    exact fault ("inherit the previous value") survives while the suite stays
    green. This extends rule 1 to the arrangement: a mutation check verifies
    the assertion, but only the arrangement decides whether the fault was ever
    observable. So when a mutant survives, suspect the setup before the
    assertion. The same masking is supplied for free by `beforeEach`, fresh
    fixtures, and auto-reset mocks — to every test in the file, silently.
    (`memory/lessons-learned.md` LL-0095.)

12. **Vary the field the behavior depends on, and read human-facing output
    before it reaches humans.** Where every test of a unit holds one input
    constant, that input is untested however many cases there are — so when
    production emits several shapes of it (session lengths of 10, 20 and 85;
    names of 3 and 60 characters; amounts of 1 and 10,000), at least one case
    uses the extreme shape rather than the modal one. This bites hardest in
    generated user-facing copy, where an assertion that a value *appears* in
    the output is satisfied by every value it could ever take, so no
    assertion can fail as the number grows into nonsense. Correctness and
    tone are different properties: substring assertions verify the first and
    cannot see the second. Before any first real send, **render the actual
    output for the actual recipients and read it** — a step that costs
    seconds, belongs immediately before the send rather than in CI, and is
    the only check positioned to catch it. (`memory/lessons-learned.md`
    LL-0104.)

13. **A dry run must not report what would happen unless it exercised the path
    that makes it happen.** A preview that shares the read and render paths with
    the real operation but stops before the side-effecting call has verified the
    inputs it computed and nothing else — most importantly, not the credentials,
    endpoint, or permissions the real call consumes, which typically live outside
    the repo. Either validate every input the real call needs (in both modes, so
    a green preview means executable and not merely renderable), or provide a way
    to perform exactly one real operation against a safe target. For a batch that
    touches people, do both, and run the single real operation first every time.
    See `memory/lessons-learned.md#LL-0106`.

## Design Decisions

- **Coverage percentage is not the goal; behavior coverage is.** A high
  percentage with untested edge cases is worse than a lower percentage
  where every documented behavior (including failure modes) is verified.
- **Regression tests explicitly reference their lesson entry** so the link
  between "bug that happened" and "test that prevents recurrence" stays
  visible.

## Best Practices

- Write the failing test before the fix when practical (see
  `docs/playbooks/` for a bug-fix procedure) — this proves the test would
  have caught the bug.
- Test the documented error contract (`docs/standards/error-handling.md`),
  not just the happy path.
- Keep test setup minimal and explicit; avoid shared mutable fixtures that
  make tests order-dependent.

## Examples

```text
describe("webhook signature verification", () => {
  it("rejects a payload with an invalid signature", () => { ... });
  it("rejects a payload with a missing signature header", () => { ... });
  it("accepts a payload with a valid signature", () => { ... });
});
```

## Common Mistakes

- Writing a test that only exercises the happy path, leaving documented
  error conditions unverified.
- Naming tests after the function under test rather than the behavior
  being verified, making failures hard to interpret at a glance.
- Adding `skip`/`todo` tests that are never revisited.
- Asserting on incidental implementation detail (internal call counts,
  private state) instead of observable behavior.
- Verifying an encoder by round-tripping it through your own decoder, or by
  asserting a prefix of its output — both stay green on a value the real
  consumer will reject, because your decoder and your encoder share the same
  blind spot.
- Recording an inconclusive observation as an environment limitation
  without testing that attribution — a harness artifact and a real defect
  look identical until you deliberately discriminate them, and a working
  alternative path makes it easy to stop looking.

## Future Improvements

- Choose a test runner/framework once a language is selected for `src/`
  and document project-specific conventions (mocking approach, fixture
  patterns) here.
- Add CI enforcement (`.github/workflows/`) once a test suite exists, per
  `docs/volumes/05-workflow-engine.md`.

## Related Documents

- `CLAUDE.md` §9, §17
- `docs/standards/error-handling.md`
- `memory/lessons-learned.md`
