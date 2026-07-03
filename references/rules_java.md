# Java Bug Rules

These rules apply to Java files only (`.java`).

---

## JAVA-001 — NullPointerException Risk
**Severity**: HIGH  
**Pattern**: A reference is dereferenced (method called, field accessed) without a prior null check,
and it could be null based on the call site, return value of a method that can return null, or an
optional/nullable annotation.  
**Natural language**: Look for method calls or field accesses on objects returned from maps
(`map.get(key)`), optional-returning methods, or parameters not annotated `@NonNull`.

---

## JAVA-002 — ConcurrentModificationException
**Severity**: HIGH  
**Pattern**: A collection is modified (add, remove) while being iterated with a for-each loop or
iterator, outside of a `CopyOnWriteArrayList` or `Collections.synchronized*` wrapper.  
**Natural language**: Look for loops over a `List`/`Map`/`Set` that call `remove()`/`add()` on the
same collection inside the loop body without using `Iterator.remove()`.

---

## JAVA-003 — Incorrect `equals`/`hashCode` Implementation
**Severity**: MEDIUM  
**Pattern**: A class overrides `equals()` but not `hashCode()` (or vice versa), breaking the
contract required for correct behavior in `HashMap`/`HashSet`.  
**Natural language**: Look for classes that define `equals()` without a corresponding `hashCode()`
override, or where one uses a field that the other doesn't.

---

## JAVA-004 — Race Condition on Non-Atomic Compound Operation
**Severity**: HIGH  
**Pattern**: A check-then-act or read-modify-write on a shared variable that is not atomic, even if
the individual reads/writes are synchronized.  
**Natural language**: In FE/BE Java code, look for patterns like `if (map.containsKey(k)) { map.get(k).use(); }`
or `counter++` on shared state without proper synchronization or use of `AtomicInteger`/`ConcurrentHashMap`.

---

## JAVA-005 — Resource Leak (AutoCloseable Not Closed)
**Severity**: MEDIUM  
**Pattern**: A resource implementing `AutoCloseable` (`InputStream`, `Connection`, `Statement`,
`ResultSet`, etc.) is created but not closed in a `try-with-resources` block or `finally` clause.  
**Natural language**: Look for `new FileInputStream(...)`, `conn.prepareStatement(...)`, etc. that
are not wrapped in `try (Resource r = ...)` or explicitly closed in `finally`.

---

## JAVA-006 — Swallowed Exception
**Severity**: MEDIUM  
**Pattern**: A `catch` block is empty, or only contains a comment/TODO, silently discarding an
exception that should be logged or re-thrown.  
**Natural language**: Look for `catch (Exception e) {}` or `catch (Exception e) { // ignore }`.

---

## JAVA-007 — String Comparison with `==`
**Severity**: HIGH  
**Pattern**: Two `String` objects are compared using `==` or `!=` instead of `.equals()`.  
**Natural language**: Look for `if (str == "literal")` or `if (a == b)` where both are `String` type.

---

## JAVA-008 — Integer Cache Pitfall
**Severity**: LOW  
**Pattern**: `Integer` (boxed) objects are compared with `==`/`!=` instead of `.equals()`, which
works correctly only for values in the JVM cache range (-128 to 127).  
**Natural language**: Look for `if (integerObj1 == integerObj2)` where the values could be outside
the cached range.

---

## JAVA-009 — Deadlock Risk
**Severity**: HIGH  
**Pattern**: Two or more locks are acquired in different orders in different code paths, creating a
potential deadlock.  
**Natural language**: Look for `synchronized(lockA) { synchronized(lockB) { ... } }` in one method
and `synchronized(lockB) { synchronized(lockA) { ... } }` in another. Also look for calling an
external/overridable method while holding a lock (alien method call).

---

## JAVA-010 — Incorrect `List.subList` Usage
**Severity**: MEDIUM  
**Pattern**: The result of `List.subList()` is used after the original list has been structurally
modified, which throws `ConcurrentModificationException`.  
**Natural language**: Look for `subList()` results stored and used after the parent list is modified.

---

## JAVA-011 — Missing `volatile` on Shared Flag
**Severity**: MEDIUM  
**Pattern**: A boolean or reference field used as a stop/done flag between threads is not declared
`volatile` or accessed via `AtomicBoolean`, allowing the JVM to cache the value in a register.  
**Natural language**: Look for `boolean running = true;` or similar flags read in one thread and
written in another without `volatile`, `synchronized`, or `Atomic*`.

---

## JAVA-012 — StarRocks FE: Incorrect Metadata Lock Usage
**Severity**: HIGH  
**Pattern** (StarRocks-specific): In FE catalog/metadata code, database/table metadata is accessed
or modified without holding the appropriate read or write lock (`db.readLock()`/`db.writeLock()`),
or locks are held across slow I/O operations creating unnecessary contention.  
**Natural language**: Look for access to `Database`, `Table`, or `Partition` objects without
surrounding lock/unlock calls, or lock regions that include RPC calls, file I/O, or sleep.

---

## JAVA-013 — StarRocks FE: Task/Job Status Not Persisted
**Severity**: MEDIUM  
**Pattern** (StarRocks-specific): A job/task state change (status update, progress update) is made
in memory but `editLog` is not called to persist the change, meaning it will be lost on FE restart.  
**Natural language**: Look for state changes to `Job`, `LoadJob`, `AlterJob`, or similar objects
where `GlobalStateMgr.getCurrentState().getEditLog().log*()` is not called afterward.

---

> **Note**: StarRocks FE **optimizer** rules (`com.starrocks.sql.optimizer`,
> `com.starrocks.qe.feedback`) live in a separate file — see `rules_optimizer.md` (`OPT-NNN`).
