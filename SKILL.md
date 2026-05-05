---
name: starrocks-bug-scanner
description: >
  Scan C++ or Java source files from the StarRocks (or any similar database/systems) codebase for
  potential bugs based on a set of rules. Use this skill whenever the user provides a file path and
  wants to detect bugs, issues, or code smells — even if they just say "scan this file", "check for
  bugs", "review this code for issues", or "analyze this C++/Java file". Also triggers when the user
  mentions StarRocks, BE (Backend Engine), FE (Frontend), or asks about code quality in a systems
  codebase.
---

# StarRocks Bug Scanner

Scan a given C++ or Java source file for potential bugs using a combination of natural-language rules
and pattern-based rules. Output a structured report of findings.

---

## Workflow

1. **Read the file** at the given path (use `bash_tool` with `cat` or `view` tool).
2. **Identify the language** (C++ if `.cpp/.h/.cc`, Java if `.java`).
3. **Load the relevant rules** from `references/rules_cpp.md` or `references/rules_java.md` (plus
   `references/rules_common.md` for rules that apply to both).
4. **Scan the file** by reading its content carefully and applying each rule.
5. **Output the report** in the structured format below.

---

## Output Format

```
## Bug Scan Report: <filename>
Language: C++ | Java
Scanned: <datetime>

### Findings (<N> issues found)

| # | Severity | Rule ID | Line(s) | Description |
|---|----------|---------|---------|-------------|
| 1 | HIGH     | CPP-003 | 42      | Potential null pointer dereference before check |
| 2 | MEDIUM   | CMN-001 | 87-91   | Resource not released on early return |
...

### Details

#### Finding #1 — [HIGH] CPP-003: Null Pointer Dereference
**Location**: Line 42
**Code**:
```cpp
ptr->method(); // called before null check on line 45
```
**Explanation**: `ptr` is used before the null check at line 45. If `ptr` is null, this will crash.
**Suggestion**: Move the null check before first use, or add an assertion.

...

### Summary
- HIGH: N
- MEDIUM: N  
- LOW: N
- Total: N
```

If no issues are found, output:
```
### Findings: No issues detected ✓
```

---

## Rules Loading Guide

- Always load `references/rules_common.md` regardless of language.
- For C++ files: also load `references/rules_cpp.md`
- For Java files: also load `references/rules_java.md`
- The user may provide **additional rules** inline in their prompt — treat these with the same priority
  as the built-in rules, and assign them Rule ID `USR-001`, `USR-002`, etc.

---

## Scanning Tips

- Focus on **actual bugs**, not style issues (unless a style issue can cause a bug).
- When uncertain, report with LOW severity and explain the uncertainty.
- For large files (>1000 lines), do a full pass — do not truncate your analysis.
- Always quote the specific lines of code in the Details section.
- If the file cannot be read (missing, binary, etc.), report the error clearly.
