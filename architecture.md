# Global Architecture Decisions & Patterns

This document records overarching architectural decisions, design patterns, and systemic blueprints adopted across projects in this repository.

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


