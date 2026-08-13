# C++ Bug Rules

These rules apply to C++ files only (`.cpp`, `.h`, `.cc`).

---

## CPP-001 — Null Pointer Dereference
**Severity**: HIGH  
**Pattern**: A pointer is dereferenced (`->`, `*`) before being checked for null, or after a code
path where it could have been set to null/not initialized.  
**Natural language**: Look for pointer dereferences that happen before the corresponding null check,
or where the null check exists only in some branches.

---

## CPP-002 — Use After Free / Use After Move
**Severity**: HIGH  
**Pattern**: A raw pointer is used after `delete`/`free()`, or an object is used after
`std::move()` has been called on it.  
**Natural language**: Look for variables that are `delete`d or `std::move()`d and then accessed
again later in the same scope or in a destructor.

---

## CPP-003 — Double Free
**Severity**: HIGH  
**Pattern**: `delete` or `free()` called more than once on the same pointer without reassigning it
to null or a new value in between.  
**Natural language**: Look for raw pointers that are explicitly freed in multiple places (e.g., both
in a destructor and in an error-handling path).

---

## CPP-004 — Uninitialized Variable
**Severity**: HIGH  
**Pattern**: A local variable is declared but not initialized before its first use, especially in
structs/POD types where default initialization doesn't apply.  
**Natural language**: Look for `int x;`, `bool flag;`, `char buf[N];` etc. that are declared and
then read without a guaranteed prior assignment.

---

## CPP-005 — Data Race / Missing Mutex Protection
**Severity**: HIGH  
**Pattern**: A shared variable (class member, global) is accessed from multiple places, some of
which are in threaded contexts, without consistent mutex/lock protection.  
**Natural language**: In StarRocks BE code, look for member variables that are read or written in
functions called from different threads (scanner threads, pipeline threads, RPC handlers) without
holding a lock. Check for inconsistent locking — locked in write path but not read path.

---

## CPP-006 — Iterator Invalidation
**Severity**: HIGH  
**Pattern**: An iterator or reference into a container (`vector`, `map`, `unordered_map`) is used
after an operation that could invalidate it (insert, erase, resize, rehash).  
**Natural language**: Look for loops that iterate over a container while also modifying it, or
iterators/references saved before a `push_back`/`insert`/`erase` and used after.

---

## CPP-007 — Signed/Unsigned Mismatch in Comparison
**Severity**: MEDIUM  
**Pattern**: Comparison between a signed integer (`int`, `int64_t`) and an unsigned integer
(`size_t`, `uint32_t`) which can cause unexpected behavior when the signed value is negative.  
**Natural language**: Look for `if (i < vec.size())` where `i` is a signed type — if `i` is
negative, it wraps around and the comparison is meaningless.

---

## CPP-008 — Exception Safety Violation
**Severity**: MEDIUM  
**Pattern**: A function acquires a resource, then calls a function that may throw, but does not use
RAII (smart pointer, `unique_ptr`, scope guard) to ensure cleanup on exception.  
**Natural language**: Look for raw `new` without `unique_ptr`, or `lock()` without a `lock_guard`,
followed by code that could throw.

---

## CPP-009 — Incorrect Use of `shared_ptr` Causing Cycles
**Severity**: MEDIUM  
**Pattern**: Two objects hold `shared_ptr` to each other (A → B → A), which will prevent
destruction and cause memory leaks.  
**Natural language**: Look for circular `shared_ptr` references. One side of the cycle should use
`weak_ptr` instead.

---

## CPP-010 — Missing Virtual Destructor
**Severity**: MEDIUM  
**Pattern**: A class with virtual methods does not declare a virtual destructor, meaning derived
class destructors won't be called when deleting via a base pointer.  
**Natural language**: Look for base classes that have `virtual` methods but whose destructor is
either missing or non-virtual, especially if they are used as base pointers.

---

## CPP-011 — Buffer Overflow via `memcpy`/`memset`
**Severity**: HIGH  
**Pattern**: `memcpy`, `memset`, or `memmove` called with a size that could exceed the destination
buffer's allocated size.  
**Natural language**: Look for `memcpy(dst, src, n)` where `n` is derived from user input or a
source size without being capped to the destination size.

---

## CPP-012 — Return Reference/Pointer to Local Variable
**Severity**: HIGH  
**Pattern**: A function returns a reference or pointer to a local (stack-allocated) variable, which
becomes a dangling reference/pointer after the function returns.  
**Natural language**: Look for `return &local_var;` or `return local_var;` where `local_var` is a
local object and the return type is a reference or pointer.

---

## CPP-013 — Incorrect `sizeof` on Pointer
**Severity**: MEDIUM  
**Pattern**: `sizeof(ptr)` used where `sizeof(*ptr)` or the actual element size was intended,
typically in `memcpy`/`malloc` calls.  
**Natural language**: Look for `sizeof` applied to a pointer variable (gives 8 on 64-bit) where the
intent was to get the size of the pointed-to object.

---

