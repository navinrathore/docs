# Skills & Commands Consolidation Plan

**Created**: 2026-11-19 | **Status**: Active
**Source Audit**: [Skills Audit Report](file:///home/navin/.gemini/antigravity/brain/3fdeb5e1-8869-4d57-ad87-2f263b8f43ad/skills-audit-report.md)

---

## 1. Current Landscape

### Source of Truth
All custom skills are managed from the **dotfiles-style repo**:
- **Repo**: [tooling/antigravity-skills/](file:///home/navin/work/AI/tooling/antigravity-skills/)
- **Enabled**: [enabled/](file:///home/navin/work/AI/tooling/antigravity-skills/enabled/) → symlinked to `~/.gemini/config/skills/`
- **Disabled**: [disabled/](file:///home/navin/work/AI/tooling/antigravity-skills/disabled/)
- **Sync**: [sync.sh](file:///home/navin/work/AI/tooling/antigravity-skills/sync.sh) bridges KIs + Active Skills

### Workspace Skills (Not in Dotfiles Repo)
- [critique](file:///home/navin/work/AI/.agents/skills/critique/SKILL.md), [find_bug](file:///home/navin/work/AI/.agents/skills/find_bug/SKILL.md), [project_dossier](file:///home/navin/work/AI/.agents/skills/project_dossier/SKILL.md), [recipe_card](file:///home/navin/work/AI/.agents/skills/recipe_card/SKILL.md)
- These live in `.agents/skills/` and are workspace-scoped, not globally synced.

### Legacy Workflows (Pre-Skill Era)
- [.agents/workflows/](file:///home/navin/work/AI/.agents/workflows/) — 6 files, 3 redundant

---

## 2. Sync Infrastructure Issues

> [!CAUTION]
> `sync.sh` targets a **stale KI path**: `~/.gemini/antigravity-ide/knowledge/`
> The actual knowledge directory is `~/.gemini/antigravity/knowledge/`.
> KI symlinks are created but never read by the current runtime.

**Fix required in** [sync.sh](file:///home/navin/work/AI/tooling/antigravity-skills/sync.sh):
```bash
# OLD (broken):
KI_DIR="$HOME/.gemini/antigravity-ide/knowledge"
# NEW (correct):
KI_DIR="$HOME/.gemini/antigravity/knowledge"
```

---

## 3. Consolidation Actions

### 🔴 Phase 1: Cleanup (Delete Dead Weight)

| Action | Target | Reason |
|:-------|:-------|:-------|
| **Delete** | `.agents/workflows/critique.md` | Superseded by `critique` skill |
| **Delete** | `.agents/workflows/find-bug.md` | Superseded by `find_bug` skill |
| **Delete** | `.agents/workflows/project-dossier.md` | Superseded by `project_dossier` skill |
| **Delete** | `enabled/llm_judge/` | Deprecated; redirect already in place. Remove from dotfiles repo entirely. |
| **Deprecate** | `enabled/manage_ki/` | Move to `disabled/`. Functionality covered by `/learn` slash command. |
| **Fix** | `sync.sh` | Update KI_DIR path from `antigravity-ide` → `antigravity` |

### 🟡 Phase 2: Fix Broken References

| Skill | File | Fix |
|:------|:-----|:----|
| `feature_spec` | [SKILL.md L22](file:///home/navin/.gemini/config/skills/feature_spec/SKILL.md#L22) | `AskUserQuestion` → `ask_question` |
| `manage_backlog` | [SKILL.md L16](file:///home/navin/.gemini/config/skills/manage_backlog/SKILL.md#L16) | `AskUserQuestion` → `ask_question` |
| `llm_stats` | [analyzer.py L35](file:///home/navin/.gemini/config/skills/llm_stats/scripts/analyzer.py#L35) | Support both `antigravity-ide/brain/` and `antigravity/brain/` paths |
| `manage_ki` | [SKILL.md L12](file:///home/navin/.gemini/config/skills/manage_ki/SKILL.md#L12) | Path `antigravity-ide/knowledge/` → `antigravity/knowledge/` (if keeping) |

### 🟢 Phase 3: Promote Legacy Workflows to Skills

| Workflow | Action | New Skill Name | Location |
|:---------|:-------|:---------------|:---------|
| `capture-qa.md` + `capture-goal.md` | **Merge & Promote** | `capture_qa` | `enabled/capture_qa/SKILL.md` |
| `list-hf-models.md` | **Promote** | `list_hf_models` | `enabled/list_hf_models/SKILL.md` |

**`capture_qa` merge plan:**
- Base: `capture-qa.md` (the more complete version)
- Add `--autonomous` / `--bg` flags from `capture-goal.md`
- Make vault paths configurable (not hardcoded to `x_objective/`)
- Add `--silent` mode from existing spec

### 🔵 Phase 4: Enhance Existing Skills

| Skill | Enhancement | Priority |
|:------|:------------|:---------|
| `changelog` | Conventional commit parsing (`feat:`, `fix:`), version/tag support, `--dry-run` | Medium |
| `manage_backlog` | Priority labels (`🔴 P0`, `🟡 P1`), tag support, progress tracking `--status` | Medium |
| `critique` | `--quick` mode, `--cross-model` docs, score history in `vault/critique_history.jsonl` | Medium |
| `project_dossier` | Tech-stack auto-detect, `--quick` mode, diff since last dossier | Low |
| `find_bug` | Generalize examples, `--scope <dir>`, auto-route bugs to backlog | Low |
| `llm_stats` | Model-aware pricing, `--chart` visualization, context window % warning | Medium |
| `universal_llm_client` | Add `stream_chat()`, newer providers (Groq, DeepSeek), `test_connection()` | Low |
| `update_global_docs` | Configurable paths, dedup check, tag entries | Low |
| `recipe_card` | Theme presets, `--from-url` extraction | Low |

### 🆕 Phase 5: New Skills to Build

| Skill | Description | Priority | Dependencies |
|:------|:------------|:---------|:-------------|
| `session_summary` | Auto-generate end-of-session summary (decisions, files changed, next steps) → daily log | **High** | Pairs with `update_global_docs` |
| `pr_review` | Structured PR review: diff analysis, security scan, test coverage, conventional commits | **High** | Industry standard |
| `test_gen` | Generate unit/integration tests per module. Auto-detect framework (pytest/vitest/jest) | **High** | Supports critique Dim 3 |
| `env_doctor` | Health-check dev environment: env vars, versions, `llm_config.yaml`, API connectivity | **Medium** | Supports `universal_llm_client` |
| `refactor` | Structured refactoring: code smells → extractions/renames → before/after verification | **Medium** | Complements `find_bug` |
| `cost_tracker` | Longitudinal LLM cost tracking across sessions/models/projects. Budget alerts. | **Medium** | Extends `llm_stats` |
| `dependency_audit` | Scan for outdated/vulnerable deps. Generate upgrade plan. | **Low** | Security hygiene |
| `daily_standup` | Auto-generate standup from git commits + open PRs + backlog status | **Low** | Pairs with `changelog` |
| `prompt_lab` | Interactive prompt engineering: multi-model comparison, version tracking, scoring | **Low** | Uses `universal_llm_client` |
| `diagram_gen` | Generate Mermaid diagrams from code analysis (imports, class hierarchies) | **Low** | Enhances `project_dossier` |

---

## 4. System Skills & Commands — What to Leverage (Not Reinvent)

| System Feature | Use Instead Of | Integration Opportunity |
|:---------------|:---------------|:------------------------|
| `/learn` | `manage_ki` | Deprecate `manage_ki`; use `/learn` for behavioral persistence |
| `/goal` | `capture-goal` autonomous mode | Merge into `capture_qa --autonomous` which internally leverages `/goal` patterns |
| `/plan` | Manual planning in `feature_spec` | `feature_spec` can recommend `/plan` as a pre-step |
| `/grill-me` | Interview step in `feature_spec` | `feature_spec` interview step adds spec-driven structure beyond generic `/grill-me` |
| `/boost` | Deep analysis in `critique` | Recommend `/boost` before running `critique` on complex systems |
| `/schedule` + `automation` | Manual cron setup | Use for `daily_standup` and `session_summary` scheduled runs |
| `migrate-workflows` | Manual workflow cleanup | Use to automate Phase 1 & Phase 3 cleanup |
| `generative_ui` | Markdown-only reports | Enhance `llm_stats --chart` and `manage_backlog --gantt` with interactive widgets |
| `run-workflow` | Sequential subagent calls | Use for `pr_review` and `test_gen` fan-out across multiple files |

---

## 5. Final Skill Registry (Post-Consolidation Target)

### Global Skills (`enabled/` → `~/.gemini/config/skills/`)

| # | Skill | Status Post-Consolidation |
|:--|:------|:--------------------------|
| 1 | `changelog` | ✅ Enhanced (conventional commits) |
| 2 | `feature_spec` | ✅ Fixed (`ask_question`) |
| 3 | `llm_stats` | ✅ Fixed (paths + pricing) |
| 4 | `manage_backlog` | ✅ Enhanced (priorities + tags) |
| 5 | `universal_llm_client` | ✅ Enhanced (streaming + providers) |
| 6 | `update_global_docs` | ✅ Enhanced (configurable paths) |
| 7 | `capture_qa` | 🆕 Promoted from workflow |
| 8 | `list_hf_models` | 🆕 Promoted from workflow |
| 9 | `session_summary` | 🆕 New |
| 10 | `pr_review` | 🆕 New |
| 11 | `test_gen` | 🆕 New |
| 12 | `env_doctor` | 🆕 New |
| ~~13~~ | ~~`manage_ki`~~ | 🚫 Deprecated → `disabled/` |
| ~~14~~ | ~~`llm_judge`~~ | 🚫 Deleted |

### Workspace Skills (`.agents/skills/`)

| # | Skill | Status Post-Consolidation |
|:--|:------|:--------------------------|
| 1 | `critique` | ✅ Enhanced (quick mode + history) |
| 2 | `find_bug` | ✅ Enhanced (generalized + scoped) |
| 3 | `project_dossier` | ✅ Enhanced (auto-detect + diff) |
| 4 | `recipe_card` | ✅ Enhanced (themes) |
| 5 | `refactor` | 🆕 New |
| 6 | `cost_tracker` | 🆕 New |
