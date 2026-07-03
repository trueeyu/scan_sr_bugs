## StarRocks Bug Scanner

A Claude Code skill for scanning C++ and Java source files from the StarRocks codebase (or similar database/systems projects) to detect potential bugs based on a curated set of rules.

## Overview

This skill applies natural-language and pattern-based rules to source files and produces a structured bug report covering severity, location, and suggested fixes.

## Project Structure

```
scan_sr_bugs/
├── SKILL.md                    # Skill definition and workflow
├── references/
│   ├── rules_common.md         # Rules applicable to both C++ and Java
│   ├── rules_cpp.md            # C++-specific rules
│   ├── rules_java.md           # Java-specific rules
│   └── rules_optimizer.md      # StarRocks FE optimizer rules (OPT-NNN)
└── README.md
```

## Usage

Invoke the skill by providing a C++ or Java source file path and asking for a scan. Example prompts:

- "Scan this file: `be/src/exec/scan_node.cpp`"
- "Check `fe/src/main/java/.../Planner.java` for bugs"
- "Review this code for issues"

The scanner will:

1. Read the file and detect the language by extension (`.cpp`/`.h`/`.cc` → C++, `.java` → Java).
2. Load `references/rules_common.md` plus the language-specific rules file, and
   `references/rules_optimizer.md` when the Java file is under the StarRocks FE optimizer
   packages (`com.starrocks.sql.optimizer` / `com.starrocks.qe.feedback`).
3. Apply each rule and collect findings.
4. Emit a structured report listing findings with severity, rule ID, line numbers, code excerpt, explanation, and suggestion.

You may also supply additional inline rules in your prompt; these are assigned IDs `USR-001`, `USR-002`, etc.

## Output Format

Findings are reported as a table followed by per-finding details and a severity summary (HIGH / MEDIUM / LOW). If nothing is found, the report shows `No issues detected`.

See `SKILL.md` for the full output template.

## Adding Rules

- Add cross-language rules to `references/rules_common.md` (`CMN-NNN`).
- Add C++ rules to `references/rules_cpp.md` (`CPP-NNN`).
- Add Java rules to `references/rules_java.md` (`JAVA-NNN`).
- Add StarRocks FE optimizer rules to `references/rules_optimizer.md` (`OPT-NNN`).

Each rule should include an ID, severity, description, and where possible a code example illustrating the bug pattern.