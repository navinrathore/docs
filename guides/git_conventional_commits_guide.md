# Git Conventional Commits & Commit Message Standards Guide

## Overview

This guide documents standard Git commit message prefixes and formatting guidelines across repositories. Adhering to standard commit message conventions improves repository readability, simplifies automated changelog generation, and makes project history clear for reviewers and collaborators.

---

## Commit Message Anatomy

A conventional commit message follows this structure:

```text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Format Rules
1. **Subject Line**: Use imperative, present tense ("add", "fix", "change", NOT "added", "fixes", "changing").
2. **No Trailing Period**: Do not end the subject line with a period `.`.
3. **Character Limit**: Keep the subject line under 50-72 characters.
4. **Lower Case**: Write subject line and prefixes in lowercase.

---

## Major Commit Types & Prefixes

| Prefix | Category | Purpose | Example |
| :--- | :--- | :--- | :--- |
| **`feat`** | Feature | Adds a new feature or functionality to the project | `feat(auth): implement JWT token rotation` |
| **`fix`** | Bug Fix | Resolves a bug, exception, or unintended behavior | `fix(eval): prevent race condition in async metric aggregation` |
| **`docs`** | Documentation | Documentation changes only (README, docstrings, guide files) | `docs(readme): update environment setup instructions` |
| **`style`** | Code Style | Formatting changes that do not affect code logic (whitespace, linting) | `style: format python files with ruff` |
| **`refactor`** | Refactoring | Code restructuring without adding a feature or fixing a bug | `refactor(llm): decouple provider interface from model class` |
| **`perf`** | Performance | Code optimization specifically aimed at improving performance | `perf(db): vectorize query embedding batching` |
| **`test`** | Testing | Adding missing tests or correcting existing tests | `test(core): add unit test coverage for prompt renderer` |
| **`build`** | Build & Tooling | Changes affecting build systems, dependencies, or external tools | `build(deps): bump pydantic from 2.5 to 2.8` |
| **`ci`** | CI/CD Workflows | Modifications to CI configuration files and automation scripts | `ci(github): add workflow for automated test execution` |
| **`chore`** | Maintenance | Auxiliary tasks, routine updates, or non-src file changes | `chore: update .gitignore rules for temporary artifacts` |
| **`revert`** | Reversion | Reverts a previous commit due to regressions or issues | `revert: revert "feat(auth): implement OAuth2 handler"` |
| **`init`** | Initialization | Initial commit, project bootstrapping, or scaffolding | `init: initialize llmops_core library structure` |

---

## Advanced Usage

### 1. Specifying Scope
Scopes provide contextual domain information in parentheses immediately following the commit type (e.g., `feat(auth): ...`).

#### Typical Scope Categories

- **Core & Architecture**: `core`, `api`, `db`, `models`, `auth`, `router`, `cli`
- **Frontend / UI**: `ui`, `components`, `styles`, `views`, `i18n`
- **Domain / Feature Modules** (e.g., for AI/LLMOps):
  - `llm` / `client` (LLM provider orchestration)
  - `eval` / `judge` (Evaluations and metrics)
  - `prompt` (Prompt templates & registries)
  - `telemetry` (Logging, tracing, metrics)
  - `guardrails` (Safety & content moderation)
  - `voice` / `audio` (Speech-to-text, text-to-speech)
- **Infrastructure & Environment**: `deps`, `docker`, `infra`, `config`, `env`, `github`
- **Documentation & SOPs**: `readme`, `sop`, `architecture`, `changelog`

#### Examples
- `feat(api): add endpoint for batch telemetry export`
- `fix(eval): handle null LLM judge outputs gracefully`
- `build(deps): upgrade fastapi to v0.111.0`


### 2. Breaking Changes
Signal breaking API changes by appending a `!` before the colon or adding a `BREAKING CHANGE:` footer:
- **With `!` modifier**:
  ```text
  feat(client)!: change default model client initialization signature
  ```
- **With Footer**:
  ```text
  feat(client): update provider interface parameters

  BREAKING CHANGE: `llm_client.generate()` now requires explicit `model_id`.
  ```

### 3. Referencing Issues
Link or close tracked issues in the footer:
```text
fix(parser): support streaming chunk JSON payloads

Closes #142
```

---

## Best Practices Summary

- **Be Specific**: Write self-contained summaries explaining *what* was changed and *why*.
- **Atomicity**: Make commits atomic (one logical change per commit).
- **Consistency**: Stick strictly to standard prefixes to allow automated tooling (e.g., semantic release, changelog generators) to parse commits accurately.
