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

**Found by this rule** — StarRocks PR #79094 (https://github.com/StarRocks/starrocks/pull/79094):
scanned out with this TEST-001 rule over all ~2000 `fe/` test sources. 15 sites in 13 files: three
self-comparisons, five over-wide float deltas, six tests made unfailable by their `catch` block —
and, found only while fixing them, three assertions whose **expected value was itself wrong**
(`ExpressionStatisticsCalculatorTest`'s `hours_diff` / `days_diff` / `datediff` asserted `0` where
the calculator returns `-0.0833` / `-0.0035`), the inflated delta being what kept them green.

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

---

## TEST-002 — Class-Wide Mock Leaking Into Live Background Threads (Flaky Test)
**Severity**: MEDIUM

**Pattern**: A test installs a mock whose scope is **the whole JVM, not one object** while real
background threads in that JVM also touch the mocked type or object. In StarRocks FE those threads
come from a live cluster (`UtFrameUtils.createMinStarRocksCluster()`, `PseudoCluster`): it starts the
`LeaderDaemon`s — `TabletChecker` (1 s in UT), `TabletScheduler`, `ColocateTableBalancer` (20 s),
`HeartbeatMgr`, `PublishVersionDaemon`, `StatisticsMetaManager`, … — which periodically walk the
catalog and the backend list.

**Which mocks leak across threads** (JMockit 1.49 / Mockito 5, as used by `fe/`; verified against the
JMockit jar during the scan below — get this right before flagging anything):

| Mock | Scope | Leaks into daemons? |
|---|---|---|
| JMockit `@Mocked T` | instance methods of every object whose **exact runtime class** is `T`, plus **all static methods and constructors** of `T`, in every thread | **yes** — but `@Mocked Table` does *not* touch real `OlapTable`s (subclass instances run real code) |
| JMockit `@Capturing T` | same, extended to every subclass / implementation | **yes**, widest |
| JMockit `new Expectations(realObj) {{...}}` (partial mock) | that one object — but a live singleton / catalog object *is* shared with daemons | **yes, if `realObj` is reachable by a daemon** (`getNodeMgr()`, a real `Replica`, `getMetadataMgr()`) |
| JMockit `new MockUp<T>()` | every instance, every thread, until the test ends | yes, but only mode 3 below (no recording) |
| JMockit `@Injectable T` | only that one instance | no |
| Mockito `mock()` / `spy()` | only that one instance | no |
| Mockito `mockStatic` / `mockConstruction` | **thread-local** — only the thread that created it | **no** |

The test is green most runs and red when a daemon tick lands inside the mock's scope. Three failure
modes, all timing-dependent:

1. **Background call recorded as an expectation.** JMockit treats every call on a mocked type made
   *while the `new Expectations() {{ ... }}` block is executing* as part of the recording, whatever
   thread makes it. A daemon calling `db.isSystemDatabase()` during recording silently adds an
   expectation with the default `minTimes = 1`; the test itself never makes that call, so
   verification fails with `Missing 1 invocation to: ...Database#isSystemDatabase()`, stack in the
   daemon (`TabletChecker.checkOneDatabase`, `LeaderDaemon.loop`). A second signature of the same
   race: `ConcurrentModificationException at mockit.internal.expectations.state.ExecutingTest
   .addInjectableMock` thrown from the Expectations constructor — two threads touching the mock
   registry at once. It is not "a JMockit bug"; it is this rule.
2. **Background calls break strict counts.** `times = n` / `maxTimes` / `Verifications` /
   `FullVerifications` count calls from all threads → `Unexpected invocation`.
3. **The daemon receives mock results.** `0` / `null` / `false` / cascaded mocks reach the daemon.
   This only fails the test if an assertion depends on state the daemon then changes — an exception
   *inside* a daemon thread is logged and swallowed by `LeaderDaemon`, it does not fail JUnit. Treat
   mode 3 alone as LOW unless you can name the assertion it corrupts.

