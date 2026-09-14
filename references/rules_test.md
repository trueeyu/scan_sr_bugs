# Test Code Bug Rules

These rules apply **only to test files** — `*Test.java`, `*Tests.java`, `*_test.cpp`, `*_test.cc`,
and anything under `src/test/`, `fe/fe-core/src/test/`, `be/test/`.

A bug in a test does not crash production, but it is still a bug: a test that cannot fail reports
green coverage for a path nobody is actually checking, so the next regression in that path ships
silently. Scan test files with these rules **in addition to** `rules_common.md` and the
language-specific rules (a test file can of course also leak resources, overflow, etc.).

---

## TEST-001 — Ineffective / Vacuous Test Assertion
**Severity**: MEDIUM

**Pattern**: An assertion is syntactically present but has no power to fail, or the case does not
exercise the thing its comment / method name claims. Four concrete shapes:

1. **Self-comparison** — the same expression is passed as both expected and actual:
   `assertEquals(x.getCount(), x.getCount(), 0.001)`, `EXPECT_EQ(v.size(), v.size())`. Always true
   regardless of what the code under test computes. Usually a copy-paste left behind while filling
   in an "expected" slot the author had not computed yet.
2. **Tolerance/delta wide enough to swallow the value** — a floating-point assertion whose `delta`
   is the same order of magnitude as (or larger than) the expected value:
   `assertEquals(actual, 10, 128)` accepts anything in `[-118, 138]`; `EXPECT_NEAR(a, b, 1e9)`.
   Rule of thumb: flag when `delta >= |expected| / 2`, or when `delta` happens to equal the value
   the test *should* be pinning (a sign the arguments got shifted one slot). Watch JUnit's argument
   order — `assertEquals(expected, actual, delta)`; tests that pass the *actual* first are exactly
   the ones that end up with the real expected value sitting in the `delta` slot.

   **Two distinct origins, worth telling apart — the fix differs:**
   - *An argument-order refactor that was only half applied.* Rewriting `assertEquals(X, 1)` into
     `assertEquals(1, X)` means prepending `1,` **and** deleting the trailing `, 1`; skip the
     deletion and the expected value silently becomes a delta. The tell is decisive: **the same
     commit usually contains sibling lines that were converted correctly**, so `git log -S` on the
     assertion text hands you the intended expected value instead of making you derive it. (Traced
     for `CreateMaterializedViewTest:674`: introduced reversed-but-exact by StarRocks #6844, broken
     by the argument-order pass in #7755, carried through the JUnit 5 migration in #60389 — three
     years unnoticed, because a bulk `Assert.` → `Assertions.` rename never inspects arity.)
   - *A delta inflated step by step to keep a wrong expected value passing.* Here the expected value
     itself is wrong, and each time reality drifted further the author widened the delta instead of
     recomputing. The tell is a **monotone series**: sibling cases sharing one formula whose deltas
     grow (`0.001` → `0.01` → `1`) exactly as the computed value moves away from what is asserted.
     Tightening the delta alone turns the test red — the expected value has to be recomputed too.
     (See `ExpressionStatisticsCalculatorTest` in the scan results below.)
3. **Wrong constant/enum, so the named subject is never exercised** — a case commented
   `// test dayofmonth function` that constructs the operator with `FunctionSet.DAY`, a
   `testXxxNullable` that passes the non-nullable type, a parameterized case whose parameter
   duplicates a sibling's. The assertions may be fine; the *input* is wrong, so the intended branch
   has zero coverage while looking covered. Cross-check every case comment / test-method name
   against the constant actually passed.
4. **Duplicated case** — two blocks in the same test method with identical setup and identical
   assertions (e.g. `FunctionSet.RAND` asserted twice with the same bounds). Costs runtime, and
   hides the fact that a *different* intended case was meant to be there.

**Common unsafe patterns to flag**:
- `assertEquals(a, a, ...)`, `assertTrue(true)`, `assertNotNull(new Foo())`, `EXPECT_EQ(x, x)`.
- `assertEquals(<call>, <literal>, <literal>)` where the third literal is larger than the second —
  almost certainly the expected/actual/delta slots got shifted.
- An assertion whose expected value is produced by re-calling the code under test
  (`assertEquals(calculate(op, stats).getMin(), columnStatistic.getMin())`) — this tests the function
  against itself and passes even when it is uniformly wrong.
