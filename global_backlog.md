# Global Backlog

- [ ] **[2026-11-19] Antigravity Skills & Commands Consolidation**: Cleanup deprecated skills (`manage_ki`, `llm_judge`), fix broken paths in `sync.sh`/`llm_stats`/`manage_ki`, promote `capture-qa` and `list-hf-models` workflows to proper skills, build new `session_summary`/`pr_review`/`test_gen` skills. Full plan: [skills_and_commands_consolidation.md](file:///home/navin/work/AI/docs/skills_and_commands_consolidation.md) | Backlog: [antigravity-skills/specs/backlog.md](file:///home/navin/work/AI/tooling/antigravity-skills/specs/backlog.md)

- [ ] Implement a mechanism to count total LLM API calls, specifically tracking major workflows (e.g., a main call followed by multiple helper/sub-calls) within LawNidhi.
- [ ] Tambola game - build a web based tambola game using html, css, js
- [ ] User registration module
- [ ] Payment Gateway integration for paid actions in LawNidhi.
- [ ] payment module 
- [ ] Implement a Prompt-RAG retrieval system for dynamic injection of system guidelines and SOPs at scale across agents.
- [ ] Agentic RAG & Dynamic Prompt Retrieval Architecture
- [x] **[2026-10-30]** Agent Memory Systems & State Persistence (Episodic/Procedural Memory & Crash Recovery) - Reference implementation delivered in `projects/hitl_email_agent` using `SqliteSaver`, `RecipientMemoryStore` (WAL), and Dual-Transcript Event Logging.
- [x] **[2026-11-12]** Universal Remote Notification Gateway & Asynchronous HITL Bridge (BL-002) - Reference implementation delivered in `projects/Agents/core/notifications.py` (Telegram, Webhook, Console, Mock) and `projects/hitl_email_agent` (`scripts/approval_server.py`).
- [x] **[2026-11-23]** Dynamic Skill Packages & Cascading Hybrid Router - Reference implementation delivered in `projects/Agents/core/skills.py` with YAML frontmatter discovery, Tier 1 deterministic trigger fast-path, Tier 2 LLM fallback, and reference `skills/data_analyst/` package.
- [x] **[2026-11-23]** OutputGuard & Payload Offloader (Data Pointer Pattern) - Reference implementation delivered in `projects/Agents/core/output_guard.py` & `projects/Agents/core/pointer_tools.py` with SHA-256 CAS, smart head/tail truncation, and selective micro-query tools (`read_pointer_slice`, `query_pointer_json`, `grep_pointer`).
- [x] **[2026-11-23]** Sub-Agent Skill Execution Engine & Context Sandboxing - Reference implementation delivered in `projects/Agents/core/subagent.py` supporting isolated child ReAct loops, parent dispatcher tool bridging (`delegate_to_<skill>`), bounded loop ceilings (Rule 1), timeout protection (Rule 4), and non-consecutive repetition detection.
- [ ] 🔴 P0 **Robust Sandboxing (Docker / Firecracker microVM)**: Transition from local `subprocess` execution in `PythonExecutor` to containerized or microVM sandboxing for safe code execution in untrusted environments.
- [ ] 🔴 P0 **CheckpointManager / StateSerializer**: Serialize and hydrate agent loop state (messages, active SOP, tool outputs) to SQLite/JSON for cross-session crash recovery and checkpoint resumption.
- [ ] Self-Hosted n8n Webhook Bridge & Multi-Channel Fanout Template: Declarative n8n workflow forwarding agent review requests to WhatsApp (Meta Cloud API), Discord, and Slack, routing button callbacks back to `scripts/approval_server.py`.
- [ ] Web/Mobile Companion Review UI: Lightweight dashboard consuming `GET /pending` and `POST /api/callback` for human operators reviewing paused agent workflows without CLI access.
- [ ] **[BL-018 / HITL]** Inbound Endpoint Authentication & Webhook Security Hardening: Protect FastAPI approval server endpoints with API key headers and Telegram webhook secret tokens (A MUST for production deployment; deferred in local/dev mode).
- [ ] Agentic Evaluation & Trajectory Evals Frameworks
- [ ] **[2026-10-07]** Automated RAG Evaluation & CI/CD Benchmarking Harness: Integrate **Ragas** (evaluating RAG Triad: Faithfulness, Answer Relevance, Context Precision, and Context Recall) with synthetic test-set generation for regression testing across retrieval pipelines (LawNidhi / Enterprise RAG).
- [ ] Knowledge Graph & GraphRAG Integration: Build embedded property graph (Kùzu/NetworkX) pipelines for multi-hop legal reasoning in LawNidhi and hierarchical community summaries in Open-NotebookLM.
- [ ] Long-Term Semantic Memory Graph for Autonomous Agents (Entity/Fact persistence across ReAct loops).



# my tasks

- [] converting quantization models from bitsandbytes to exllama
- [] implement a service for exllm model inference 
- [] implement a monitoring dashboard for exllm model inference
- [ ] Implement SGLang-based high-performance model serving daemon & engine with RadixAttention KV reuse (`LocalModelsProject`)
- [ ] Implement automated Ragas evaluation script & synthetic test-data generation harness for RAG pipelines
- [ ] **[Logits Tweaking / Constrained Decoding Spike]**: Implement a lightweight standalone test proof-of-concept for custom token logit masking / logit processor (FSM-guided next-token logit biasing to $-\infty$) using PyTorch / HuggingFace `LogitsProcessor` or Outlines/XGrammar for deterministic output generation and static token fast-forwarding.