**Establishing "a daemon is live during this test" — the part scans get wrong**:
- `UtFrameUtils.setUpForPersistTest()` / `PseudoJournalReplayer` / `GlobalStateMgrTestUtil
  .createTestState()` start **no** daemons. A regex that treats them as "starts a cluster" is wrong.
- `fe-core` runs surefire with `reuseForks=false`: one JVM per test class, so only the class and its
  base classes matter. But inside that JVM `createMinStarRocksCluster()` is a one-shot singleton that
  stays up — a cluster started in *one test method* is live for every method JUnit runs after it
  (default order: by method-name hash, not source order). A class with no cluster in its setup can
  still be exposed.
- Daemons are often disabled or starved in a given base class: `TabletChecker`, `TabletScheduler`,
  `ColocateTableBalancer` run only in **shared-nothing**; `PlanTestNoneDBBase` sets
  `tablet_sched_max_scheduling_tablets = -1`, which makes `TabletChecker` return early every tick;
  `StatisticAutoCollector`, the history syncers and `TemporaryTableCleaner` return early under
  `FeConstants.runningUnitTest`; `StatisticsMetaManager` first acts ~60 s after FE start;
  `ConnectorTableMetadataProcessor` ticks every 10 min. Name the daemon, its interval, and the
  call path that reaches the mocked type — without that, it is not a finding.

**Common unsafe patterns to flag**:
- A live cluster (as established above) **plus** `@Mocked` / `@Capturing` on a type a *running*
  daemon calls: `Database`, `OlapTable`, `LocalTablet`, `Replica`, `Backend`, `SystemInfoService`,
  `NodeMgr`, `GlobalStateMgr` (its static `getCurrentState()` becomes a cascaded mock for everyone),
  `ColocateTableIndex` — **with an `Expectations` / `Verifications` block** recording on that type.
- `new Expectations(GlobalStateMgr.getCurrentState().getNodeMgr())` or any partial mock of a live
  singleton or real catalog object that a daemon also calls.
- Workarounds that betray the race — each one is a finding, not a fix:
  `Thread.sleep(...)` **inside** an Expectations block (it widens the recording window);
  dummy calls after the assertions "to use up" leaked expectations
  (`getClusterInfo().getIdToBackend();` with no assignment); a comment blaming JMockit for a
  `ConcurrentModificationException`; stopping daemons one by one in `@BeforeAll`.
- CI signature: JMockit `Missing invocation` / `Unexpected invocation` whose `Caused by` stack is in
  a daemon thread.

**NOT a bug** (do not flag):
- Mockito `mockStatic` / `mockConstruction` — thread-local, cannot reach a daemon thread.
- `@Mocked` on a class whose catalog instances are all subclasses (`@Mocked Table`), or whose only
  real instances would be created by the test (`@Mocked LakeTable` in shared-nothing).
- A live cluster, a mocked type the daemons *do* touch, but **no Expectations / Verifications on it
  and no assertion a daemon can influence** — mode 3 with nothing to corrupt. Latent only.
- Classes with no daemons (persist-harness tests), and daemons that are disabled / starved as listed
  above, or whose interval dwarfs the class runtime.
- `@Injectable` mocks and Mockito `mock()` / `spy()`.

**Natural language**: For each mock, ask *"does this replace one object, or the whole type?"* (use
the table). If the whole type — or a shared object — ask *"which running thread calls it, how often,
through which path?"* and name the daemon. Then ask *"what records or counts calls on it, or which
assertion reads state the daemon could change?"* Only when all three have concrete answers is the
test flaky. Suggested fix, in order: `@Injectable` / Mockito `mock()` so only the instance passed in
is mocked — **but first check how the code under test obtains the object**: if it goes through
`GlobalStateMgr.getCurrentState().getXxx()` or reads real catalog replicas, `@Injectable` silently
stops reaching it and the test breaks; then pair it with a `MockUp` of the single getter that hands
it out (`MockUp<NodeMgr>.getClusterInfo()`), or set real state instead of mocking (row counts on real
replicas, `new Database(id, name)`). Moving the case to a class with no cluster also works. Do not
"fix" it with `minTimes = 0` on the recorded calls, a `Thread.sleep`, or stopping one named daemon —
the next daemon re-opens the race.