- `try { call(); fail("should throw"); } catch (Exception e) { }` where the call before `fail()`
  cannot throw, or where the `catch` is broad enough to swallow the `AssertionError` that `fail()`
  itself raises (`catch (Throwable)` / `catch (Error)` after `fail()` is always vacuous).
- An assertion placed after an unconditional `return`/`break`, or inside a loop over a collection
  that is empty in the fixture — present in the source, never executed.

**NOT a bug** (do not flag):
- A deliberately loose bound documented as such (a timing / heuristic / statistics estimate where the
  test asserts only a range and the delta is clearly narrower than the value being pinned).
- Smoke tests that assert only "does not throw", when that is the stated intent.
- `assertEquals(x.foo(), y.foo())` on two *different* objects — that is a real equivalence check.

**Natural language**: For each assertion, ask *"what change to the production code would make this
fail?"* If the answer is "none", flag it. Concretely: compare the expected and actual expressions
for textual identity; compare any float delta against the magnitude of the expected value; match each
case's comment / method name against the enum, constant, or type it actually passes; then scan the
method for two blocks with identical setup+assertions. Suggested fix: assert the concrete expected
value with a tight delta (compute it once by hand and hardcode it), put `expected` in the first
argument slot, fix the constant so the named function is really exercised, and delete the duplicate
block rather than keeping it as "extra coverage".

**Before you rewrite a swallowing `catch` into `assertThrows`, prove the throw is reachable.**
An empty or vacuous `catch` hides *which* of two facts is true: "the call throws and we ignore it",
or "the call never throws at all". Reading the production method only tells you what happens **if**
control reaches the throwing statement; a unit-test harness often short-circuits before that. Check
the guards the method opens with, and check what the test's base class actually starts — a test that
extends a base which spins up no cluster, no catalog and no session leaves early-exit branches
(`isForwardToLeader()`, `isEmpty()`, `!isInitialized()`) taking the path you did not read. When you
cannot establish it from the code, run the test before rewriting, or assert the weaker fact you do
have evidence for (`assertDoesNotThrow`, the returned value) rather than a throw you assumed.
A wrong `assertThrows` fails loudly, which is recoverable — but it burns a CI round and, worse,
invites "fixing" it back into something vacuous.

Corollary: when a method's **name** promises the exception (`testFooWithException`) and the body
turns out never to throw, the name is itself an instance of shape 3 — rename it to what the test
actually pins, or the next reader re-introduces the same wrong assumption.

**Reference fix — StarRocks PR #78430** (https://github.com/StarRocks/starrocks/pull/78430):
`ExpressionStatisticsCalculatorTest.testUnaryFunctionCall()` contained all four shapes at once — a
self-comparison for `from_unixtime`, a delta of `128` on a value of `128` for `ascii`, a case
commented "test dayofmonth function" that passed `FunctionSet.DAY`, and `rand` covered twice with
identical setup.
```java
// BEFORE (buggy)
// ascii: assertEquals(expected, actual, delta) — args shifted, delta 128 accepts [-118, 138]
Assertions.assertEquals(columnStatistic.getDistinctValuesCount(), 10, 128);
// from_unixtime: compares the value to itself, cannot fail
Assertions.assertEquals(columnStatistic.getDistinctValuesCount(), columnStatistic.getDistinctValuesCount(), 0.001);
// "test dayofmonth function" — but DAY is passed, so dayofmonth is never exercised
callOperator = new CallOperator(FunctionSet.DAY, FloatType.DOUBLE, Lists.newArrayList(columnRefOperator));
// duplicate of the RAND case a few lines above: identical setup and assertions
callOperator = new CallOperator(FunctionSet.RAND, FloatType.DOUBLE, Lists.newArrayList(columnRefOperator));

// AFTER (fixed)
Assertions.assertEquals(128, columnStatistic.getDistinctValuesCount(), 0.001);
Assertions.assertEquals(100, columnStatistic.getDistinctValuesCount(), 0.001);
callOperator = new CallOperator(FunctionSet.DAYOFMONTH, FloatType.DOUBLE, Lists.newArrayList(columnRefOperator));
// duplicate RAND block deleted
```