## CPP-014 — TOCTOU (Time-of-Check to Time-of-Use)
**Severity**: MEDIUM  
**Pattern**: A condition is checked (e.g., file existence, pointer non-null, container non-empty)
and then the resource is used, but the state could change between the check and use in a multithreaded
context.  
**Natural language**: In multithreaded BE code, look for `if (!map.empty()) { map.begin()-> ... }`
or `if (ptr) { use(ptr); }` patterns where another thread could modify the state between the check
and the use without holding a lock across both operations.

---

## CPP-015 — Brpc/Network Buffer Not Checked
**Severity**: MEDIUM  
**Pattern** (StarRocks-specific): In Brpc RPC handlers or network code, the deserialized message
fields are used without checking whether they are set/valid (proto3 fields have default values and
won't fail deserialization even if missing).  
**Natural language**: Look for protobuf message fields accessed directly without checking
`has_field()` or validating the value makes sense (e.g., size > 0, index in range).

---

## CPP-016 — Unsafe Column Sharing / Column Reuse Causing Crash
**Severity**: HIGH  
**Pattern** (StarRocks-specific): A `Column` (especially a `NullColumn` inside a `NullableColumn`)
is shared between multiple owners — e.g. two `NullableColumn`s share the same `NullColumn`, two
slots in a `Chunk` share the same `Column`, or different `Chunk`s share the same `Column` — so that
later in-place mutation of one (filter, append, resize, assign_value) silently corrupts the others
and triggers `Check failed: _data_column->size() == _null_column->size()` or similar size/state
mismatches.

**Common unsafe patterns to flag**:
1. `std::move(*nullable->null_column()).mutate()` or
   `std::move(*nullable->data_column()).mutate()` — when the source `ColumnPtr`'s `use_count == 1`
   the COW `mutate()` returns the **same** object instead of cloning, so the resulting
   `MutableColumnPtr` aliases the input's null/data column. The safe replacement is
   `Column::mutate(nullable->null_column())` (or `NullColumn::static_pointer_cast(Column::mutate(...))`),
   which takes a `ColumnPtr` by value, bumps `use_count >= 2`, and forces a clone.
2. Building a `NullableColumn` (or any composite column) by directly reusing another column's
   internal `null_column()` / `data_column()` pointer without going through `Column::mutate(...)` or
   an explicit `clone()`.
3. Putting the same `ColumnPtr` into two different slot positions of a single `Chunk`, or appending
   the result of an expression that returned an aliased column from the input chunk as a new slot
   (e.g. expression evaluators like `map_apply`, `array_length`, `array_*`, `map_*`, binary
   functions, `ngram` that derive a result column from an input's null/data column).
4. Caching a `ColumnPtr` across chunks (member field, static, captured by lambda) and then handing
   it back into a new `Chunk` whose downstream operators may filter/resize it.

**Natural language**: In StarRocks BE expression / function code, flag any place that obtains a
mutable child column from a `NullableColumn` (or other composite column) via
`std::move(*x->null_column()).mutate()` / `std::move(*x->data_column()).mutate()` — recommend
`Column::mutate(x->null_column())` instead. Also flag constructions where a new `NullableColumn` /
chunk slot is built that reuses an input column's `null_column()` or `data_column()` pointer
directly. Be suspicious of any code path that produces a result column for a new slot in the same
`Chunk` while structurally sharing storage with an existing slot — downstream `Chunk::filter`,
`append`, `resize`, or per-column mutation will desynchronize sizes and crash. Reference fix:
StarRocks PR #71258 (and #71207).

---

## CPP-017 — Thread Name Exceeds pthread_setname_np 15-Character Limit
**Severity**: LOW
**Pattern** (StarRocks-specific): A thread (or thread-pool / scanner / compaction / bthread / loader
worker) is created with a `name` string whose length is **>= 16 characters**. On Linux,
`pthread_setname_np` restricts the name to 16 bytes **including** the terminating null byte, i.e.
at most **15 visible characters**. When the limit is exceeded:

1. `Thread::set_thread_name()` in `be/src/common/thread/thread.cpp` truncates the name in place by
   doing `str.at(15) = '\0'` — this writes a NUL into the middle of a `std::string` but does NOT
   shrink `size()`, leaving an embedded NUL inside the buffer.
2. `pthread_setname_np` is then called with that buffer; on some kernels / glibc versions the call
   still returns non-zero (`ERANGE`), and `Thread::set_thread_name` falls through to
   `LOG(WARNING) << "failed to set thread name: " << name;`.
3. Because `name` (and/or the truncated `str`) contains the embedded NUL, the warning prints with a
   trailing `^@` byte, producing log lines like:

   ```
   W20260518 14:43:15.781343 23109709274176 thread.cpp:279] failed to set thread name: compact_data_di^@
   ```

   These warnings are noisy, can mask real issues during incident triage, and indicate the thread
   is running with a truncated / mis-labeled name (making `top -H`, `pstack`, perf, and gdb harder
   to read).

**Common unsafe patterns to flag**:
1. Any call to `Thread::set_thread_name(...)`, `pthread_setname_np(...)`, `bthread_setname(...)`,
   `prctl(PR_SET_NAME, ...)` where the second argument is a string literal of **16 or more**
   characters, e.g. `"compact_data_directory"`, `"segment_replicate_executor"`,
   `"async_delta_writer_pool"`.
2. `ThreadPoolBuilder("...").set_...` / `Thread::create("category", "name", ...)` /
   `ThreadPool::set_min_threads(... ,"name")` where the `name` argument (the per-worker name, not
   the category) is a literal with `length() >= 16`. The category is fine; the worker name is what
   gets passed down to `pthread_setname_np`.
3. Names built by concatenation that obviously exceed 15 chars at compile time, e.g.
   `"compact_data_" + dir_suffix` where the prefix alone is 13 chars and the suffix is non-empty
   and bounded (`"_directory"`, partition id, etc.). Flag if any reasonably-sized suffix pushes the
   total past 15.
4. Reusing a long human-readable subsystem name (`"PublishVersionDaemon"`,
   `"LakeTabletScheduler"`) directly as the worker thread name instead of a short abbreviation.

**Natural language**: Scan for any string literal or easily-bounded string expression that is
passed as a thread/worker/bthread name (function names containing `set_thread_name`,
`setname_np`, `set_name`, `set_worker_name`, `prctl(PR_SET_NAME`, or the `name` field of
`ThreadPoolBuilder`, `Thread::create`, `std::thread` wrappers, scanner/compaction worker
constructors, brpc `bthread` task names). If the resulting name is >= 16 characters, report it —
suggest shortening to <= 15 chars (e.g. `compact_data_directory` → `compact_dir`,
`segment_replicate_executor` → `seg_replicate`). Do **not** flag names <= 15 chars. Do not flag
the *category* argument of `Thread::create(category, name, ...)` — only the per-thread `name`.
Reference symptom: warning lines from `be/src/common/thread/thread.cpp` of the form
`failed to set thread name: <name>^@`.

---

## CPP-018 — Captured Status Discarded in Favor of a Hardcoded Success Return
**Severity**: HIGH
**Pattern** (StarRocks-specific): A function returning `Status` calls something that can fail and
captures the result into a local (`auto res = ...;` / `Status st = ...;`), but the function's actual
`return` statement is a hardcoded success (`return Status::OK();`) instead of the captured variable.
The variable is written but never read, so any error the callee reports — spill failure, closed
exchanger, backpressure, RPC failure — is silently swallowed and execution proceeds as if it had
succeeded. This is easy to miss on a quick read because the status *looks* handled (it's captured
into a named variable, not dropped outright), but the capture is dead.

This is distinct from CMN-003 (return value never captured at all — discarded outright with no
variable) and CMN-006 (error handling present but inconsistent across branches): here there is a
single straight-line path where the value is captured and then unconditionally overridden by a
hardcoded success before returning.

**Common unsafe patterns to flag**:
1. `auto res = <fallible_call>(...); ... return Status::OK();` (or any other hardcoded
   success/default status) where `res` is never referenced between its assignment and the function's
   return.
2. Pipeline operator methods (`push_chunk`, `pull_chunk`, `set_finishing`, `process_chunk`, `send`,
   `close`) that capture an exchanger/channel/sink call's `Status` and then return a fixed value
   instead of propagating it.
3. A sibling/peer implementation (another operator class for a different exchange strategy, an
   overload, a near-duplicate function) that correctly `return`s the captured status — making this
   instance a copy-paste regression relative to the established pattern.

**Natural language**: In functions returning `Status` (or an equivalent result type), find calls that
can fail and are captured into a local variable, then check whether that variable is actually used
before the function returns — if the function instead returns a different, hardcoded success value,
flag it: the error path is being dropped. When present, cross-check sibling/peer classes or overloads
performing the same kind of call to see whether they correctly propagate the status, which both
confirms the bug and shows the expected fix.

**Reference fix — StarRocks PR #77737** (https://github.com/StarRocks/starrocks/pull/77737):
`GroupedExecutionSinkOperator::push_chunk()` captured `_exchanger->accept()`'s return value into
`res` but then unconditionally `return Status::OK();`, silently swallowing any error the exchanger
reported (e.g. spill failures, closed-exchanger states). The sibling
`LocalExchangeSinkOperator::push_chunk()` already returned this value correctly, exposing the
inconsistency.
```cpp
// BEFORE (buggy) — res captured but never returned
Status GroupedExecutionSinkOperator::push_chunk(RuntimeState* state, const ChunkPtr& chunk) {
    auto res = _exchanger->accept(chunk, _driver_sequence);
    _peak_memory_usage_counter->set(_exchanger->get_memory_usage());
    return Status::OK();
}

// AFTER (fixed) — propagate the captured status
Status GroupedExecutionSinkOperator::push_chunk(RuntimeState* state, const ChunkPtr& chunk) {
    auto res = _exchanger->accept(chunk, _driver_sequence);
    _peak_memory_usage_counter->set(_exchanger->get_memory_usage());
    return res;
}
```
