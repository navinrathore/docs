# Global Learnings & Technical Gotchas

This document records practical lessons learned, operational gotchas, and debugging discoveries across projects in the AI workspace.

---

## 1. LangGraph Checkpointing & StateGraph Execution Mechanics

**[2026-10-30]**  
*Source: `projects/hitl_email_agent`*

- **Automatic Checkpointing Cadence**: LangGraph executes `checkpointer.put(...)` automatically after **every superstep** (every node completion), at `START`/`END`, and upon calling `interrupt()`. Application developers do not need to manually save state during normal graph execution.
- **On-Demand State Injection**: State can be inspected or updated outside node execution using `app.get_state(config)` and `app.update_state(config, values)`. Updating state writes a new checkpoint snapshot without triggering node transitions.
- **Pydantic Model Msgpack Serialization**: When passing custom Pydantic models (e.g. `EmailDraft`) into LangGraph state with `SqliteSaver`, set the environment variable:
  ```python
  import os
  os.environ.setdefault("LANGGRAPH_ALLOWED_MSGPACK_MODULES", "hitl_agent.schemas")
  ```
  and configure `JsonPlusSerializer(allowed_msgpack_modules=[("hitl_agent.schemas", "EmailDraft")])`. Without this, checkpoint deserialization will fail or fall back to raw dictionaries.
- **Defensive State Coercion**: Even with typed schemas, nodes receiving resumed state must defensively coerce dictionary inputs using `Model.model_validate(data)` because JSON serialization rounds objects to native dicts upon deserialization.

---

## 2. Memory Subsystems: The "Storage vs. Hydration" Wiring Gap

**[2026-10-30]**  
*Source: `projects/hitl_email_agent`*

- **The Trap**: It is easy to build comprehensive persistent memory stores (`RecipientMemoryStore`, `learn_from_review()`) and write test suites verifying that interactions are stored in SQLite, yet leave the agent blind because early workflow nodes (`research_node`, `draft_node`) never call the retrieval/hydration function.
- **Best Practice**: Whenever adding an L2/L3 memory store, always trace the full lifecycle:
  1. **Write Path**: Ingestion & feedback recording on workflow completion / review.
  2. **Read Path (Hydration)**: Retrieval and injection of relevant profile context, tone, and past objections into the initial prompt context before generation.
  3. **Verification**: Write an automated test confirming that stored learnings change the subsequent draft output.

---

## 3. Deterministic Cross-Process Crash Recovery Verification

**[2026-10-30]**  
*Source: `projects/hitl_email_agent`*

- To test crash resilience without flaky sleep timers or OS kills:
  1. Start a workflow on thread $T$ until it halts at `__interrupt__`.
  2. Explicitly call `.close()` and delete the controller/runtime instance (`del controller`).
  3. Instantiate an entirely separate controller instance pointing to the same SQLite file on disk.
  4. Query `list_pending_workflows()` and verify that the draft and thread state hydrate intact.
  5. Resume execution with `resume_workflow(T, payload)` and verify full completion.

---

## 4. The Fallback Dilemma: When Heuristics Accelerate CI vs. Mask Production Failures

**[2026-11-04]**  
*Source: `projects/hitl_email_agent` (Multi-Agent Supervisor & Critic Gate)*

- **The Dual Purpose of Heuristic Fallbacks**:
  In early development and CI, deterministic fallbacks allow full end-to-end testing of complex state machines (SqliteSaver checkpoints, interrupt recovery, event logging) with $0 token spend, zero API keys, and instantaneous test cycles.
- **The Production Danger**:
  If an application silently falls back to hardcoded string generation or heuristic passes during a live model outage, it creates a **silent degradation failure mode**:
  - The agent appears operational on dashboards (no 500 errors).
  - But the output quality collapses to static templates, and compliance gates stop auditing content.
- **Hard Rule for Production Deployments**:
  1. **Strict Production Toggling**: Expose an explicit environment variable (e.g. `ALLOW_DETERMINISTIC_FALLBACKS=false`) in production environments. When set to `false`, LLM client timeouts or API failures must raise explicit exceptions (`AgentLLMUnavailableError`) rather than falling back to canned text.
  2. **Fail-Closed Critic Standard**: If a critic gate cannot reach its evaluation model, it must NEVER emit `passed=True`. It must set `passed=False` and report `"Evaluation engine unreachable; human review required"`.

---

## 5. Starlette 1.x / FastAPI Interoperability Gotcha & Backward Compatibility Shim

**[2026-11-12]**  
*Source: `projects/hitl_email_agent/scripts/approval_server.py`*

- **The Problem**: When upgrading environments or running Starlette 1.x alongside FastAPI 0.111.x, Starlette removed the deprecated `on_startup` and `on_shutdown` keyword arguments from `starlette.routing.Router.__init__` and added a `self.max_body_size` attribute access in `build_middleware_stack()`. Instantiating `FastAPI()` or invoking `TestClient(app)` raises:
  - `TypeError: Router.__init__() got an unexpected keyword argument 'on_startup'`
  - `AttributeError: 'FastAPI' object has no attribute 'max_body_size'`
- **The Solution (Clean Compatibility Shim)**: Before importing or instantiating `FastAPI`, patch the Starlette routing constructor and FastAPI initialization to safely strip the legacy kwargs and initialize `self.max_body_size = None`:
  ```python
  import starlette.routing
  _orig_router_init = starlette.routing.Router.__init__
  def _safe_router_init(self, *args, **kwargs):
      kwargs.pop("on_startup", None)
      kwargs.pop("on_shutdown", None)
      return _orig_router_init(self, *args, **kwargs)
  starlette.routing.Router.__init__ = _safe_router_init

  from fastapi.applications import FastAPI as _FastAPIClass
  _orig_fastapi_init = _FastAPIClass.__init__
  def _safe_fastapi_init(self, *args, **kwargs):
      self.max_body_size = None
      return _orig_fastapi_init(self, *args, **kwargs)
  _FastAPIClass.__init__ = _safe_fastapi_init
  ```
  This restores full compatibility across test suites and runtime servers without breaking environment constraints or lockfiles.

---

## 6. SqliteSaver Checkpoint Deserialization: `allowed_msgpack_modules` & Model Tiering

**[2026-11-12]**  
*Source: `projects/hitl_email_agent/src/hitl_agent/graph.py`, `projects/hitl_email_agent/src/hitl_agent/supervisor.py`*

- **Why `allowed_msgpack_modules` is Required**: LangGraph's checkpointer uses `ormsgpack` / `msgpack` for high-throughput binary serialization. For security against arbitrary code execution during deserialization, `JsonPlusSerializer` strictly restricts which Python modules are permitted to unpack structured objects. If custom Pydantic schemas (e.g. `EmailDraft`, `ResearchBriefing`) are not explicitly registered in `allowed_msgpack_modules`, the checkpointer strips them into raw dictionaries or rejects deserialization.
- **Cross-Model Supervisor Decoupling**: In multi-agent hub-and-spoke graphs, the supervisor router does not need to run on an expensive frontier model. The supervisor router's primary job is deciding turn-taking between `research_node`, `draft_node`, and `critic_node`. It can run on a small, fast local model or deterministic heuristics ($0 token burn), reserving larger, creative LLMs (e.g. Claude 3.5 Sonnet, GPT-4o, Mistral Large) exclusively for the specialist content nodes.


