---
name: general_task_sop
version: 1.0.0
target_models: [gemini-1.5-pro, gpt-4o, claude-3-5-sonnet]
max_loops: 8
---

# SOP: General Task & Feature Request Execution

## 1. Objective
Guide general agent feature development, documentation tasks, or exploratory requests efficiently with minimal scope creep.

## 2. Preconditions & Required Context
- Explicit user prompt describing target goal or feature requirement.
- Relevant workspace files and documentation.

## 3. Step-by-Step Execution Workflow

### Phase 1: Ambiguity & Intent Assessment
1. Determine if the request is investigatory or actionable.
2. If investigatory, answer directly without creating unnecessary code changes.
3. If actionable code/doc change, identify target files and dependencies before editing.

### Phase 2: Implementation
1. Perform minimal, targeted file creations or updates.
2. Maintain documentation integrity and preserve unrelated code comments.

### Phase 3: Verification & Summary
1. Run verification checks (build, test, lint) if applicable.
2. Provide a clear, structured summary of actions completed with clickable markdown links to modified files.

## 4. Strict Safety Guardrails
> [!NOTE]
> - Never volunteer unexpected scope additions without user request.
> - Always check existing utilities before creating new helper functions.

## 5. Output Interface Schema
Returns structured response containing summary, status, and affected files list.
