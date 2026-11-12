# Global Backlog

- [ ] Implement a mechanism to count total LLM API calls, specifically tracking major workflows (e.g., a main call followed by multiple helper/sub-calls) within LawNidhi.
- [ ] Tambola game - build a web based tambola game using html, css, js
- [ ] User registration module
- [ ] Payment Gateway integration for paid actions in LawNidhi.
- [ ] payment module 
- [ ] Implement a Prompt-RAG retrieval system for dynamic injection of system guidelines and SOPs at scale across agents.
- [ ] Agentic RAG & Dynamic Prompt Retrieval Architecture
- [x] **[2026-10-30]** Agent Memory Systems & State Persistence (Episodic/Procedural Memory & Crash Recovery) - Reference implementation delivered in `projects/hitl_email_agent` using `SqliteSaver`, `RecipientMemoryStore` (WAL), and Dual-Transcript Event Logging.
- [x] **[2026-11-12]** Universal Remote Notification Gateway & Asynchronous HITL Bridge (BL-002) - Reference implementation delivered in `projects/Agents/core/notifications.py` (Telegram, Webhook, Console, Mock) and `projects/hitl_email_agent` (`scripts/approval_server.py`).
- [ ] Self-Hosted n8n Webhook Bridge & Multi-Channel Fanout Template: Declarative n8n workflow forwarding agent review requests to WhatsApp (Meta Cloud API), Discord, and Slack, routing button callbacks back to `scripts/approval_server.py`.
- [ ] Web/Mobile Companion Review UI: Lightweight dashboard consuming `GET /pending` and `POST /api/callback` for human operators reviewing paused agent workflows without CLI access.
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
