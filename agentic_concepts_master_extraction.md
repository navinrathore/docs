# Agentic AI Concepts — Master Extraction & Gap Analysis (Phase 1)

> **Purpose**: Concept-centric knowledge map extracted from DeepLearning.AI intermediate short courses.
> Maps each concept to your implementation status across workspace projects, identifies gaps,
> rates importance/complexity, and flags specific video segments worth watching.
>
> **Scope**: Phase 1 — Agentic AI Core, Protocols, Tool Use, Safety & Evaluation only.
> Phase 2 (Fine-Tuning, Quantization, Inference, Multimodal) deferred.

---

## Status Legend

| Status | Meaning |
|--------|---------|
| ✅ Implemented | Working code exists and is tested in at least one project |
| 🟡 Partial | Code exists but incomplete, untested, or only documented (not coded) |
| ❌ Gap | Concept not implemented anywhere in the workspace |
| 📄 Documented | Covered in `agentic_blueprints` docs or `docs/` guides but no code |

---

## Theme 1: Agent Architecture & Design Patterns

### 1.1 ReAct Loop (Reasoning + Acting)
- **Concept**: Agent iterates through Observe → Think → Act → Observe cycles with tool calling until a task is complete or a hard loop limit is hit.
- **Source Courses**: _Agentic AI (Andrew Ng)_, _AI Agents in LangGraph_
- **Status**: ✅ Implemented
- **Where**: `Agents/data_analyst/agent.py` — full ReAct loop with `max_loops=10`, `RepetitionDetector`, tool dispatch, and `ExecutionTimer`.
- **Gaps**: Loop counter is flat — no exponential backoff or dynamic loop budget based on task complexity. No trajectory scoring to decide "should I keep looping?"
- **Importance**: 🔴 Critical | **Complexity**: ⭐⭐
- **Video**: Skip — you have this down. Only revisit _Agentic AI_ Lesson 1 if you want Andrew Ng's mental model framing for documentation.

### 1.2 Workflow Topologies (Chaining, Routing, Parallelization, Orchestrator-Worker, Evaluator-Optimizer)
- **Concept**: The 5 canonical patterns for structuring multi-step agent workflows. Each has distinct latency/cost/complexity profiles.
- **Source Courses**: _Agentic AI (Andrew Ng)_, _Multi AI Agent Systems (CrewAI)_
- **Status**: ✅ Implemented + 📄 Documented
- **Where**:
  - Documented: `agentic_blueprints/docs/02_architectural_patterns/01_workflow_topologies.md` — all 5 topologies with mermaid diagrams, trade-off tables, and adoption triggers.
  - Implemented: Prompt Chaining in `hitl_email_agent` (research → draft → critique → review → dispatch). Routing in `Agents/data_analyst/sops.py` (SOP router). Evaluator-Optimizer in `hitl_email_agent` (critic loop with `max_critique_loops`).
- **Gaps**: **Parallelization** (map-reduce) and **Orchestrator-Worker** (dynamic sub-task fan-out) are documented but not implemented in any project. No working example of parallel agent execution.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Multi AI Agent Systems (CrewAI)_ **Lesson 5-6** (hierarchical and parallel process orchestration) for the parallelization pattern specifically.

### 1.3 Supervisor / Hierarchical Orchestration
- **Concept**: A supervisor agent that dynamically routes tasks to specialized worker agents, evaluates their outputs, and decides next steps. Central command-and-control pattern.
- **Source Courses**: _Multi AI Agent Systems (CrewAI)_, _AI Agents in LangGraph_
- **Status**: 🟡 Partial
- **Where**: `hitl_email_agent/graph.py` has a `supervisor_node` that routes between research, drafting, critique, and review — but it's a fixed pipeline, not truly dynamic task allocation. Tests in `test_supervisor_hierarchy.py` validate the routing logic.
- **Gaps**: Supervisor is hard-coded to a single email workflow. Not generalized for arbitrary task decomposition. No ability for the supervisor to spawn sub-agents on-the-fly or re-assign failed tasks.
- **Importance**: 🔴 Critical | **Complexity**: ⭐⭐⭐
- **Video**: Watch _AI Agents in LangGraph_ **Lesson 5** (Human-in-the-Loop) + _CrewAI_ **Lesson 4** (Hierarchical Process with Manager Agent).

