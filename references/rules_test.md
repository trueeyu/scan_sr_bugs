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
     for a `CreateMaterializedViewTest` replication-num assertion: introduced reversed-but-exact by StarRocks #6844, broken
     by the argument-order pass in #7755, carried through the JUnit 5 migration in #60389 — three
     years unnoticed, because a bulk `Assert.` → `Assertions.` rename never inspects arity.)
   - *A delta inflated step by step to keep a wrong expected value passing.* Here the expected value
     itself is wrong, and each time reality drifted further the author widened the delta instead of
     recomputing. The tell is a **monotone series**: sibling cases sharing one formula whose deltas
     grow (`0.001` → `0.01` → `1`) exactly as the computed value moves away from what is asserted.
     Tightening the delta alone turns the test red — the expected value has to be recomputed too.
     (Fixed in `ExpressionStatisticsCalculatorTest` by PR #79094: `hours_diff` / `days_diff` /
     `datediff` asserted `0` where the calculator returns `-0.0833` / `-0.0035`.)
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
- A `catch` whose body is an always-true assertion — `catch (Exception e) { assertFalse(false); }`,
  `catch (...) { assertTrue(true); }` — so both "threw" and "did not throw" pass.
- A comparison of two calls in one expression meant to check stability over time
  (`assertEquals(lock.getHeldTimeNs(), lock.getHeldTimeNs())` under "after close() the total is
  stable") — snapshot, wait, then compare.
- An assertion placed after an unconditional `return`/`break`, or inside a loop over a collection
  that is empty in the fixture — present in the source, never executed.

**NOT a bug** (do not flag):
- A deliberately loose bound documented as such (a timing / heuristic / statistics estimate where the
  test asserts only a range and the delta is clearly narrower than the value being pinned).
- Smoke tests that assert only "does not throw", when that is the stated intent.
- `assertEquals(x.foo(), y.foo())` on two *different* objects — that is a real equivalence check.
- Reflexivity / `hashCode`-stability checks in `equals` contract tests (`assertEquals(x, x)`,
  `assertEquals(x.hashCode(), x.hashCode())`), and determinism checks that call the same producer
  twice on purpose (`assertArrayEquals(digestOf(sql), digestOf(sql))`, folding a constant twice).
- `assertTrue(true)` where the comment states the intent is "reached here without throwing", and
  `assertNotNull(new Foo())` written as deliberate constructor coverage.
- The StarRocks `assertEquals(1, 2)` / `assertEquals(1, 1)` fail/pass-marker idiom inside
  try/catch — it works (the `(1, 2)` in the `try` is the real failure trigger); `fail()` /
  `assertThrows` merely reads better.
- A permissive `catch` whose javadoc says both outcomes are acceptable, or whose narrow exception
  type already fails the test on the wrong exception — "fixing" it into `assertThrows` manufactures
  flakiness.

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
a scan of all ~2000 `fe/` test sources fixed 15 sites in 13 files — three self-comparisons, five
over-wide float deltas, six tests made unfailable by their `catch` block — plus three assertions whose
**expected value was itself wrong**, kept green only by an inflated delta.

**Lessons from fixing the findings** (each is a way a first read of a finding was wrong):
- **A delta that looks merely loose may be load-bearing.** Recompute the expected value from the
  formula before tightening it; if tightening turns the test red, the expected value is wrong too.
  Check sibling cases that share the formula even if they were not flagged.
- **Read the lines after the `catch` before calling it vacuous.** A following assertion on the
  result (`assertNull(result)`) may already catch the non-throwing path; the real gap may only be
  that the exception's type and message go unpinned.
- **Read the javadoc before calling a permissive `catch` a bug** — some tests deliberately accept
  both outcomes; only the stray `assertTrue(true)` is the defect.
- **Prove the throw is reachable** before rewriting into `assertThrows` (see the warning above) —
  a test on a base class with no cluster took an early-exit branch and never threw.

**Scanning shape 4 (duplicated case):** do it per file, not repo-wide. A repo-scale detector is
swamped by intentional near-duplicates whose only difference is one constant inside a multi-line SQL
string or one constructor argument; require the *distinguishing* constant of the case — not just the
assertions around it — to be identical.

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
JMockit jar — get this right before flagging anything):

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
- A running daemon is not enough — **it must have work in this fixture that reaches the method**.
  Most daemons iterate a job list, a transaction list or a queue: hand-built jobs (`new XxxJob(...)`,
  factory-created) that were never registered with their manager are invisible to it;
  `PublishVersionDaemon` idles without committed transactions (no loads → no work); the MV scheduler
  idles when every MV is `REFRESH DEFERRED MANUAL`. Conversely, some daemons run on a very short tick
  and execute the same code the test drives by hand — e.g. in shared-data the `TabletReshardJobMgr`
  leader daemon ticks every **10 ms** and calls `colocateChecker.runOneCycle()` first.