**Candidate instances found by this rule** (scan of `fe/` test sources @ `d4bbe0d9943`, 2026-09-14).
Line numbers are at that commit. All were fixed in **StarRocks PR #79094**
(https://github.com/StarRocks/starrocks/pull/79094); the entries below are the findings **as first
reported**, and the "Corrections" block at the end of this list records where fixing them proved the
first read wrong. Read both — the corrections are the part with teaching value.

*Shape 1 — assertion compares a constant to itself, so the real value is never checked:*
- `fe/fe-core/src/test/java/com/starrocks/service/InformationSchemaDataSourceTest.java:509-511` —
  `assertEquals("NO", "NO", "isGrantable should be NO")` ×3. The three neighbouring lines assert
  `adminRole.getUser()` / `getHost()`; these three should assert `adminRole.getIs_grantable()`,
  `getIs_default()`, `getIs_mandatory()` (fields 7-9 of `TApplicableRolesInfo` in
  `FrontendService.thrift`). Those three columns of `information_schema.applicable_roles` have no
  coverage at all.
- `fe/fe-core/src/test/java/com/starrocks/common/PropertyAnalyzerTest.java:256` —
  `assertEquals(true, true)` right after
  `enablePeristentIndex = PropertyAnalyzer.analyzeEnablePersistentIndex(property5);`. Every sibling
  case (L243, L251, L261) asserts `enablePeristentIndex`. The "explicit `true` while
  `enable_persistent_index_by_default = false`" case is unverified — exactly the PR #78430 shape.
- `fe/fe-core/src/test/java/com/starrocks/common/lock/YieldableLockTest.java:81` —
  `assertEquals(lock.getHeldTimeNs(), lock.getHeldTimeNs())` under the comment "After close() the
  total is stable". Both calls are in one expression, so a still-ticking counter would not be
  caught; should snapshot the value, wait, then compare. LOW.

*Shape 2 — float `delta` wide enough to swallow the value:*
- `fe/fe-core/src/test/java/com/starrocks/sql/optimizer/rewrite/ScalarOperatorFunctionsTest.java:1136`
  — `assertEquals(1.0, divideDouble(O_DOUBLE_100, O_DOUBLE_100).getDouble(), 1)` accepts `[0, 2]`;
  `divideDouble(100, 100)` returning 0 or 2 passes. Sibling `divideDecimal`/`multiplyLargeInt` use
  exact comparison.
- `.../ScalarOperatorFunctionsTest.java:1040` — `assertEquals(0.0, subtractDouble(100, 100), 1)`
  accepts `[-1, 1]`; `subtractInt`/`subtractBigInt` next to it use the exact 2-arg form.
- `fe/fe-core/src/test/java/com/starrocks/sql/optimizer/rewrite/DefaultPredicateSelectivityEstimatorTest.java:546`
  — `assertEquals(estimate(dtGe2, statistics), 0.002, 0.1)` accepts `[-0.098, 0.102]`, i.e. any
  plausible selectivity. The symmetric `dtGt2` line (L540) uses delta `0.001`; the whole file passes
  *actual* first, which is what lets a stray delta go unnoticed.
- `fe/fe-core/src/test/java/com/starrocks/analysis/CreateMaterializedViewTest.java:674` —
  `assertEquals(1, tableProperty.getReplicationNum().shortValue(), 1)` accepts a replication num of
  0, 1 or 2. Every other assertion in the block uses the 2-arg form; the trailing `1` is a stray.
- `fe/fe-core/src/test/java/com/starrocks/sql/optimizer/statistics/ExpressionStatisticsCalculatorTest.java:619,620,625,630`
  — `hours_diff` / `minutes_diff` / `seconds_diff` min/max asserted against `0` with delta `1`, while
  the adjacent `days_diff` / `datediff` use `0.01` and `mod` uses `0.001`. LOW.

*Related shape — the `catch` block makes the test unfailable* (same "what change would make this
fail? none" test, worth folding into TEST-001 when scanning):
- `fe/fe-core/src/test/java/com/starrocks/sql/parser/ParserTest.java:323-328` and
  `fe/fe-core/src/test/java/com/starrocks/sql/util/TestUtil.java:97-101` — `Assertions.fail();`
  inside `try`, with an **empty** `catch (Throwable)`. `fail()` throws `AssertionFailedError`, which
  *is* a `Throwable`, so the catch swallows the failure: `setLargeDecimalUnderlyingType("foobar")`
  and `add()` after `seal()` can silently succeed and the test still passes. Use
  `assertThrows(...)`, or narrow the catch to the expected exception type.
- `fe/fe-core/src/test/java/com/starrocks/catalog/TableFunctionTableTest.java:167-168` and
  `fe/plugin/spark-dpp/src/test/java/com/starrocks/load/loadv2/dpp/DppUtilsTest.java:141-142` —
  `catch (Exception e) { Assertions.assertFalse(false); }`. Any exception from the code under test
  is swallowed by an assertion that always passes.
- `fe/fe-core/src/test/java/com/starrocks/qe/StmtExecutorNewTest.java:115-120` —
  comment says "This should not throw exception even if planning fails", but the `catch` asserts
  `assertTrue(true)`, so both outcomes pass.
- `fe/fe-core/src/test/java/com/starrocks/connector/iceberg/IcebergApiConverterTest.java:635-640` —
  `catch (DdlException e) { assertTrue(true); }` followed by `assertEquals(sortOrder, null)`. A run
  that never throws also leaves `sortOrder` null, so the test cannot tell "rejected duplicate sort
  column" from "returned null". Use `assertThrows(DdlException.class, ...)`.
- `fe/fe-core/src/test/java/com/starrocks/journal/bdbje/BDBEnvironmentTest.java:247-252` —
  `assertTrue(true)` plus a `catch (JournalException e) { LOG.warn(...) }` around the `setup(true)`
  that the comment says "will get rollback exception"; if it does not throw, the test still passes.

**Corrections that only surfaced while fixing these** (each one is a way the *scan* was wrong, not
the code):
- `StmtExecutorNewTest` — reported as "catch asserts `assertTrue(true)`, so both outcomes pass".
  True, but I then rewrote it to assert the `AnalysisException` that `generateExecPlan()` produces
  for an unknown table, and the test failed: *nothing is thrown*. `StmtExecutorNewTest` extends a
  base class that starts no cluster, so the FE is not the leader, `generateExecPlan()` takes its
  `if (!isForwardToLeader())` early exit, and the planner is never reached. Of the method's two
  contradictory comments ("should not throw" / "exception is expected") the **first** was right.
  This is what the reachability warning above was written from.
- `ExpressionStatisticsCalculatorTest` — reported as LOW, "delta `1` where neighbours use `0.01`".
  Understated. Computing `(left.min - right.max) / interval = -300 / interval` showed the asserted
  expected value of `0` is itself wrong for `hours_diff` (`-0.0833`) *and* for `days_diff` /
  `datediff` (`-0.0035`, delta `0.01` — a pair the scan did not even flag). A delta that looks
  merely loose is worth recomputing: it may be load-bearing.
- `IcebergApiConverterTest` — reported as "cannot tell a thrown `DdlException` from a `null`
  return". Overstated: the following `assertEquals(sortOrder, null)` does catch a non-null return,
  so the gap is only that the exception's type and message go unpinned. Read the lines *after* the
  catch before calling a block vacuous.
- `BDBEnvironmentTest` — reported as "the expected rollback exception is only logged, not asserted".
  Wrong: the method's javadoc states both outcomes are acceptable, and the narrow
  `catch (JournalException)` already fails the test if a raw `RollbackException` escapes. Only the
  stray `assertTrue(true)` was real. **Read the javadoc before calling a permissive catch a bug**;
  some are deliberate, and "fixing" them into `assertThrows` manufactures flakiness.

**False-positive classes confirmed during this scan** (already covered by the NOT-a-bug list, keep
excluding them): reflexivity/`hashCode`-stability checks in `equals` contract tests
(`assertEquals(x, x)`, `assertEquals(x.hashCode(), x.hashCode())`); determinism checks that call the
same producer twice on purpose (`assertArrayEquals(digestOf(sql), digestOf(sql))`,
`ConstantOperatorTest` folding a binary twice); `assertTrue(true)` in tests whose comment states the
intent is "reached here without throwing" (`OAuth2Test`, `TabletSchedulerTest`,
`OdpsCacheUpdateProcessorTest`, `FrontendServiceImplDeadlockTest`); `assertNotNull(new Foo())` written
as deliberate constructor coverage (`UpdatePlanTest:606`); and the StarRocks `assertEquals(1, 2)` /
`assertEquals(1, 1)` fail/pass-marker idiom in try-catch blocks (works correctly — the `(1, 2)` in
the `try` is the real failure trigger — but `fail()` / `assertThrows` reads better).

**Shape 4 (duplicated case) — no confirmed instance in `fe/`.** A repo-scale detector for it is
dominated by intentional near-duplicates: cases whose only difference is one constant inside a
multi-line SQL string or one constructor argument (`MaterializedViewTest` "test int type" vs "test
date type", `ColumnDefTest.testAutoIncrement`, `PublishVersionDaemonTest.testInvalidInitConfiguration`).
Scan for this shape per-file, not repo-wide, and require the *distinguishing* constant of the case —
not just the assertions around it — to be identical.