### 1.4 Role-Based Multi-Agent Systems
- **Concept**: Assigning each agent a distinct role, goal, backstory, and tool set. Agents cooperate through shared state or message passing to solve complex tasks.
- **Source Courses**: _Multi AI Agent Systems (CrewAI)_
- **Status**: 📄 Documented
- **Where**: `agentic_blueprints/docs/06_frontiers_catalog/01_tiered_readiness_matrix.md` — Multi-Agent Debate pattern documented. `Agents/docs/agentic_design_patterns.md` covers the concept.
- **Gaps**: No working multi-agent system with distinct roles. The `hitl_email_agent` has specialized nodes (researcher, drafter, critic) but they're functions within one graph, not independent agents with their own prompts/tools/memory.
- **Importance**: 🔴 Critical | **Complexity**: ⭐⭐⭐
- **Video**: Watch _CrewAI_ **Lessons 2-3** (defining agents with roles/goals/backstories, task dependencies). Also **Lesson 7** (real-world automation use cases).

### 1.5 Evaluator-Optimizer / Critic Loop
- **Concept**: An inner loop where a "critic" agent evaluates a "generator" agent's output against quality criteria, with iterative refinement until a quality threshold is met or a loop budget is exhausted.
- **Source Courses**: _Agentic AI (Andrew Ng)_
- **Status**: ✅ Implemented
- **Where**: `hitl_email_agent/graph.py` — `critic_node` evaluates drafts via `CriticEvaluationReport`, routes back to `draft_node` for revision if quality fails, bounded by `max_critique_loops=2`.
- **Gaps**: Critic evaluation is LLM-only — no deterministic scoring rubric or automated metrics (like BLEU, ROUGE, or custom domain checks) alongside the LLM judge. No "confidence threshold" that auto-approves high-quality drafts without human review.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Agentic AI_ **Lesson 2** (Reflection pattern) for the theoretical framing.

### 1.6 Reflection Pattern
- **Concept**: An agent reviews its own output, identifies flaws, and self-corrects — either via self-prompting or by routing through a separate critic prompt. Distinct from the evaluator-optimizer in that it's self-directed rather than externally judged.
- **Source Courses**: _Agentic AI (Andrew Ng)_
- **Status**: 🟡 Partial
- **Where**: The critic loop in `hitl_email_agent` is external reflection (separate critic prompt). No self-reflection where the same agent re-reads and revises its own output in a single turn.
- **Gaps**: No implementation of single-agent self-reflection (e.g., "Review your draft for factual errors and rewrite"). This is a cheap, high-impact pattern that doesn't require multi-agent infrastructure.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐
- **Video**: Watch _Agentic AI_ **Lesson 2** (Reflection) — 15 minutes, specifically the self-critique prompting technique.

---

## Theme 2: State Management, Persistence & Human-in-the-Loop

### 2.1 LangGraph StateGraph & State Schema
- **Concept**: Modeling agent workflows as directed graphs with typed state. Nodes are pure functions `(state) → partial_state_update`. Edges can be conditional. State is a `TypedDict` or Pydantic model with reducer annotations.
- **Source Courses**: _AI Agents in LangGraph_
- **Status**: ✅ Implemented
- **Where**: `hitl_email_agent/graph.py` — `EmailAgentState(TypedDict)` with 15+ fields, 6 nodes, conditional routing via `route_after_review`, compiled with `StateGraph`.
- **Gaps**: State reducers (e.g., `Annotated[list, operator.add]`) not used — all state updates are full replacements. No sub-graph composition (nested StateGraphs).
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐
- **Video**: Watch _AI Agents in LangGraph_ **Lesson 2** (Building an Agent from Scratch) and **Lesson 3** (LangGraph Components) for the state reducer patterns.

### 2.2 Checkpointing & State Persistence (SqliteSaver)
- **Concept**: Serializing full graph state to disk at every node transition, enabling pause/resume, crash recovery, and time-travel debugging.
- **Source Courses**: _AI Agents in LangGraph_
- **Status**: ✅ Implemented
- **Where**: `hitl_email_agent/graph.py` — `SqliteSaver` with `JsonPlusSerializer`, custom allowed msgpack modules for Pydantic model serialization. `checkpoints.db` persists state across sessions.
- **Gaps**: None significant. Working and tested. Could explore PostgresSaver for multi-process deployments but that's production scaling, not a concept gap.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐
- **Video**: Skip — you've implemented this comprehensively.

### 2.3 Human-in-the-Loop (Interrupt-Save-Notify-Resume)
- **Concept**: Agent autonomously works until it reaches a high-stakes decision boundary, then explicitly interrupts, persists state, notifies a human, and waits for approval/edit/rejection before resuming.
- **Source Courses**: _AI Agents in LangGraph_, _Building Adaptive AI Agents_
- **Status**: ✅ Implemented
- **Where**: `hitl_email_agent` — the flagship pattern. `interrupt()` in `human_review_node`, `Command(resume=payload)` in controller, `HumanReviewPayload` with approve/edit/reject decisions, `NotificationGateway` for multi-channel notification (Telegram, Webhook, Console, Mock).
- **Gaps**: HITL is binary (single interrupt point). No concept of **tiered approval thresholds** (e.g., auto-approve low-risk, interrupt for medium-risk, escalate for high-risk based on confidence score). The confidence_score field exists on `EmailDraft` but isn't used for routing.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐
- **Video**: Watch _AI Agents in LangGraph_ **Lesson 5** (Human-in-the-Loop) for the tiered threshold concept only — 10 minutes.

