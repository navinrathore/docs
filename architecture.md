# Global Architecture Decisions & Patterns

This document records overarching architectural decisions, design patterns, and systemic blueprints adopted across projects in this repository.

## Table of Contents
1. [Agent Memory Hierarchy & Lifecycle](#1-agent-memory-hierarchy--lifecycle)
2. [Interrupt-Save-Notify-Resume Pattern (Sovereign HITL Gate)](#2-interrupt-save-notify-resume-pattern-sovereign-hitl-gate)
3. [Hybrid Supervisory Wrapper Pattern (Rule 11)](#3-hybrid-supervisory-wrapper-pattern-rule-11)
4. [FastMCP Stdio Server & Hybrid Transport Pattern](#4-fastmcp-stdio-server--hybrid-transport-pattern)
5. [Architectural Triad: Deterministic Fallbacks vs. Fail-Closed Error Handling](#5-architectural-triad-deterministic-fallbacks-vs-fail-closed-error-handling)
6. [Universal Remote Notification Gateway & Asynchronous HITL Bridge](#6-universal-remote-notification-gateway--asynchronous-hitl-bridge)
7. [Universal Provider-Native Structured Output & Adaptive Tri-Mode Engine](#7-universal-provider-native-structured-output--adaptive-tri-mode-engine)
8. [OutputGuard & The Payload Offloading (Data Pointer) Pattern](#8-outputguard--the-payload-offloading-data-pointer-pattern)
9. [Dynamic Skill Architecture: Cascading Hybrid Routing & Progressive Disclosure](#9-dynamic-skill-architecture-cascading-hybrid-routing--progressive-disclosure)
10. [Sub-Agent Execution Engine & Context Sandboxing Pattern](#10-sub-agent-execution-engine--context-sandboxing-pattern)
11. [Resilient Async Tool Dispatcher & Bounded Concurrency](#11-resilient-async-tool-dispatcher--bounded-concurrency)
12. [Declarative OpenAPI Tool Auto-Ingestion & Schema Synthesis](#12-declarative-openapi-tool-auto-ingestion--schema-synthesis)
13. [Deterministic Code Sanitization & AST Syntax Verification](#13-deterministic-code-sanitization--ast-syntax-verification)
14. [Non-Consecutive Cyclic Repetition Detection](#14-non-consecutive-cyclic-repetition-detection)

---

## 1. Agent Memory Hierarchy & Lifecycle

**[2026-10-30]**  
*Source: `projects/hitl_email_agent`*

Agent memory must be decoupled into 4 distinct functional tiers rather than treated as a monolithic database:

| Tier | Memory Type | Implementation Pattern | Scope & Lifecycle | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **L0** | **Working Memory (Scratchpad)** | In-memory TypedDict / Pydantic schema in RAM (`StateGraph(StateSchema)`). | **Mandatory & Ephemeral**: Lives only while the process and node are executing. | Node-to-node variable handoff, local reasoning scratchpad. |
| **L1** | **Checkpoints (Execution State Snapshots)** | `SqliteSaver` / PostgreSQL with `JsonPlusSerializer` / `ormsgpack`. | **Optional & Durable**: Snapshot taken automatically after every node execution and on `interrupt()`. | Crash recovery, process halts (0 CPU/token burn), multi-day pauses, thread isolation (`thread_id`). |
| **L2** | **Episodic Memory (Touchpoints & Tracing)** | Relational interaction tables + dual transcripts (`transcript.jsonl` / `transcript_full.jsonl`). | **Append-Only History**: Logged per interaction/thread. | Auditing, regression testing, chronological interaction context. |
| **L3** | **Long-Term Semantic Memory (Profiles & Facts)** | Entity-keyed document store (`email TEXT PRIMARY KEY`) with SQLite WAL; upgradeable to vector indices (Chroma/FastEmbed). | **Cross-Session Persistent**: Shared across independent execution threads. | Recall recipient/user preferences, historical objections, and feedback adaptations across runs. |

---

## 2. Interrupt-Save-Notify-Resume Pattern (Sovereign HITL Gate)

**[2026-10-30]**  
*Source: `projects/hitl_email_agent`*

Autonomous agents performing high-stakes external actions (email dispatch, financial transactions, shell commands) must never use naive synchronous blocking (`input()`). Instead, use the **Interrupt-Save-Notify-Resume** pattern:

1. **Interception Gate**: An explicit state machine node (`human_review_node`) halts autonomous execution via LangGraph's native `interrupt()`.
2. **State Freeze**: LangGraph commits the entire active execution frame to SQLite (`checkpoints.db`) and terminates `.invoke()` with `__interrupt__`.
3. **Zero-Resource Pause**: The Python worker process terminates cleanly (0 CPU, 0 token burn). The review wait can last seconds, hours, or days.
4. **Notify & Operator Review**: The human operator reviews the draft via CLI, Webhook, or Slack, choosing:
   - `Approve`: Dispatches original draft.
   - `Edit`: Modifies fields (subject, recipient, multiline body) and injects edited state.
   - `Reject`: Safely routes to `abort_node` with feedback.
5. **Resume Controller**: Resumption occurs via `app.invoke(Command(resume=payload), config={"configurable": {"thread_id": tid}})`. The checkpointer rehydrates state exactly where it halted.
6. **Dual-Transcript Auditing (Rule 9)**: Every transition is mirrored into an immutable SQLite table (`agent_events`) and dual JSONL transcripts.

---

## 3. Hybrid Supervisory Wrapper Pattern (Rule 11)

**[2026-10-30]**  
*Source: `projects/hitl_email_agent`, `projects/Agents/core/mcp`*

- Core domain logic, MCP tool adapters, and database operations remain 100% deterministic, standalone, and fast.
- The AI / LangGraph layer acts exclusively as an optional supervisory orchestrator communicating via validated Pydantic schemas.
- Non-AI batch scripts or operators can bypass LLM calls with zero token overhead while maintaining exact schema compatibility.

---

## 4. FastMCP Stdio Server & Hybrid Transport Pattern

**[2026-10-30]**  
*Source: `projects/hitl_email_agent/src/hitl_agent/mcp_email_server.py`, `projects/Agents/core/mcp`*

- **Process Isolation**: External tools are encapsulated as standalone MCP servers running over standard I/O (`stdio` transport) using Anthropic's FastMCP SDK. Eliminates port conflicts and networking overhead.
- **Dual-Mode Hybrid Delivery Engine**:
  - **Local Simulated Mode (Default / Zero-Config)**: Safely redirects tool outputs (e.g. emails, shell actions) to local SQLite tables (`data/outbox.db`). Guarantees offline safety, deterministic testing, and zero accidental real-world side effects.
  - **Live Production Mode**: When external credentials (`GMAIL_USER`, `GMAIL_APP_PASSWORD`) are detected in the environment, transparently escalates to real-world network dispatch (e.g. TLS SMTP relay).
- **Tool Adapter Auto-Generation**: MCP JSON Schema tool declarations are dynamically converted into LangChain/LangGraph `StructuredTool` instances via `MCPToolAdapter`, keeping agent definitions protocol-agnostic.

---

## 5. Architectural Triad: Deterministic Fallbacks vs. Fail-Closed Error Handling

**[2026-11-04]**  
*Source: `projects/hitl_email_agent` (Supervisor, Researcher, Copywriter & Critic Gate)*

When architecting production agentic systems, deterministic fallbacks are often introduced for test hermeticity, bootstrapping, or degraded availability. However, applying deterministic heuristics indiscriminately across all agent layers introduces critical failure risks. A rigorous agent architecture must separate the system into a **Three-Tier Policy**:

```
+-------------------------------------------------------------------------------+
| Tier 1: Routing & Orchestration (Supervisor)      -> SAFE TO FALL BACK        |
| -> Heuristic rules enforce forward progress with 0 tokens and 0ms latency.    |
+-------------------------------------------------------------------------------+
| Tier 2: Content Generation (Researcher/Copywriter)-> CONDITIONAL / ANNOTATED  |
| -> Acceptable in low-risk marketing; DANGEROUS in clinical/legal/financial.    |
| -> When falling back, MUST annotate state with provenance (is_fallback=True). |
+-------------------------------------------------------------------------------+
| Tier 3: Safety, Compliance & Guardrails (Critic)  -> STRICT FAIL-CLOSED       |
| -> NEVER fallback to auto-passing. If LLM fails, raise error or block action. |
+-------------------------------------------------------------------------------+
```

### 1. Where Deterministic Fallbacks Are Superior (Production Asset)
- **Supervisor Routing**: State progression (e.g. `no research -> research; research ready -> draft; draft ready -> critic; critic approved -> review`) is structurally deterministic. Relying on an LLM for trivial turn-taking imposes a continuous latency and token tax. Deterministic routing guarantees forward progress, zero token spend, and immunity to model outages.
- **Offline / Sandboxed CI/CD Verification**: Hermetic deterministic mocks allow entire 80+ test suites to execute offline in seconds with $0 cloud spend and 0 network flakiness.
- **Structured Output Repair**: Deterministic regex extraction and JSON stripping (`extract_json_payload`) before invoking LLM repair loops saves 100% of re-prompt tokens for common markdown code-block wrapping.

### 2. Where Deterministic Fallbacks Become Dangerous Anti-Patterns
- **The "Silent Hallucination" Trap (Content Generation)**: In high-stakes domains (legal, medical, enterprise contracts), falling back to boilerplate text when an LLM fails masks an upstream outage. The operator or customer assumes the agent conducted bespoke research, whereas it silently emitted generic canned copy.
- **The "Fail-Open" Security Vulnerability (Safety & Compliance Gates)**: If the Pre-Review Critic Gate encounters an API timeout or 500 error, falling back to a deterministic heuristic that marks `passed=True` is catastrophic: unvetted, potentially toxic, or legally non-compliant drafts would slip directly to human dispatch.

### 3. Production Guidelines & Design Rules
1. **Routing is Fail-Safe**: Orchestrators should maintain deterministic state machine fallback rules.
2. **Generators Require Provenance**: When a generator uses a template fallback due to model unavailability, it must tag the output metadata (`provenance="heuristic_fallback"`, `model="deterministic_template"`).
3. **Audit & Safety Gates are Fail-Closed**: Compliance/safety judges must default to `passed=False` with explicit alerting whenever the model is unavailable, escalating to human review with `error_message="Critic evaluation unavailable; manual audit mandatory"`.

---

## 6. Universal Remote Notification Gateway & Asynchronous HITL Bridge

**[2026-11-12]**  
*Source: `projects/Agents/core/notifications.py`, `projects/hitl_email_agent/scripts/approval_server.py`*

Deploying autonomous agents with human-in-the-loop (HITL) checkpoints requires remote notification and out-of-band resumption. Operators cannot sit in front of terminal CLIs waiting for interrupts. To support cross-platform mobile and desktop approvals without tightly coupling agent logic to specific chat vendors, the architecture adopts a **Decoupled Hub-and-Spoke Gateway**:

```
Agent State Machine (LangGraph interrupt())
         │
         ▼
NotificationGateway (projects/Agents/core/notifications.py)
         │ [ReviewRequestPayload]
         ├──► TelegramChannel (Inline Bot Keyboard - $0 Freeware)
         ├──► WebhookChannel (Generic HTTP POST JSON -> n8n / Zapier / Custom UI)
         └──► ConsoleChannel / MockChannel (Offline CI & Terminal Fallback)
                                   │
                      Human Operator taps [Approve] / [Reject]
                                   │
                                   ▼
Inbound Approval Server (FastAPI: scripts/approval_server.py)
         │ POST /api/callback or /api/telegram/webhook
         ▼
Controller Resumption: Command(resume=HumanReviewPayload)
         │
         ▼
Checkpointer Hydrates Thread & Unpauses LangGraph Execution
```

### 1. Key Architectural Principles
- **Provider-Agnostic Notification Engine**: The core agent logic never imports or directly manages proprietary chat SDKs. It emits a standardized, validated Pydantic model (`ReviewRequestPayload`) containing metadata, thread ID, target recipient, summary, and action choices (`NotificationAction`).
- **Zero-Cost / Freeware First**: Prioritize channels that incur \$0 in operational costs:
  - **Telegram Bot API**: Free bot token, instant push notifications, and native interactive inline keyboard buttons (`[Approve]`, `[Reject]`).
  - **Generic Webhook**: Universal bridge emitting standard JSON HTTP POST payloads.
- **Architectural Symmetry (Telegram vs. WhatsApp)**: Telegram and WhatsApp share an identical two-way interaction contract: outbound card dispatch with action IDs + inbound callback carrying `thread_id` and `action`. Developing and verifying on Telegram delivers 100% architectural parity with WhatsApp (Meta Cloud API / Twilio) with zero setup friction and zero dollar cost.
- **Webhook as the Integration Bridge (iPaaS)**: Rather than writing bespoke integrations for Slack, Teams, Discord, and WhatsApp inside Python agent code, the `WebhookChannel` points to an automation engine (e.g. self-hosted n8n or Zapier). The iPaaS handles formatting, authentication, and multi-channel fanout, while Python agent code remains completely isolated from third-party API drift.
- **Asynchronous Inbound Resumption**: Inbound approval callbacks are handled by an independent, non-blocking HTTP bridge (`scripts/approval_server.py`). The approval server translates external webhooks or inline button taps into `ctrl.resume_workflow(thread_id, payload)`, allowing workflows to pause safely for minutes, hours, or days without burning process memory or compute.

---

## 7. Universal Provider-Native Structured Output & Adaptive Tri-Mode Engine

**[2026-11-18]**  
*Source: `projects/Agents/core/llm_clients.py`, `projects/hitl_email_agent/src/hitl_agent/llm.py`*

In production agentic loops, relying on plain text prompting and post-hoc regex or string extraction to obtain structured JSON is an antipattern. When errors compound across a 10-step agent loop, even a 5% parsing error rate results in a $\approx 40\%$ failure rate ($0.95^{10} \approx 60\%$).

To ensure zero parsing errors while preserving provider agnosticism and open-source runtime flexibility, the architecture establishes a **Universal Tri-Mode Structured Output Engine**:

```
client.generate_structured(messages, schema=MySchema, mode="auto"|"native"|"prompt_injection")
                       │
                       ▼
          [Check Effective Mode]
          ├── "native" / "auto" ──► Attempt Provider-Level Constrained Decoding
          │                           ├── OpenAI / HF Router / vLLM:
          │                           │     response_format={"type": "json_schema", "strict": True}
          │                           │     (pre-processed with prepare_strict_json_schema)
          │                           ├── Anthropic Claude:
          │                           │     Forced Tool Calling (tools=[schema], tool_choice=schema)
          │                           └── MockLLMClient:
          │                                 Direct JSON validation & mode tracking
          │
          └── If "auto" & Endpoint Rejects (e.g. 400 Bad Request / Unsupported Parameter)
              OR if mode == "prompt_injection":
                       │
                       ▼
              [Graceful Fallback Mode]
              Inject JSON Schema into Prompt + StructuredOutputParser + Single-Turn Repair Loop
```

### 1. Key Architectural Principles
- **Grammar-Guided Decoding at the Boundary**: For providers and runtimes supporting constrained decoding (OpenAI, vLLM via Outlines/XGrammar, Ollama, Hugging Face Router), the schema is passed via `response_format`. The inference engine masks illegal logits to $-\infty$ at token-sampling time, mathematically preventing syntax or field errors.
- **Provider-Agnostic Adapter Translation**:
  - **OpenAI / OpenAI-Compatible (vLLM / HF Router)**: Maps Pydantic schema to OpenAI `json_schema` with `strict: True`.
  - **Anthropic Claude**: Anthropic lacks a native top-level `json_schema` response format; the engine automatically maps the Pydantic schema to a forced tool definition (`tools=[...]`, `tool_choice={"type": "tool", "name": schema.__name__}`).
- **Strict Schema Invariants (`prepare_strict_json_schema`)**: OpenAI strict mode enforces rigorous rules that vanilla Pydantic schemas do not supply by default. The recursive pre-processor ensures:
  1. `additionalProperties: False` is added to all object nodes.
  2. All properties declared in `properties` are explicitly added to `required` (ensuring optional fields use nullable unions rather than omitted keys).
- **Adaptive Fallback Defense ("Auto" Mode)**: If an external OpenAI-compatible router or local model server rejects `response_format` or strict grammar masking with a `400 Bad Request`, `mode="auto"` logs a warning and transparently falls back to prompt injection + JSON repair without terminating the agent workflow.
- **Static Token Fast-Forwarding (Compute Optimization)**: When using constrained decoding, predictable schema boundaries (e.g., `", "field": `) have only one legal transition in the DFA. Production inference engines bypass GPU forward passes for these static paths and append them directly to the KV cache, achieving 30–50% latency and compute cost savings.
- **Chronological Schema Design (The Reasoning Buffer)**: Due to causal autoregressive attention ($P(t_k \mid t_{<k})$), schemas must order fields chronologically: `thought_process` / `reasoning` (unconstrained strings) must precede `decision` / `actions` (constrained enums). This populates the model's KV cache with analytical working memory before it samples final actions.

---

## 8. OutputGuard & The Payload Offloading (Data Pointer) Pattern

**[2026-11-23]**  
*Source: `projects/Agents/core/output_guard.py`, `projects/Agents/core/pointer_tools.py`, `projects/Agents/core/dispatcher.py`*

In production autonomous agent loops, tools frequently produce massive outputs (megabytes of SQL records, API JSON dumps, or full file trees). Dumping these raw strings directly into LLM context causes severe **Context Satiation**, massive token costs, and attention dilution.

To solve this deterministically, the architecture implements the **Payload Offloading Pattern**:

```
[Tool Execution] ──► Raw Output (e.g. 150 KB / 5000 lines)
                             │
                             ▼
                    [OutputGuard Engine]
                    ├── Output <= Threshold ──► Passthrough to Context
                    └── Output > Threshold  ──► Offload & Truncate:
                                                 ├── 1. Persist lossless payload to disk
                                                 │      (vault/pointers/{sha256}.txt)
                                                 ├── 2. Generate DataPointer manifest
                                                 │      (pointer_id, byte_size, line_count, preview)
                                                 └── 3. Smart Truncate for LLM Context:
                                                        [Head Lines (preserved)]
                                                        [... N lines omitted (DataPointer: id) ...]
                                                        [Tail Lines (preserved)]
                                                              │
                                                              ▼
                                                   [Selective Inspection Tools]
                                                   ├── read_pointer_slice(pointer_id, offset, limit)
                                                   ├── grep_pointer(pointer_id, regex)
                                                   └── query_pointer_json(pointer_id, jsonpath)
```

### 1. Key Architectural Principles
- **Smart Head/Tail Preservation**: Most operational diagnostic signals occur at the start (headers, schema definitions) or at the end (summary counts, error stacks). The truncator keeps both and cleanly displays omitted statistics.
- **Content-Addressable Storage**: Payloads are indexed by their SHA-256 hash. If duplicate payloads are produced across turns, they deduplicate automatically without redundant storage.
- **Pointers as First-Class Citizens**: The LLM receives a structured `DataPointer` reference. When deeper inspection is needed, it uses lightweight pointer query tools (`read_pointer_slice`, `grep_pointer`, `query_pointer_json`) to pull only relevant micro-slices into working memory.

---

## 9. Dynamic Skill Architecture: Cascading Hybrid Routing & Progressive Disclosure

**[2026-11-23]**  
*Source: `projects/Agents/core/skills.py`*

Static system prompts containing all possible instructions and tool definitions fail at enterprise scale. Loading 50+ tool schemas consumes tens of thousands of tokens per turn and causes tool-hallucination.

The architecture decouples tools and operating procedures into self-contained **Skill Packages** activated via a **Cascading Hybrid Router**:

```
User Query: "Check customer churn rate in Postgres"
                      │
                      ▼
          [Cascading Hybrid Router]
          ├── Tier 1: Deterministic Fast Path (Sub-millisecond, $0 Token Tax)
          │           Evaluate trigger keywords/prefixes from SKILL.md frontmatter
          │           Match Found? ──► Activate SkillPackage
          │
          └── Tier 2: Semantic / LLM Fallback (Invoked only on ambiguous queries)
                      Lightweight structured classifier maps intent to skill ID
                      Match Found? ──► Activate SkillPackage
                                            │
                                            ▼
                           [Progressive Disclosure Flow]
                           ├── System Prompt: Hydrate declarative SOP only
                           └── Dispatcher: Dynamically mount skill tool callables
```

### 1. Key Architectural Principles
- **Tier 1 Fast Path**: Over 70% of standard user commands match predefined trigger keywords. Bypassing the LLM for routing saves hundreds of milliseconds and thousands of API calls daily.
- **Progressive Disclosure**: Only the active skill's Standard Operating Procedure (`SKILL.md`) and tool schemas enter prompt context. Underlying execution code is never shown to the LLM.
- **Zero-Friction Fallback**: If routing produces zero confident matches, the agent degrades gracefully to default conversational assistance.

---

## 10. Sub-Agent Execution Engine & Context Sandboxing Pattern

**[2026-11-23]**  
*Source: `projects/Agents/core/subagent.py`*

When an agent needs to execute a complex multi-turn task (e.g., iterative SQL exploration, multi-step code refactoring, or document auditing), running those intermediate turns inside the primary conversation pollutes the parent's context window with temporary errors and noisy tool outputs.

The architecture solves this via **Context Sandboxing with Sub-Agent Tool Bridging**:

```
[Parent Agent Loop]
        │
        ├── LLM issues Tool Call: delegate_to_data_analyst(task="...")
        │           │
        │           ▼
        │   [Async Tool Dispatcher Bridge]
        │           │
        │           ▼
        │   [SkillSubAgent Isolated ReAct Loop]
        │   ├── Dedicated System Prompt (Skill SOP only)
        │   ├── Scoped Dispatcher (Skill tools only)
        │   ├── Safety Ceilings:
        │   │     ├── max_loops (Hard loop limit, Rule 1)
        │   │     ├── timeout_seconds (asyncio.wait_for, Rule 4)
        │   │     └── RepetitionDetector (Non-consecutive error cycle abort)
        │   └── Multi-turn Execution & Final Synthesis
        │           │
        │           ▼
        └── Tool Result: "Users table has 4 columns: id, name, email, created_at."
            (Parent context receives ONLY the clean final summary!)
```

### 1. Key Architectural Principles
- **Sub-Agent as an Async Tool**: The entire sub-agent lifecycle is encapsulated as a standard async tool function on the parent dispatcher.
- **Concurrent Sub-Agents**: Because tools run asynchronously through `AsyncToolDispatcher`, the parent agent can launch multiple sub-agents concurrently (e.g. `delegate_to_data_analyst` and `delegate_to_security_scanner` running in parallel via `asyncio.gather`).
- **Complete Context Isolation**: Child reasoning turns, intermediate errors, and raw outputs are completely sandboxed. The parent agent's context receives strictly the final synthesized answer.

---

## 11. Resilient Async Tool Dispatcher & Bounded Concurrency

**[2026-11-23]**  
*Source: `projects/Agents/core/dispatcher.py`*

In production agent architectures, executing tools synchronously or in an unbounded parallel loop creates severe bottlenecks: a single hanging tool freezes the agent, and calling 10 tools simultaneously can exhaust database connection pools or trigger API rate limits.

The `AsyncToolDispatcher` provides a robust, production-grade tool execution runtime:

```
[Agent Emits N Tool Calls] ──► [AsyncToolDispatcher]
                                     │
                                     ├── [Bounded Semaphore] (max_concurrency=4)
                                     │     Prevents connection pool exhaustion
                                     │
                                     ├── [Per-Tool Timeout Guard] (asyncio.wait_for)
                                     │     Isolates slow tools (e.g. 10s ceiling)
                                     │
                                     ├── [Exception Shielding] (return_exceptions=True)
                                     │     One tool failure does NOT abort sibling tools
                                     │
                                     └── [Order-Preserving Result Collector]
                                           Maps results back to exact tool_call_id
```

### 1. Key Architectural Principles
- **Semaphore-Bounded Concurrency**: Tools execute concurrently via `asyncio.gather`, but concurrency is strictly throttled by an `asyncio.Semaphore(max_concurrency)` to prevent thread, file descriptor, or network socket exhaustion.
- **Per-Tool Timeout Isolation**: Every tool call is wrapped in `asyncio.wait_for(coro, timeout=...)`. If a third-party API or database query stalls, it times out independently and returns a structured error string, allowing the agent to diagnose the failure without crashing.
- **Exception Shielding & Structured Result Guarantees**: Dispatcher never allows unhandled tool exceptions to bubble up and kill the agent process. Exceptions are captured, formatted into `ToolExecutionResult(success=False, error=str(e))`, and fed cleanly into the context.
- **Unified Transport Abstraction**: The dispatcher seamlessly mounts:
  1. Standard Python sync and async functions (automatically offloaded to thread pool via `asyncio.to_thread` if sync).
  2. FastMCP stdio/SSE tools (`register_mcp_tool`).
  3. HITL approval-gated actions (`require_approval=True`, `risk_predicate`).
  4. Sub-agent child ReAct loops.
  5. `OutputGuard` automatic payload truncation and offloading.

---

## 12. Declarative OpenAPI Tool Auto-Ingestion & Schema Synthesis

**[2026-11-23]**  
*Source: `projects/Agents/core/openapi_loader.py`*

Writing custom Python tool wrappers by hand for hundreds of corporate REST endpoints is brittle, slow, and error-prone. 

The `OpenAPIToolLoader` dynamically ingests OpenAPI 3.0/3.1 and Swagger specs (JSON/YAML) and compiles them into typed, validated agent tools:

```
[OpenAPI 3.0 / 3.1 Spec (JSON/YAML)]
                 │
                 ▼
       [OpenAPIToolLoader]
                 │
                 ├── 1. Path & Parameter Extraction
                 │      (query, path, header, requestBody)
                 │
                 ├── 2. JSON Schema Parameter Synthesis
                 │      (Compiles parameters into strict OpenAI/Anthropic schemas)
                 │
                 ├── 3. Dynamic HTTP Execution Handlers
                 │      (Injects URL path parameters, headers, and payloads via httpx)
                 │
                 └── 4. Enterprise Safety Policies:
                        ├── read_only=True (Filters to GET operations only)
                        ├── tag_whitelist / path_filter (Scoped access)
                        └── dry_run=True (Simulates requests without network side-effects)
```

### 1. Key Architectural Principles
- **Schema Auto-Compilation**: The loader maps OpenAPI parameter types (`string`, `integer`, `array`, `object`) and nested schema components into standard JSON Schema parameter dictionaries compatible with OpenAI function calling and Anthropic tool schemas.
- **Dynamic Callable Generation**: Each operation is converted into a native async execution callable that automatically injects URL path variables (e.g. `/users/{id}` $\rightarrow$ `/users/123`), formats query parameters, and serializes request bodies.
- **Configurable Synthesis Modes**:
  - **1:1 Endpoint Mapping**: Every endpoint operation (`getUser`, `createOrder`) becomes a distinct, named agent tool.
  - **Dynamic API Runner**: Ingests the spec as an API catalog and exposes a single generic `call_api(path, method, params)` tool, ideal for frontier models navigating vast API surfaces without bloating tool definitions.
- **Safety by Default**: Built-in support for `read_only=True` enforces that mutation methods (`POST`, `PUT`, `DELETE`, `PATCH`) are excluded at ingestion time, mathematically preventing destructive operations.

---

## 13. Deterministic Code Sanitization & AST Syntax Verification

**[2026-11-23]**  
*Source: `projects/Agents/core/sanitizer.py`*

Autonomous code-generation and data-science agents frequently receive outputs where the LLM wrapped code in conversational preamble (`"Here is the script you requested:"`), trailing commentary, multi-level markdown fences, or shell wrappers (`cat << 'EOF'`). Passing raw LLM responses to a Python interpreter or bash shell results in frequent syntax failures.

The `CodeSanitizer` provides a deterministic pipeline for extracting, cleaning, and verifying code before execution:

```
[Raw LLM Response]
        │
        ├── 1. Line Ending & Indentation Normalization
        │      (Preserves relative indent across lines before dedenting)
        │
        ├── 2. Multi-Backtick Fence Extraction via Named Backreferences
        │      (?P<ticks>`{3,})(?:[a-zA-Z0-9_-]+)?\n(?P<code>[\s\S]*?)\n(?P=ticks)
        │      (Prevents premature termination on nested 3-backtick docstrings)
        │
        ├── 3. Conversational Fluff & Heredoc Stripping
        │      (Removes "Here is the code:", "cat << 'EOF'", trailing explanations)
        │
        └── 4. Static Python AST Verification (ast.parse)
               ├── Syntax Valid?   ──► SanitizationResult(success=True, code=clean_code)
               └── Syntax Error?   ──► SanitizationResult(success=False, error="SyntaxError: ...")
```

### 1. Key Architectural Principles
- **Nested Fence Safety**: By using regex named backreferences (`(?P<ticks>`{3,})...(?P=ticks)`), code blocks wrapped in 4 or 5 backticks that contain inner 3-backtick markdown blocks are extracted without premature truncation.
- **Indentation Preservation**: Stripping leading whitespace from the first line destroys the common indentation prefix required by `textwrap.dedent()`. The sanitizer preserves relative line indentation prior to conversational extraction.
- **Pre-Flight AST Parsing**: Rather than executing unverified code and failing inside an expensive sandbox or subprocess, `ast.parse` validates Python syntax in sub-millisecond static analysis, catching syntax errors before execution.

---

## 14. Non-Consecutive Cyclic Repetition Detection

**[2026-11-23]**  
*Source: `projects/Agents/core/utils.py`*

A major failure mode in autonomous ReAct loops is the **ping-pong error trap**: the agent encounters an error $A$, attempts an alternative approach that causes error $B$, and then returns to the first approach, repeating $A \rightarrow B \rightarrow A \rightarrow B$ until its context window or credit balance is exhausted.

Traditional repetition detectors only check consecutive identical errors (e.g., $A \rightarrow A \rightarrow A$) and completely miss alternating or multi-step cycles.

The upgraded `RepetitionDetector` implements **Sliding-Window Hash-Based Cycle Detection**:

```
Agent Tool Output
        │
        ▼
[Normalize & Hash] ──► SHA-256 Digest (e.g. "a3f5...e9")
                             │
                             ▼
                    [Sliding Window Deque] (maxlen = 10)
                    [ h(A), h(B), h(C), h(A), h(B), h(A) ]
                             │
                             ▼
                    [Counter Frequency Check]
                    Frequency of h(A) in window >= threshold (3)?
                    ├── YES ──► ABORT LOOP (Cyclic Repetition Trap Detected)
                    └── NO  ──► Continue Loop
```

### 1. Key Architectural Principles
- **Normalization Before Hashing**: Strips timestamps, memory addresses (e.g., `<object at 0x7f...>`), and variable whitespace before hashing so identical semantic errors produce identical SHA-256 digests.
- **Non-Consecutive Detection**: By tracking hash frequencies across an in-memory sliding window (`deque(maxlen=window_size)` + `collections.Counter`), multi-step cycles ($A \rightarrow B \rightarrow A \rightarrow B$) are caught reliably regardless of sequence order.
- **Loop Circuit Breaker**: Once the repetition threshold is breached, the loop terminates immediately with an actionable warning, preventing token burn and API rate-limiting (enforcing Rule 1 and Rule 9).






