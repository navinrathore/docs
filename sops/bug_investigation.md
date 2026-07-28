---
name: bug_investigation_sop
version: 1.2.0
target_models: [gemini-1.5-pro, gpt-4o, claude-3-5-sonnet]
max_loops: 10
---

# SOP: Bug Investigation & Root Cause Remediation

## 1. Objective
Systematically diagnose, isolate root cause, and implement minimal targeted fixes for reported bugs without causing regressions or breaking public API contracts.

## 2. Preconditions & Required Context
- Target codebase workspace path.
- Reproducible steps or failure stack traces provided in user request.
- Permissions for file reading/writing and local test execution.

## 3. Step-by-Step Execution Workflow

### Phase 1: Log & Traceback Analysis (Mandatory First Step)
1. **Fetch Raw Logs:** Read the full untruncated stack trace or error log before forming any diagnostic hypothesis.
2. **Locate Failure Site:** Identify exact file paths, line numbers, and failing assertion metrics.
3. **Inspect Active Source:** View surrounding lines around the failure site using file viewing tools. Never guess logic or parameter signatures.

### Phase 2: Reproduction & Isolation
1. Run existing unit tests to reproduce the failure cleanly.
2. Isolate whether failure originates from upstream data parsing, internal state mutation, or tool invocation.

### Phase 3: Targeted Remediation
1. Design a minimal, surgical fix addressing the root cause (do not mask symptoms).
2. Preserve existing docstrings, API signatures, and unrelated formatting.

### Phase 4: Runtime Verification & Walkthrough
1. Re-run automated test suite to verify fix.
2. Confirm zero regressions across adjacent tests.
3. Record test output in walkthrough documentation.

## 4. Strict Safety Guardrails
> [!CAUTION]
> - **NEVER** mask symptoms by swallowing exceptions with silent `try/except: pass`.
> - **NEVER** modify or delete failing unit test assertions without user confirmation.
> - **NEVER** edit code without inspecting the full source definition first.

## 5. Output Interface Schema
Returns structured JSON matching the `BugResolutionSummary` schema:
```json
{
  "root_cause": "Detailed explanation of why the bug occurred",
  "files_modified": ["src/parser.py"],
  "test_command_executed": "pytest tests/test_parser.py",
  "verification_passed": true
}
```