- Match the **exact overload** the mock replaces. A daemon calling a sibling method does not count:
  `TxnTimeoutChecker` aborts via `DatabaseTransactionMgr.abortTransaction`, never the
  `GlobalTransactionMgr.abortTransaction(long, long, String)` tests usually mock; background stats
  queries use the 3-argument `StatementPlanner.plan`, not the 2-argument one.

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
- A `MockUp` (any mock scoped to the whole JVM) whose `@Mock` body has a **side effect the test
  asserts on** — `count++`, `called = true`, `captured.add(arg)`, `lastArg = arg` — while a running
  daemon also calls that method. This is mode 2 without JMockit's counting: the daemon's call mutates
  the same variable. Two outcomes, both findings:
  - *flaky-fail* — `assertEquals(1, count)`, `assertFalse(called)`, `assertEquals(expected, captured)`,
    an asserted argument the daemon overwrote;
  - *masked-pass* (LOW) — `assertTrue(called)`, `count >= 1`, `assertFalse(list.isEmpty())` are
    *made true* by the daemon, so the test passes even if the code under test never makes the call:
    the TEST-001 "cannot fail" problem, produced by concurrency. A comment such as "the background
    scheduler may also tick, so the exact count is not asserted" is this shape admitted in writing —
    the assertion was weakened instead of the daemon being stopped.
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
replicas, `new Database(id, name)`). Moving the case to a class with no cluster also works. For a
side-effect `MockUp`, guard the side effect with `if (Thread.currentThread() == testThread)` (#70316)
or filter on the argument the test cares about; when a whole class races one short-tick daemon, stop
that daemon in `@BeforeAll` and wait for it to go quiet (`setStop()` + `awaitQuiesced`, #78742) and
then restore exact assertions. Do not "fix" it with `minTimes = 0` on the recorded calls, a
`Thread.sleep`, or by weakening the assertion — the next daemon re-opens the race.

**`@Mocked` → `@Injectable` is safe only when the failure would be loud.** `@Injectable` stops
mocking the other instances, the static methods and the constructors of the type. If the code under
test reached the mock through one of those, the swap either fails loudly (an `Expectations` with the
default `minTimes = 1` reports `Missing invocation` — fine, you will see it) or passes silently on
the real code path (no `Expectations`, or every recorded call `minTimes = 0`, which is common in
StarRocks) — and the test no longer covers what it claims. After each swap, prove the test can still
fail: break the recorded `result` or the branch under test and watch it go red. New tests should
default to `@Injectable` and use `@Mocked` only with a comment naming what it must intercept.

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

**Earlier fixes of the same bug class** (from `git log` — the pattern recurs, and
each was fixed one file at a time): #64772 removed `@Mocked Database` from
`AlterTableOperationStmtTest` (same TabletChecker path as #79915); #8951 and #66178 removed
`@Mocked GlobalStateMgr` from `RefreshTableStmtTest` / `IcebergHiveCatalogTest`; #69036 replaced
partial-mock `new Expectations(getNodeMgr())` in `IcebergMetadataTest` with `MockUp`s; #70316
replaced `@Mocked GlobalStateMgr` + `times =` in `StmtExecutorTest` with a `MockUp` guarded by
`Thread.currentThread() != testThread`; #33345, #60796, #67416, #67526 stopped `ColocateTableBalancerTest`'s
interfering daemons one at a time; #73662 / #78742 stubbed or quiesced the 10 ms `TabletReshardJobMgr`
daemon for reshard tests.