**Reference fix — StarRocks PR #79915** (https://github.com/StarRocks/starrocks/pull/79915):
`ColocateTableIndexTest` starts a full FE in `@BeforeEach`; the three
`testAfterTabletCreationRouting*` cases took `@Mocked Database db, @Mocked OlapTable/LakeTable`. When
`TabletChecker.checkOneDatabase()` called `db.isSystemDatabase()` on a real database while the
`Expectations` block was recording, JMockit recorded it with `minTimes = 1`, and the test failed
intermittently with `Missing 1 invocation to: com.starrocks.catalog.Database#isSystemDatabase()`,
caused by `TabletChecker.checkOneDatabase(TabletChecker.java:317)` ← `LeaderDaemon.loop`. Note every
recorded call in the test already had `minTimes = 0` — it did not help.
```java
// BEFORE (flaky) — @Mocked replaces Database in every thread, including TabletChecker
@Test
public void testAfterTabletCreationRoutingForNonLakeTable(
        @Mocked Database db, @Mocked OlapTable olapTable) throws Exception {
    new Expectations() {{
        db.getId(); result = 100L; minTimes = 0;
        ...
    }};

// AFTER (fixed) — @Injectable mocks only the instances passed to addTableToGroup()
@Test
public void testAfterTabletCreationRoutingForNonLakeTable(
        @Injectable Database db, @Injectable OlapTable olapTable) throws Exception {
```

