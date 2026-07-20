# Common Bug Rules (C++ & Java)

These rules apply to both C++ and Java files.

---

## CMN-001 — Resource Leak on Early Return
**Severity**: MEDIUM  
**Pattern**: A resource (file handle, connection, lock, buffer) is acquired, but there exists a code
path (early return, thrown exception, `break` in a loop) where it is not released.  
**Natural language**: Look for patterns like `open()`/`close()`, `lock()`/`unlock()`,
`acquire()`/`release()`, `new`/`delete` where one side can be skipped.

---

## CMN-002 — Integer Overflow in Arithmetic
**Severity**: HIGH  
**Pattern**: Arithmetic on `int` types where the result could exceed the type's max value, especially
in size calculations like `a * b` where both are user-controlled or could be large.  
**Natural language**: Look for multiplication or addition of int/int32 values used as sizes, offsets,
or array indices without overflow checks or casting to a wider type.

---

## CMN-003 — Unchecked Return Value of Critical Function
**Severity**: MEDIUM  
**Pattern**: Return value of a function that can fail (e.g., `read()`, `write()`, `seek()`,
`send()`, `recv()`, status-returning APIs) is ignored without any error handling.  
**Natural language**: Look for calls to I/O, network, or status-returning functions whose return
value is discarded (result not stored, not checked in `if`, not passed anywhere).

---

## CMN-004 — Off-by-One Error
**Severity**: HIGH  
**Pattern**: Loop bounds use `<=` instead of `<` (or vice versa) when iterating over arrays or
buffers; index calculations use `size` instead of `size - 1`.  
**Natural language**: Look for array/vector accesses inside loops where the upper bound might allow
accessing one element past the end.

---

## CMN-005 — Use of Magic Numbers in Critical Logic
**Severity**: LOW  
**Pattern**: Hardcoded numeric literals (other than 0 and 1) in buffer sizes, timeouts, retry
counts, or protocol fields without a named constant or comment explaining the value.

---

## CMN-006 — Inconsistent Error Handling
**Severity**: MEDIUM  
**Pattern**: A function handles errors (returns error code, throws exception) in some branches but
silently swallows errors in other branches (empty catch, ignored status).  
**Natural language**: Look for `catch` blocks that are empty or only log without re-throwing, and
`if (status != OK)` checks that exist in some places but not others in the same function.

---

## CMN-007 — Dangerous String/Buffer Operation Without Size Check
**Severity**: HIGH  
**Pattern (C++)**: Use of `strcpy`, `sprintf`, `gets`, `strcat` without explicit size validation.  
**Pattern (Java)**: Constructing strings from external/untrusted input fed directly into SQL, shell
commands, or file paths without sanitization.  
**Natural language**: Look for string manipulation that doesn't validate length or content when the
input could be attacker-controlled or unexpectedly large.

---

## CMN-008 — Condition Always True or Always False
**Severity**: MEDIUM  
**Pattern**: A boolean condition that is logically always true or always false due to the range of
the variable, a prior assignment, or tautological logic (e.g., `x >= 0` when `x` is unsigned).  
**Natural language**: Look for redundant checks, unsigned comparisons with 0, or conditions that
contradict each other within the same scope.

---

## CMN-009 — Missing Default Case in Switch
**Severity**: LOW  
**Pattern**: A `switch` statement over an enum or integer that lacks a `default:` case, which could
silently ignore unexpected values.

---

## CMN-010 — Copy-Paste Bug (Duplicate Conditions / Wrong Variable)
**Severity**: HIGH  
**Pattern**: Two consecutive `if` branches with identical conditions, or a chain of conditions where
the same variable is compared in consecutive branches when clearly different variables were intended.  
**Natural language**: Look for `if (a == X) { ... } else if (a == X) { ... }` or similar patterns
where the programmer likely intended different variables or values.

---

## CMN-011 — Dynamically-Mutable Config Read Once and Frozen
**Severity**: MEDIUM  
**Pattern** (StarRocks-specific): A configuration value that is declared **dynamically mutable** is
read exactly once and cached, so runtime changes (`ADMIN SET`, `update information_schema.be_configs`,
`ADMIN SET FRONTEND CONFIG`) never take effect. The value is captured in a `static` local, a
`static final` / long-lived field, a lambda, or is read *outside* a long-running loop and then reused
as that loop's interval / timeout / threshold — a context that is executed many times but only ever
sees the first-read value.

**Step 0 — prove the config is mutable (do this before flagging anything):**
- **C++**: the config must be declared with a `CONF_m*` macro (e.g. `CONF_mInt64`, `CONF_mBool`,
  `CONF_mString`) in `be/src/common/config.h`. Non-`m` macros (`CONF_Int32`, `CONF_Bool`, …) are
  **startup-only** — caching them is *not* a bug.
