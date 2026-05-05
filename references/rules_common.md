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