**Earlier fixes of the same bug class** (found in `git log` during the scan — the pattern recurs, and
each was fixed one file at a time): #64772 removed `@Mocked Database` from
`AlterTableOperationStmtTest` (same TabletChecker path as #79915); #8951 and #66178 removed
`@Mocked GlobalStateMgr` from `RefreshTableStmtTest` / `IcebergHiveCatalogTest`; #69036 replaced
partial-mock `new Expectations(getNodeMgr())` in `IcebergMetadataTest` with `MockUp`s; #70316
replaced `@Mocked GlobalStateMgr` + `times =` in `StmtExecutorTest` with a `MockUp` guarded by
`Thread.currentThread() != testThread`; #33345 and a series of later commits in
`ColocateTableBalancerTest` (below).

**Scan results** (`fe/` test sources @ `9a5c851e68b`, 2026-09-29). A regex pass found 141 live-cluster
classes with a class-wide mock on a catalog/system type (601 sites, ~500 of them `MockUp`). The
`@Mocked` / `@Capturing` / `mockStatic` subset — 33 files, ~95 sites — and 40 partial-mock
`new Expectations(sharedObj)` sites on live singletons were triaged by hand.

*PLAUSIBLE — confirmed in the past, mitigated today, not fixed:*
- `fe/fe-core/src/test/java/com/starrocks/clone/ColocateTableBalancerTest.java` — 8 test methods
  (L258, L361, L460, L586, L650, L754, L932, L1006) take `@Mocked SystemInfoService`,
  `@Mocked Backend`, `@Mocked ClusterLoadStatistic` in a class that starts a shared-nothing cluster
  (L136). Every symptom in this rule is in the file's history: a removed comment recording
  `Missing 1 invocation to: SystemInfoService#getIdToBackend() ... at HeartbeatMgr
  .runAfterCatalogReady`; a `ConcurrentModificationException at ExecutingTest.addInjectableMock`
  blamed on JMockit and "fixed" with `Thread.sleep(2000)` **inside** the Expectations block (L364-376,
  #33345); dummy `getIdToBackend()` / `getBackendIds()` calls at L451-453 to use up leaked
  expectations; and `@BeforeAll` stopping HeartbeatMgr, TabletScheduler, TabletCollector, AlterJobMgr
  and a MockUp'd TabletChecker one at a time (#60796, #67416, #67526). No short-interval daemon still
  reaching `SystemInfoService` was found, so today it is LOW — but the next daemon added to the FE
  re-opens it. Fix: `@Injectable`, plus `MockUp<NodeMgr>.getClusterInfo()` returning it for
  `testOverallGroupBalance` / `testPerGroupBalance`, whose code under test fetches
  `getClusterInfo()` itself; then delete the sleep and the dummy calls.
- `fe/fe-core/src/test/java/com/starrocks/sql/plan/PlanFragmentWithCostTest.java:951` —
  `@Mocked Replica` in a plan test. TabletChecker is starved (`tablet_sched_max_scheduling_tablets =
  -1`), but `ColocateTableBalancer` (20 s) still walks the real replicas of PlanTestBase's
  `colocate_t0` … tables via `TabletChecker.getColocateTabletHealthStatus` → `Replica.getBackendId()`.
  LOW: mode 1 needs a 20 s tick to overlap a ~7-call recording. `@Injectable` would **break** it (the
  planner reads real replicas); set row counts on the real replicas or `MockUp<Replica>.getRowCount`.

*NOT a bug after triage — kept as calibration for the next scan:*
- Persist-harness classes misread as live clusters: `CatalogRecycleBinTest`, `RoutineLoadJobTest`,
  `RoutineLoadManagerTest`, `SharedDataStorageVolumeMgrTest`, `LocalMetastoreSimpleOpsEditLogTest`,
  `GlobalTransactionMgrTest`, `ExportHandleTest`.
- `@Mocked MetadataMgr` / `CatalogMgr` in 11 analyzer tests (`AlterTableOperationStmtTest`,
  `UseDbStmtTest`, `HiveTableTest`, …): the only daemon path to them is `StatisticsMetaManager`
  creating `_statistics_` tables once, ~60 s after FE start.
- `@Mocked Table` (`AnalyzeTruncateTableTest`, `StmtExecutorTest:2152`), `@Mocked LakeTable` ×9 in
  `ColocateTableIndexTest` (shared-nothing): exact-class matching, no real instances.
- Live cluster, daemons do reach the mocked type, but no Expectations / Verifications and no
  daemon-affected assertion: `KuduScanNodeTest:140`, `LakeTableHelperTest:127,177`,
  `IcebergMetadataTest:3166,3208,3252`.
- Mockito `mockStatic(GlobalStateMgr.class)` in `VacuumTest`, `LeaderOpExecutor*Test`,
  `FrontendServiceImplTest`, `ExecPlanAIProviderTest`, `GlobalStateMgrTest:837` — thread-local.
- Partial mocks of shared objects: `LimitTest:929` (`new Expectations(realReplica)` — TabletChecker
  starved, `t0` not colocated); `ReportHandlerTest:284` (`times = 2` on the live
  `ResourceUsageMonitor`, but its only other caller is `ReportHandler.handleReport` and the UT
  `MockedBackend` sends no reports); the `StatisticStorage` / `IDictManager` partial mocks in plan
  tests (no running daemon calls them).
- `GlobalStateMgrTest:197` — no cluster *yet*: the only `createMinStarRocksCluster()` is in a method
  JUnit orders 29th, this one runs 4th. Fragile to renames, not flaky today.

*Not triaged:* the ~500 JVM-wide `MockUp<T>` sites (mode 3 only, so a finding needs a named
corrupted assertion), and BE C++ tests (gmock has no class-wide mocking).

*Adjacent issues seen, outside this rule:* `CatalogLevelTest:65,119` installs a `@Mocked MetadataMgr`
globally with `setMetadataMgr()` and never restores it; `ColocateTableIndexTest` ~L285 installs a
JVM-wide `MockUp<OlapTable>` (`isCloudNative = true`) that makes TabletChecker skip every real table
while active.
