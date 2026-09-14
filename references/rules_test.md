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
