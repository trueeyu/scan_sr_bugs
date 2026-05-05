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