### 2.4 Asynchronous Approval Webhooks & Callback Security
- **Concept**: External webhook endpoints that receive human decisions (approve/reject) asynchronously, allowing agents to resume from any device. Includes callback data integrity, two-way UI acknowledgment, and safe callback data sizing.
- **Source Courses**: _Building Adaptive AI Agents_, _Event-Driven Agentic Document Workflows_
- **Status**: ✅ Implemented
- **Where**: `hitl_email_agent/scripts/approval_server.py` — FastAPI server with `/api/callback` endpoint. `Agents/core/notifications.py` — `TelegramChannel` with `answerCallbackQuery` acknowledgment, SHA-256 callback data hashing for byte-limit compliance.
- **Gaps**: No API key / webhook secret token authentication on inbound endpoints (tracked as `BL-018` in backlog). No rate limiting.
- **Importance**: 🟡 Medium | **Complexity**: ⭐⭐⭐
- **Video**: Skip — you've built this. The security hardening is a backlog item, not a concept gap.

---

## Theme 3: Agent Memory Systems

### 3.1 Three-Tier Memory Architecture (Working / Episodic / Semantic)
- **Concept**: Structured memory with distinct layers — L1 Working Memory (scratchpad, checklist), L2 Episodic Buffer (sliding-window conversation history with micro-summarization), L3 Semantic Store (persistent facts and entity registry).
- **Source Courses**: _Agent Memory (Oracle)_, _Long-Term Agentic Memory with LangGraph_
- **Status**: ✅ Implemented
- **Where**: `Agents/core/memory.py` — `AgentMemoryManager` with `WorkingMemory`, `EpisodicBuffer`, and `SemanticStore`. Includes `export_state()` / `import_state()` for checkpoint serialization.
- **Gaps**: 
  - L3 Semantic Store is an in-memory dict — not persisted to disk or vector DB. Loses all learned facts on restart.
  - No **embedding-based retrieval** for semantic memory (just key-value lookup).
  - No **memory consolidation** (promoting episodic memories to semantic facts after repeated patterns).
- **Importance**: 🔴 Critical | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Agent Memory (Oracle)_ **Lessons 3-4** (memory persistence, long-timeline retention). Watch _Long-Term Agentic Memory with LangGraph_ **Lessons 2-3** (LangMem integration, memory types).

### 3.2 Recipient-Specific Long-Term Memory
- **Concept**: Per-entity persistent memory that tracks interaction history, preferences, objections, and tone across multiple sessions. Enables personalized agent behavior.
- **Source Courses**: _Agent Memory (Oracle)_, _Long-Term Agentic Memory with LangGraph_
- **Status**: ✅ Implemented
- **Where**: `hitl_email_agent/memory.py` — `RecipientMemoryStore` with SQLite (WAL mode), tracking recipient profiles, interaction counts, preferred tone, and historical objections.
- **Gaps**: Memory retrieval is exact-match by recipient email. No fuzzy matching, no semantic similarity search across memory entries. No decay or relevance scoring (old memories weighted equally to recent ones).
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Long-Term Agentic Memory with LangGraph_ **Lesson 4** (memory scoring and relevance decay).

