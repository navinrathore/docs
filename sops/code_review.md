---
name: code_review_sop
version: 1.1.0
target_models: [gemini-1.5-pro, gpt-4o, claude-3-5-sonnet]
max_loops: 5
---

# SOP: Automated Code & Pull Request Review

## 1. Objective
Perform thorough, automated code reviews evaluating software quality, security vulnerabilities, performance bottlenecks, and adherence to architectural guidelines.

## 2. Preconditions & Required Context
- Modified git diff or changed files list.
- Project architectural guidelines (`AGENTS.md` or linters).

## 3. Step-by-Step Execution Workflow

### Phase 1: Context & Diff Parsing
1. Inspect the modified lines and files in the diff.
2. Read surrounding file context to ensure full understanding of changed call sites and signatures.

### Phase 2: Multi-Dimensional Evaluation
1. **Correctness & Edge Cases:** Check for null dereferences, unhandled exceptions, and off-by-one errors.
2. **Security & Data Boundaries:** Scan for injection risks, unhandled input sanitization, and exposed credentials.
3. **Performance & Caching:** Check for unindexed DB queries, N+1 loops, or missing KV-cache optimization.
4. **Architectural Consistency:** Verify adherence to project design patterns and guardrails.

### Phase 3: Feedback Synthesis
1. Categorize findings by severity: `CRITICAL`, `WARNING`, `SUGGESTION`.
2. Provide concrete, copy-pasteable code diffs for any proposed changes.

## 4. Strict Safety Guardrails
> [!IMPORTANT]
> - Do not comment on purely subjective formatting if automated linters handle it.
> - Always justify security warnings with concrete attack vector examples.

## 5. Output Interface Schema
Returns structured JSON matching the `CodeReviewReport` schema:
```json
{
  "review_status": "APPROVED_WITH_COMMENTS",
  "critical_issues_count": 0,
  "findings": [
    {
      "severity": "WARNING",
      "file": "src/auth.py",
      "line_number": 42,
      "message": "Potential unhandled token expiration exception",
      "suggested_fix": "Add explicit TokenExpiredError handling block"
    }
  ]
}
```