- **Java (FE)**: the field must be annotated `@ConfField(mutable = true)` in `Config.java`. A bare
  `@ConfField` (default `mutable = false`) is startup-only — caching it is *not* a bug.

**Pattern (C++)** — a `CONF_m*` value frozen in a repeatedly-executed context:
1. `static` / `static const` **local** variable initialized from `config::<mutable_name>`. `static`
   means the initializer runs only on the first call, so later dynamic changes are ignored. This is
   the canonical form (see reference fix below):
   ```cpp
   // BUG: static local captures the mutable config once and never refreshes
   static const int64_t kWaitTimeout = config::datacache_disk_adjust_interval_seconds * 1000 * 1000;
   ```
2. A `config::<mutable_name>` read into a variable **before / outside** a background/daemon `while`
   or `for` loop (functions named `*_callback`, `*_daemon`, `*_thread`, monitor/GC/compaction/report
   loops), then used *inside* the loop as the sleep interval / threshold.
3. A `config::<mutable_name>` captured **by value** into a lambda that lives for the process lifetime.

**Pattern (Java / FE)** — a `mutable = true` value frozen in a daemon or static field:
1. A `Daemon` / `FrontendDaemon` / `LeaderDaemon` subclass passes a `mutable = true` interval config
   to `super(name, Config.<name>)` **once at construction** and never calls `setInterval(...)` to
   refresh it. The base `Daemon.run()` sleeps `getInterval()` each cycle, so the interval stays frozen
   at the constructor value. The correct pattern (e.g. `TabletStatMgr`) re-reads and refreshes at the
   top of each cycle:
   ```java
   // CORRECT — honors runtime changes:
   if (getInterval() != Config.tablet_stat_update_interval_second * 1000L) {
       setInterval(Config.tablet_stat_update_interval_second * 1000L);
   }
   ```
2. A `mutable = true` `Config.<field>` captured into a `static final` (or other class-load-time)
   field and reused for the process lifetime.
3. A `mutable = true` `Config.<field>` read *outside* a `runAfterCatalogReady` / `run()` loop and
   used repeatedly *inside* it as the interval / threshold.

**NOT a bug** (do not flag):
- Any non-mutable config (`CONF_` without `m`, or `@ConfField` without `mutable = true`) cached
  anywhere — those are startup-only by design.
- A config re-read **fresh at the top of each loop iteration** (`config::x` / `Config.x` inside the
  loop body) — this is the correct pattern.
- Meyers singletons / caches that hold **immutable** data, or objects that are **reconstructed per
  operation** (a memtable, delta writer, scan node, compaction policy) — re-reading would not change
  behavior because the object is short-lived anyway.
- Cases where a dynamic-update **callback** re-applies the value (e.g. BE
  `config_update_hooks.cpp` registering a setter, FE `setInterval` / `setFixedThreadPoolSize` /
  `changeMaxSize` refresh paths).

**Natural language**: For each read of a config that you have confirmed is mutable, ask: *"is this
value captured once and then used many times?"* Flag `static`/`static const` locals and
`static final` fields initialized from a mutable config; flag daemon constructors that pass a
`mutable = true` interval to `super(...)` with no `setInterval` refresh anywhere in the class; flag
a mutable config hoisted above a long-running loop and used as its interval/threshold. Always cite
the config's declaration macro/annotation as proof of mutability, and confirm the caching site runs
repeatedly. Suggested fix: read the config fresh each iteration (C++ — drop `static`), or add a
`setInterval(...)` refresh at the top of each cycle (Java daemons).

**Reference fix — StarRocks PR #56410** (https://github.com/StarRocks/starrocks/pull/56410):
`DiskSpaceMonitor::_adjust_datacache_callback()` cached `config::datacache_disk_adjust_interval_seconds`
(a `CONF_mInt64`) in a `static const int64_t kWaitTimeout`, so `update be_configs set
value="1" where name="datacache_disk_adjust_interval_seconds"` had no effect. The fix removed
`static`, recomputing the timeout each loop iteration.

**Candidate instances found by this rule** (FE, unconfirmed/not yet fixed — a mutable-interval daemon
that never calls `setInterval`): `RoutineLoadScheduler` (`routine_load_scheduler_interval_millisecond`),
`ConnectorTableMetadataProcessor` (`background_refresh_metadata_interval_millis`), `PipeScheduler` /
`PipeListener` (`pipe_scheduler_interval_millis` / `pipe_listener_interval_millis`),
`TabletWriteLogHistorySyncer` (`tablet_write_log_history_sync_interval_sec`), `StatisticsMetaManager`
(`statistic_manager_sleep_time_sec`); plus `IcebergCachingFileIO` caching four `mutable = true`
iceberg cache configs in `static final` fields.