### 3.3 Memory Engineering (Beyond Simple Context)
- **Concept**: Transitioning from "stuff everything into the context window" to engineered memory systems that actively manage what the agent remembers, forgets, and retrieves. Includes memory compaction, selective recall, and cross-session learning.
- **Source Courses**: _Agent Memory (Oracle)_, _Context Engineering_
- **Status**: 🟡 Partial
- **Where**: `Agents/core/memory.py` has basic eviction (sliding window) and micro-summarization in `EpisodicBuffer`. `hitl_email_agent` has recipient-specific memory persistence.
- **Gaps**:
  - No **active forgetting** (deleting or deprioritizing stale/contradictory memories).
  - No **memory consolidation pipeline** (batch processing episodic memories into durable semantic facts).
  - No **retrieval-augmented memory** (embedding-based recall of relevant past interactions).
  - No **cross-agent memory sharing** (one agent learning from another's experiences).
- **Importance**: 🔴 Critical | **Complexity**: ⭐⭐⭐⭐
- **Video**: Watch _Agent Memory (Oracle)_ **Full course** (1 hour — the entire thing is memory engineering). Also _Context Engineering_ **Lessons 2-3** (context rot mitigation).

---

## Theme 4: Protocols & Interoperability

### 4.1 Model Context Protocol (MCP) — Server Side
- **Concept**: Exposing deterministic tools, resources, and prompts via the MCP standard. FastMCP server with typed tool definitions, field descriptions, and resource URIs.
- **Source Courses**: _MCP: Build Rich-Context AI Apps (Anthropic)_, _Build AI Apps with MCP Servers (Box)_
- **Status**: ✅ Implemented
- **Where**: `hitl_email_agent/mcp_email_server.py` — full FastMCP server with `send_email`, `get_outbox`, `verify_email_health` tools. Declarative registry in `mcp_servers.yaml`.
- **Gaps**: Only one MCP server implemented (email). No resource endpoints (MCP `resource://` URIs for data access). No prompt templates exposed via MCP.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐
- **Video**: Watch _MCP (Anthropic)_ **Lessons 3-4** (building MCP resources and prompt templates, MCP Inspector).

### 4.2 Model Context Protocol (MCP) — Client Side
- **Concept**: Connecting to MCP servers, discovering available tools, calling tools through the protocol, managing server lifecycle (subprocess spawning, timeout, graceful shutdown).
- **Source Courses**: _MCP: Build Rich-Context AI Apps (Anthropic)_
- **Status**: ✅ Implemented
- **Where**: `Agents/core/mcp/client.py` — `MCPServerConnection` and `MCPClientManager` with async stdio_client, session initialization, tool listing, tool calling, and graceful teardown via `AsyncExitStack`.
- **Gaps**: No SSE transport (only stdio). No MCP server health monitoring or auto-reconnection on failure.
- **Importance**: 🟡 Medium | **Complexity**: ⭐⭐⭐
- **Video**: Skip for now — your stdio implementation is solid. SSE transport is a stretch goal.

### 4.3 Agent-to-Agent Protocol (A2A)
- **Concept**: Google Cloud's open protocol for agents to discover and communicate with each other across different frameworks. A2A SDK wraps agents in discoverable servers. Handles sequential, hierarchical, and parallel cross-framework orchestration.
- **Source Courses**: _A2A: The Agent2Agent Protocol (Google Cloud / IBM)_
- **Status**: ❌ Gap
- **Where**: Not implemented or documented anywhere.
- **Gaps**: Complete gap. No A2A server wrapping, no agent discovery, no cross-framework communication. This is the "horizontal" complement to MCP's "vertical" tool integration.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐
- **Video**: Watch _A2A: The Agent2Agent Protocol_ **Full course** (1 hour — the protocol is new and the entire course is relevant). Focus on **Lessons 2-3** (wrapping agents in A2A servers, agent discovery).

### 4.4 Agent Skills (Structured Markdown Skill Packs)
- **Concept**: Packaging specialized agent behaviors as structured markdown files/folders that the agent can dynamically load to handle specific workflow types (research, code review, data analysis).
- **Source Courses**: _Agent Skills with Anthropic_
- **Status**: ✅ Implemented
- **Where**: `Agents/data_analyst/sops/` — declarative SOP markdown files (`data_cleaning.md`, `visualization.md`, `default_analyst.md`). `sops.py` dynamically routes to the correct SOP based on user intent. Also documented in `agentic_blueprints` as Pattern 1.1 (Dynamic SOP Ingestion).
- **Gaps**: SOP routing is keyword-based, not semantic. No embedding-based SOP selection. No composable skill stacking (using multiple SOPs simultaneously).
- **Importance**: 🟡 Medium | **Complexity**: ⭐⭐
- **Video**: Watch _Agent Skills (Anthropic)_ **Lesson 2** (skill structure conventions) if you want to align your SOP format with Anthropic's conventions.

---

## Theme 5: Tool Use, Function Calling & Structured Output

### 5.1 Function Calling (Tool Definitions & Dispatch)
- **Concept**: Defining tools as JSON schemas with typed parameters, letting the LLM decide when and how to call them, parsing tool call responses, and dispatching execution. Includes parallel and nested function calls.
- **Source Courses**: _Function-Calling and Data Extraction with LLMs (Nexusflow)_
- **Status**: ✅ Implemented
- **Where**: `Agents/data_analyst/tools.py` — `get_anthropic_tools()` and `get_openai_tools()` with JSON schema definitions. `Agents/core/dispatcher.py` — `AsyncToolDispatcher` with concurrent execution, semaphore throttling, and MCP tool bridging. `agent.py` dispatches tool calls in the ReAct loop. `Agents/core/mcp/adapters.py` converts MCP tool schemas to Anthropic/OpenAI formats.
- **Where**: `Agents/data_analyst/tools.py` — `get_anthropic_tools()` and `get_openai_tools()` with JSON schema definitions. `Agents/core/dispatcher.py` — `AsyncToolDispatcher` with concurrent execution, semaphore throttling, and MCP tool bridging. `Agents/core/openapi_loader.py` — `OpenAPIToolLoader` with multi-strategy tool generation (direct vs generic runner vs read-only filtering) and dry-run execution. `agent.py` dispatches tool calls in the ReAct loop. `Agents/core/mcp/adapters.py` converts MCP tool schemas to Anthropic/OpenAI formats.
- **Gaps**: 
  - ~~No parallel tool calling~~: ✅ **Implemented** via `core/dispatcher.py` (`AsyncToolDispatcher`) with bounded semaphore concurrency (`max_concurrency`), configurable sequential/parallel execution toggle, per-tool timeout/fault-shielding, and native MCP async tool bridging.
  - ~~No OpenAPI spec integration~~: ✅ **Implemented** via `core/openapi_loader.py` (`OpenAPIToolLoader`) with configurable direct 1:1, generic meta-runner (`call_api`), tag filtering, and read-only safety guardrails.
  - No **nested/chained tool calls** (output of one tool fed as input to another within a single turn).
  - No **OpenAPI spec integration** (auto-generating tool schemas from API specifications) — 🟡 *Tracked in Agents backlog*.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Function-Calling & Data Extraction_ **Lessons 4-5** (parallel function calls, OpenAPI spec integration).

### 5.2 Pydantic Schema Enforcement for LLM Outputs
- **Concept**: Using Pydantic models to validate, coerce, and constrain LLM outputs at the API boundary. Ensures structured, deterministic outputs for downstream processing.
- **Source Courses**: _Pydantic for LLM Workflows_, _Getting Structured LLM Output_
- **Status**: ✅ Implemented
- **Where**: `projects/Agents/core/llm_clients.py` — Configurable tri-mode structured output engine (`auto`, `native`, `prompt_injection`):
  - `OpenAILLMClient` & `HuggingFaceClient`: Provider-native constrained decoding via `response_format={"type": "json_schema", "json_schema": {"strict": True, ...}}` with recursive schema sanitization (`prepare_strict_json_schema`).
  - `AnthropicClient`: Provider-native forced tool calling via `tools` + `tool_choice={"type": "tool", "name": ...}`.
  - `MockLLMClient`: Hermetic zero-token offline validation and fallback testing.
  - `hitl_email_agent/schemas.py`: Pydantic v2 schemas (`EmailDraft`, `HumanReviewPayload`, `CriticEvaluationReport`).
- **Gaps**: None. Fully implemented with seamless `auto` fallback to prompt-injection + repair if endpoints reject native grammar constraints.
- **Importance**: 🔴 Critical | **Complexity**: ⭐⭐
- **Video**: _Pydantic for LLM Workflows_ **Lessons 3-4** (good theoretical reinforcement).

### 5.3 Natural Language to SQL / Database Agents
- **Concept**: Agents that translate natural language queries into SQL, execute against databases, and interpret results. Includes schema introspection, query validation, and iterative SQL refinement.
- **Source Courses**: _Building Your Own Database Agent (Microsoft)_
- **Status**: 🟡 Partial
- **Where**: `NyayaGraph` has SQLite databases (`lawnidhi.db`, `nyayagraph.db`), but NL-to-SQL agent is not implemented. The data_analyst agent in `Agents/` executes Python code (not SQL) against data.
- **Gaps**: No NL-to-SQL capability. No database schema introspection tool. No SQL validation or injection prevention layer.
- **Importance**: 🟡 Medium | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Building Your Own Database Agent_ **Lessons 2-4** (schema introspection, query generation, function calling over tabular data).

---

## Theme 6: Context Engineering & Optimization

### 6.1 Context Window Lifecycle Management
- **Concept**: Actively managing what goes into the LLM context window — pruning, compaction, selective injection — to prevent context rot (degraded performance as context fills with stale/contradictory information).
- **Source Courses**: _Context Engineering_
- **Status**: 🟡 Partial
- **Where**: `Agents/core/llm_clients.py` has context pruning (summarizing older turns when token thresholds are reached). `agentic_blueprints/docs/02_architectural_patterns/03_context_window_lifecycle.md` documents the lifecycle. `llmops_core/docs/06_context_engineering.md` covers theory.
- **Gaps**: 
  - Pruning is a simple sliding window — no **importance-based retention** (keeping high-value turns regardless of age).
  - No **structured context slots** (reserving fixed context budget for system prompt, tools, memory, and conversation separately).
  - No **context budget monitoring** at runtime (tracking how full the window is and alerting before degradation).
- **Importance**: 🔴 Critical | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Context Engineering_ **Full course** (1 hour — this is the emerging discipline that underpins all agent performance). Focus on **Lessons 2-4** (context rot diagnosis, structured context slots, trace-based context formatting).

### 6.2 Semantic Caching
- **Concept**: Caching LLM responses keyed by semantic similarity of the query (not exact string match). Uses embeddings and cross-encoders to detect when different queries have the same intent, returning cached responses to skip redundant API calls.
- **Source Courses**: _Semantic Caching for AI Agents_
- **Status**: 🟡 Partial
- **Where**: `llmops_core/packages/llmops_core/cost_ops/semantic_cache.py` exists but needs verification of implementation completeness.
- **Gaps**: Need to verify if the implementation includes embedding-based similarity matching, hit-rate metrics, and integration with the agent loop. Likely scaffolded but not production-tested.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Semantic Caching for AI Agents_ **Lessons 2-4** (building the cache with Redis, embedding similarity thresholds, precision tracking).

### 6.3 Prompt Caching (KV Cache Reuse)
- **Concept**: Leveraging provider-level prompt caching (Anthropic, OpenAI) to cache static system prompts and large reference documents, avoiding redundant KV computation on repeated calls. Cuts input token costs by 80-90%.
- **Source Courses**: _Building Toward Computer Use with Anthropic_, _Efficient Inference with SGLang_
- **Status**: 📄 Documented
- **Where**: Documented in `agentic_blueprints/docs/05_guardrails_and_operations/02_token_economics_and_budgeting.md`. Concept covered in AGENTS.md Rule 6 (Prompt Caching Friendliness). Not implemented in code.
- **Gaps**: No code uses `cache_control` headers (Anthropic) or `cached` prefix blocks. The architectural principle is known but no agent actually enables it.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐
- **Video**: Watch _Building Toward Computer Use (Anthropic)_ **Lesson on Prompt Caching** (the specific segment on `cache_control` block placement and ephemeral vs. persistent caching).

---

## Theme 7: Safety, Guardrails & Evaluation

### 7.1 Input/Output Guardrails
- **Concept**: Wrapping LLM inputs and outputs with validation layers that detect and block hallucinations, off-topic responses, PII leakage, and harmful content before they reach the user or downstream systems.
- **Source Courses**: _Safe and Reliable AI via Guardrails (Guardrails AI)_
- **Status**: 🟡 Partial
- **Where**: `llmops_core/packages/llmops_core/guardrails/guardrails.py` exists (needs depth check). `agentic_blueprints/docs/05_guardrails_and_operations/03_security_and_adversarial_hardening.md` has comprehensive threat modeling and defensive patterns. AGENTS.md enforces loop limits as a guardrail.
- **Gaps**: 
  - No runtime PII detection/masking on LLM outputs.
  - No toxicity/harm classifier in the pipeline.
  - No `guardrails-ai` library integration (the actual Python package with validators).
  - Guardrails are conceptual/defensive-coding, not a pluggable middleware layer.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐
- **Video**: Watch _Safe and Reliable AI via Guardrails_ **Lessons 3-5** (implementing input/output guards, PII detection, hallucination checks in a RAG chatbot context).

### 7.2 Prompt Injection & Indirect Context Poisoning Defense
- **Concept**: Defending against adversarial inputs that attempt to override system instructions — either directly (jailbreaks) or indirectly (through poisoned documents, emails, or web pages ingested by the agent).
- **Source Courses**: _Red Teaming LLM Applications (Giskard)_
- **Status**: 📄 Documented
- **Where**: `agentic_blueprints/docs/05_guardrails_and_operations/03_security_and_adversarial_hardening.md` — full threat taxonomy (direct injection, indirect injection, tool abuse, privilege escalation) with defensive patterns (structural role isolation, boundary anchoring, untrusted data tagging, Pydantic tool validation).
- **Gaps**: Documentation is excellent but no automated testing for injection resistance. No red-teaming test suite. No Giskard integration for automated vulnerability scanning.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Red Teaming LLM Applications_ **Lessons 3-5** (automated red-teaming with Giskard, building a red-team test suite, end-to-end assessment).

### 7.3 LLM Evaluation & Judge Patterns
- **Concept**: Using LLMs as evaluators/judges to score agent outputs on dimensions like correctness, groundedness, relevance, and safety. Includes single-judge, multi-judge, and reference-based evaluation.
- **Source Courses**: _Building and Evaluating Advanced RAG (TruEra)_, _Automated Testing for LLMOps_
- **Status**: 🟡 Partial
- **Where**: `llmops_core/packages/llmops_core/eval/judge.py` exists. `agentic_blueprints/docs/05_guardrails_and_operations/01_testing_and_evaluation.md` documents evaluation patterns. The workspace has a `critique` skill with 5-dimension rubric.
- **Gaps**: 
  - No **RAG Triad evaluation** (Faithfulness, Answer Relevance, Context Precision/Recall) implemented.
  - No **Ragas** library integration (tracked in global backlog).
  - No automated evaluation CI/CD pipeline (also tracked in backlog).
  - LLM-as-judge exists in `llmops_core` but not wired into any agent workflow for runtime quality gates.
- **Importance**: 🔴 Critical | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Building and Evaluating Advanced RAG_ **Lessons 5-7** (TruLens RAG Triad, automated evaluation). Watch _Automated Testing for LLMOps_ **Lessons 2-4** (CI/CD integration, regression tracking).

### 7.4 Dual-Transcript Observability & Secret Redaction
- **Concept**: Maintaining two parallel event logs — a compact `transcript.jsonl` for fast debugging and a `transcript_full.jsonl` with complete payloads for deep analysis. Includes secret redaction to prevent credential leakage in logs.
- **Source Courses**: _Automated Testing for LLMOps_
- **Status**: ✅ Implemented
- **Where**: `hitl_email_agent/event_logger.py` — `HITLEventLogger` with SQLite audit table + dual JSONL transcripts with configurable truncation. Documented in `agentic_blueprints` and workspace Rule 9.
- **Gaps**: Redaction filter documented in `agentic_blueprints/docs/05_guardrails_and_operations/03_security_and_adversarial_hardening.md` but not implemented in the actual event logger. Transcripts may contain API keys or PII.
- **Importance**: 🟡 Medium | **Complexity**: ⭐⭐
- **Video**: Skip — your implementation is solid. Just implement the redaction filter from your own docs.

---

## Theme 8: Event-Driven & Async Agent Architectures

### 8.1 Event-Driven Agentic Workflows
- **Concept**: Asynchronous, event-driven agent architectures that execute tasks in parallel with complex branching, looping, and concurrent execution paths. Agents react to events (document arrival, approval callback, timer) rather than running synchronously.
- **Source Courses**: _Event-Driven Agentic Document Workflows (LlamaIndex)_
- **Status**: 🟡 Partial
- **Where**: `hitl_email_agent` is partially event-driven (approval webhooks trigger resume), but the core graph execution is synchronous. No true async event loop where multiple workflows run concurrently.
- **Gaps**:
  - No event bus / message queue architecture.
  - No concurrent workflow execution (can only run one email workflow at a time per process).
  - No document-arrival triggers (agent starts on webhook/file-watcher events).
  - No LlamaIndex Workflow-style step decorators (`@step`) for composable async pipelines.
- **Importance**: 🟠 High | **Complexity**: ⭐⭐⭐⭐
- **Video**: Watch _Event-Driven Agentic Document Workflows_ **Full course** (1 hour — the async patterns are architecturally distinct from LangGraph's synchronous model). Focus on **Lessons 3-5** (parallel step execution, branching, HITL in async context).

### 8.2 Adaptive Agent Behavior & Error Recovery
- **Concept**: Agents that dynamically adjust their strategy when encountering errors, unexpected outputs, or changed environments. Includes retry with modified parameters, fallback strategies, and self-healing loops.
- **Source Courses**: _Building Adaptive AI Agents_
- **Status**: 🟡 Partial
- **Where**: `Agents/core/llm_clients.py` has provider failover (Anthropic → HF → OpenAI). `Agents/core/utils.py` has `RepetitionDetector` to break stuck loops. `agentic_blueprints/docs/02_architectural_patterns/04_resilience_and_fallback_chains.md` documents fallback patterns.
- **Gaps**: 
  - No **tool-level retry with modified parameters** (e.g., if a web scrape fails, try a different URL or extraction method).
  - No **adaptive planning** (agent modifies its plan based on intermediate results).
  - No **environment sensing** (detecting changed conditions and adjusting behavior).
- **Importance**: 🟡 Medium | **Complexity**: ⭐⭐⭐
- **Video**: Watch _Building Adaptive AI Agents_ **Lessons 2-4** (error recovery patterns, adaptive navigation, reliability scoring).

---

## 📊 Gap Summary Matrix

| Concept | Status | Priority to Address |
|---------|--------|-------------------|
| A2A Protocol | ❌ Gap | 🟠 High — emerging standard |
| Parallelization topology (working code) | ❌ Gap | 🟠 High |
| Role-based multi-agent system (working code) | ❌ Gap | 🔴 Critical |
| Provider-native structured outputs | ✅ Implemented | Closed (Configurable Tri-Mode) |
| Memory consolidation pipeline | ❌ Gap | 🔴 Critical |
| Embedding-based memory retrieval | ❌ Gap | 🔴 Critical |
| Context budget monitoring | ❌ Gap | 🔴 Critical |
| RAG Triad / Ragas evaluation | ❌ Gap | 🔴 Critical |
| Red-teaming test suite | ❌ Gap | 🟠 High |
| Prompt caching (code) | ❌ Gap | 🟠 High |
| PII detection/masking | ❌ Gap | 🟠 High |
| Event-driven async architecture | ❌ Gap | 🟠 High |
| Parallel tool calling | ✅ Implemented | Closed (Configurable AsyncToolDispatcher) |
| OpenAPI spec auto-ingestion | ❌ Gap | 🟡 Medium — tracked in Agents backlog |
| OpenAPI spec auto-ingestion | ✅ Implemented | Closed (Light-mode multi-strategy loader) |
| Tiered HITL approval thresholds | 🟡 Partial | 🟠 High |
| Self-reflection pattern | 🟡 Partial | 🟠 High |
| Dynamic supervisor (generalized) | 🟡 Partial | 🔴 Critical |
| Semantic caching (verify & complete) | 🟡 Partial | 🟠 High |
| Transcript secret redaction | 🟡 Partial | 🟡 Medium |
| NL-to-SQL agent | 🟡 Partial | 🟡 Medium |

---

## 🎬 Video Watch List (Prioritized)

> Only videos where you have a **concept gap** that watching would help close.

| Priority | Course Video | What to Watch | Duration | Addresses Gap |
|----------|-------------|---------------|----------|---------------|
| 🔴 1 | _Context Engineering_ | Full course, focus Lessons 2-4 | ~1 hr | Context rot, structured slots, budget monitoring |
| 🔴 2 | _Agent Memory (Oracle)_ | Full course, focus Lessons 3-4 | ~1 hr | Memory engineering, consolidation, persistence |
| 🔴 3 | _CrewAI Multi-Agent Systems_ | Lessons 2-3 (roles/goals), 4 (hierarchical), 5-6 (parallel) | ~40 min | Multi-agent roles, parallelization |
| 🔴 4 | _Pydantic for LLM Workflows_ | Lessons 3-4 (provider-native structured output) | ~20 min | Native `response_format` integration |
| 🟠 5 | _A2A Protocol (Google/IBM)_ | Full course, focus Lessons 2-3 | ~1 hr | A2A server wrapping, agent discovery |
| 🟠 6 | _Building & Evaluating Advanced RAG_ | Lessons 5-7 (TruLens RAG Triad) | ~30 min | RAG evaluation, Ragas integration |
| 🟠 7 | _Red Teaming LLM Applications_ | Lessons 3-5 (automated red-teaming, Giskard) | ~30 min | Red-team test suite |
| 🟠 8 | _Event-Driven Document Workflows_ | Lessons 3-5 (async steps, parallel execution) | ~30 min | Event-driven architecture |
| 🟠 9 | _Function-Calling & Data Extraction_ | Lessons 4-5 (parallel calls, OpenAPI specs) | ~20 min | Parallel tool calling |
| 🟠 10 | _Semantic Caching for AI Agents_ | Lessons 2-4 (Redis, embedding similarity) | ~30 min | Semantic cache completion |
| 🟠 11 | _Long-Term Agentic Memory (LangGraph)_ | Lessons 2-4 (LangMem, memory scoring) | ~30 min | Memory persistence & scoring |
| 🟡 12 | _MCP (Anthropic)_ | Lessons 3-4 (resources, prompt templates) | ~20 min | MCP resources |
| 🟡 13 | _Safe AI via Guardrails_ | Lessons 3-5 (PII, hallucination guards) | ~30 min | Runtime guardrails |
| 🟡 14 | _Agentic AI (Andrew Ng)_ | Lesson 2 (Reflection pattern) | ~15 min | Self-reflection framing |

**Total estimated viewing**: ~7-8 hours (can be spread over 2-3 days focusing on the 🔴 items first).

---

> [!TIP]
> **Next Step**: Review this document, correct any status assessments I got wrong, then we pick the first concept gap to implement together in `hitl_email_agent` or `Agents`. My recommendation: start with **Role-based multi-agent system** (the biggest structural gap) or **Provider-native structured outputs** (the quickest win).
