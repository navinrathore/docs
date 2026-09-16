# AI Conversation Scratch Pad & Key Takeaways

**Purpose:** A centralized reference log capturing key technical concepts, architectural patterns, and takeaways discussed during AI pair programming sessions.

---

## 📌 Topic 1: Prompt Caching & Prefix Optimization (Frontier LLMs)

### 1. Underlying Mechanics
- **KV Cache Reuse:** Autoregressive Transformers calculate Key ($K$) and Value ($V$) tensors for each token during the prefill phase. Frameworks (vLLM RadixAttention, SGLang, Anthropic, OpenAI, Gemini) cache these $KV$ tensors in GPU memory using a Radix Tree.
- **Causal Masking Constraint:** Token $N$ only attends to prior tokens ($1 \dots N-1$). The representation of token $N$ depends strictly on the exact sequence before it.
- **Prefix Matching (Index 0):** Caching requires an *exact prefix match* starting at token index `0`. Modifying even a single token near the top invalidates all subsequent KV cache states.

### 2. Economics & Performance
- **Cost Discount:** Providers pass FLOPs savings to users (Anthropic ~90% discount, OpenAI ~50% discount, Gemini/DeepSeek up to 90%).
- **Latency (TTFT):** Significantly reduces Time-To-First-Token on long prompts (10k–100k+ tokens).

### 3. Key Rules for Prompt Engineering & Architecture
- **Top-Heavy Static Context:** Place static elements (system prompts, tool definitions, SOPs, developer guidelines) at the **very top** of the prompt payload.
- **Bottom-Heavy Dynamic Context:** Append dynamic variables (user query, timestamps, RAG retrieval outputs) at the **end** of the prompt payload.

---

## 📌 Topic 2: Multi-Turn Chat Conversation Caching

### 1. How It Works in Practice
- **Append-Only History:** In a chat session, Turn $N$ contains `[System Prompt] + [Turn 1] + [Assistant 1] + ... + [Turn N]`.
- **Automatic Cache Hits:** Because previous chat turns form an exact prefix match for the next turn, the provider reuses the KV cache for the entire chat history.

### 2. Cost Trajectory
- **Without Caching:** Billed at full prefill rate for cumulative history every turn ($1\text{K} + 2\text{K} + 3\text{K} + \dots + 20\text{K}$ tokens).
- **With Caching:** You only pay full price for the *new incremental tokens* on each turn; previous turns are billed at the ~90% discounted cache rate.

### 3. Key Constraints to Avoid Breaking Cache
1. **No Editing Mid-History:** Modifying or summarizing an earlier message breaks the prefix match from that point forward.
2. **Time-To-Live (TTL):** Providers keep KV caches warm for 5–60 minutes. Long idle gaps between user turns can cause a cache miss.
3. **Cache Affinity:** Route consecutive chat turns to the same model/provider endpoint to maximize cache hit rates.

## 📌 Topic 3: Small Language Models (SLMs) — Architecture, Resource Sizing & CPU Dynamics

### 1. Underlying Mechanics & API Parity
- **API Parity**: Network interfaces (`/v1/chat/completions`) and SDK abstractions (`BaseLLMClient`, `LiteLLM`) are **100% identical** between frontier LLMs and local SLMs.
- **Operational Differences**: SLMs require concise system prompts, mandatory JSON Schema/GBNF grammar-guided output enforcement at API boundaries, and lower context window tolerance (avoid context stuffing; use RAG/KIs).
- **Reasoning SLMs**: Distilled reasoning models (e.g. DeepSeek-R1-Distill) output explicit `<think>...</think>` tokens that must be parsed out before tool invocation.

### 2. Hardware Resource Sizing & VRAM Formulas
- **VRAM Formula**: $\text{VRAM}_{\text{Total}} = \text{VRAM}_{\text{Weights}} + \text{VRAM}_{\text{KV Cache}} + \text{Overhead}$.
- **Weight Size Formula**: $\text{VRAM}_{\text{Weights}} = P \times b \times 1.15$ (where $P$ is billions of params, $b=0.55$ for 4-bit Q4_K_M).
- **Rule of Thumb (4-bit Q4_K_M)**:
  - **1B - 3B Params**: Fits in 2GB - 4GB VRAM (Integrated GPU or basic card).
  - **7B - 8B Params**: Fits in 8GB VRAM (RTX 3060/4060) with ~5GB VRAM footprint.
  - **14B Params**: Fits in 12GB - 16GB VRAM (RTX 3060 12GB / RTX 4070).

### 3. CPU Execution Dynamics & Memory Bandwidth
- **Compute vs. Memory Bound**:
  - **Prefill (Prompt Processing)**: Compute-bound ($O(N^2)$). Scales linearly across all physical CPU cores using SIMD extensions (AVX-512, AMX, ARM NEON).
  - **Generation (Token-by-Token)**: Memory-bandwidth bound ($O(N)$). The CPU must read the full model weight matrix from System RAM for *every single token*.
- **Token Speed Formula**: $\text{Tokens/Sec} \approx \frac{\text{System RAM Bandwidth (GB/s)}}{\text{Model Size in RAM (GB)}}$.
  - DDR4 (~35 GB/s) $\to$ 7.2 tokens/sec (7B Q4).
  - DDR5 (~70 GB/s) $\to$ 14.5 tokens/sec (7B Q4).
  - Apple Unified Memory (~400 GB/s) $\to$ 83 tokens/sec (7B Q4).
- **Physical Core Rule**: Set CPU threads (`-t`) equal to **physical CPU cores**, NOT logical hyperthreads, to avoid cache thrashing.

---

## 📌 Topic 4: Tokenization Mechanics & Token Volume Dynamics

### 1. Model-Family Vocabulary Variations & VRAM Trade-Offs
- Tokenizers convert strings into discrete integer token IDs via BPE, Unigram, or SentencePiece. Byte-fallback mechanisms ($0\dots255$) guarantee 100% character and sequence coverage even for unknown characters.
- **Vocab Expansion Impact**: Llama 2 (32k) $\to$ Llama 3 (128k) $\to$ Qwen 2.5 (151k) $\to$ GPT-4o (200k) expanded vocabularies yield ~15–35% higher compression efficiency (fewer tokens per code/text block).
- **The Vocabulary Tax**: Embedding & `lm_head` weights scale as $2 \times (H \times V)$. Expanding $V$ from 32k to 151k on a 7B model ($H=4096$) increases static model weights by ~1 Billion parameters (~2.48 GB VRAM in FP16), trading static memory for faster generation speed ($O(N)$ fewer steps).
- **LLM Total Parameter Accounting Formula**: $\text{Params}_{\text{Total}} = 2VH + L(4H^2 + 3H d_{\text{ffn}}) + (2L+1)H$. Breakdown: FFN (SwiGLU) accounts for ~64% of total weights, Self-Attention (Q,K,V,O) for ~32%, Vocab Embeddings for 4–15%, and RMSNorms for ~0.01%.

### 2. FLOPs Math & Prefill vs. Generation Hardware Dynamics
- **$O(N^2)$ Self-Attention FLOPs**: Self-attention pairwise dot-products scale as $2 \times N^2 \times H$. Compressing sequence length $N$ by 33% (from 1,500 to 1,000 tokens for the exact same prompt text) reduces quadratic self-attention FLOPs by **55.5%**!
- **Prefill (Compute-Bound)**: All $N$ prompt tokens processed in parallel ($GEMM$). FLOPs savings directly reduce Time-To-First-Token (TTFT).
- **Generation (Memory-Bandwidth Bound)**: 1 token generated per step. Generating fewer total output tokens reduces sequential VRAM-to-SRAM weight fetches, raising tokens/sec throughput.

---

## 📌 Topic 5: Knowledge Graphs & GraphRAG — Mechanics, Incremental Ingestion & Schema Evolution

### 1. Underlying Mechanics & Triplet Abstractions
- **Semantic Triplets**: Fundamental unit of knowledge represented as $\text{(Subject Entity)} \xrightarrow{\text{Predicate / Relationship}} \text{(Object Entity)}$.
- **Property Graph (LPG) vs. RDF**: Labeled Property Graphs (e.g. **Kùzu**, Neo4j, NetworkX) store key-value property bags directly on nodes and edges. For Python AI backends, embedded columnar LPGs (like Kùzu) eliminate server daemons, run in-process with SQLite-level simplicity, and support openCypher.
- **Why Pure Vector RAG Fails**: Dense vector embeddings compute similarity over isolated text chunks, remaining blind to:
  1. *Multi-Hop Traversal* ($A \to B \to C$ across different documents).
  2. *Global Corpus Summarization* ("What are recurring penalty justifications across all 2024 cases?").
  3. *Exact Structured Aggregations* (counts, filters, date intervals).
- **GraphRAG (Hybrid Retrieval)**: Combines dense vector retrieval with graph community summaries (Leiden clustering) and multi-hop neighborhood extraction for holistic, grounded reasoning.

### 2. Incremental Ingestion & Real-Time Sync (Zero Full Rebuilds)
- **`MERGE` Idempotency**: Node and edge insertions execute parameterized `MERGE (n:LegalEntity {id: $id}) ON CREATE SET ... ON MATCH SET ...`.
- **Delta-Only Ingestion**: When a single new document (cause list or order PDF) arrives, only the new `HEARING` or `ORDER` node is created. Existing `CASE`, `JUDGE`, or `COUNSEL` nodes are reused without duplication.
- **Sub-100ms Ingestion**: Ingesting a single cause list or judgment takes under 100ms, making real-time hooks inside daily CLI commands (`sync-cause-lists`, `download-case-orders`) practical.
- **Batch Bootstrap vs. Incremental Sync**: `graph-sync` serves as a maintenance/recovery utility; daily operations update only incremental deltas.

### 3. Zero-Migration Schema Evolution & Backfilling
- **Elastic Storage Design**: Storing `relation_type: STRING` and `properties: STRING (JSON)` on the `RelatesTo` edge table allows introducing arbitrary new relationship types (e.g. `CO_COUNSEL_WITH`, `TAGGED_WITH_LEAD`, `MODIFIES_ORDER`) at any time without database migrations (`ALTER TABLE`).
- **3-Step Schema Expansion**:
  1. Add new relation name to `RelationType` Pydantic Literal in `schema.py`.
  2. Ingest with `store.insert_relation(LegalRelation(src, "NEW_REL", tgt))`.
  3. Query immediately in Cypher.
- **Safe Backfilling**: Re-running extractors on existing graph nodes uses `MERGE` to attach new edges without risk of duplicate nodes or loss of prior relationship history.

### 4. Temporal Graph Modeling (Cause List Case Study)
- **Hearing Chaining**: Modeling `HEARING` nodes and chronological `(Hearing_New)-[:FOLLOWS_HEARING {days_gap}]->(Hearing_Old)` edges creates an instant listing timeline for any case.
- **Enabled Operational Intelligence**:
  - *Case Listing Progression*: Exact hearing sequence with interval gaps.
  - *Previous/Next Hearing Resolution*: Instant calculation of prior hearing date and upcoming scheduled date.
  - *Daily Courtroom Boards*: All cases on a daily board ordered by item number.
  - *Counsel Clash Detection*: Detecting multi-courtroom scheduling conflicts for an advocate on the same morning.

### 5. Multi-Counsel & Multi-Party Extraction Mechanics & Scope Boundaries
- **The Issue (Row Wrap Flattening & Delimiter Mismatch)**:
  - In court cause list tables (NGT, High Courts), multiple advocate names in a cell are listed on separate lines (`\n`) or separated by semicolons and role indicators (*"for Applicant"*, *"for R-1"*, *"for MoEF&CC"*).
  - In naive parsing, table continuation lines joined with spaces (`" "`) and only split on commas (`,`), collapsing 5+ advocates into a single giant string.
- **The 3-Layer Deterministic Resolution**:
  1. *Row Wrap Linebreak Preservation*: Append wrapped table rows with `\n` instead of `" "` during `pdfplumber` cell extraction.
  2. *Multi-Delimiter Role Boundary Splitter*: Split on `\n`, `\r`, `;`, `,`, `&`, `along with`, and regex role qualifiers (`(?i)\bfor\s+(?:applicant|appellant|respondent|res|r-\d+|state|cpcb|moef|uoi)\b`).
  3. *Noise Sanitizer & Honorific Stripper*: Remove `Adv.`, `Mr.`, `Ms.`, `Dr.`, `Sh.`, `Smt.` and filter non-person artifacts (`Applicant in Person`, `None`, `-`).
- **Graph Scope Boundaries & Decision**:
  - *Co-Counsel Edges*: Preserved as a future backlog item; currently omitted to prevent graph density bloat.
  - *Multiple Parties & Respondents*: Extracted as clean, distinct `PARTY` entity nodes with their respective roles (`Applicant` vs `Respondent`). No artificial relations are assumed among co-respondents.

### 6. Predefined vs Dynamic Relationships in Property Graphs
- **Predefined Typed Schema (Strict Legal Ontology)**:
  - Domain-defined predicates (`LISTED_AT`, `PRESIDED_BY`, `REPRESENTS`, `PARTY_TO`, `FOLLOWS_HEARING`, `CITES_PRECEDENT`, `INVOKES_STATUTE`).
  - Enables deterministic, sub-millisecond local indexing with 0 token tax.
- **Dynamic / Discovered Schema (Zero-Migration Graph Engine)**:
  - Edge table stores `relation_type` as a string and JSON `properties`. Any new relationship discovered at runtime (e.g. `CONSTITUTED_COMMITTEE`, `DEPOSITED_COMPENSATION`) is ingested dynamically without schema migration or database rebuilds.

### 7. Multigraph / Multi-Edge Relationship Mechanics
- **The Domain Reality**: A single advocate can represent multiple different cases before the same bench/hearing session on the same morning (e.g. Item 5, Item 23, and Item 32 in Court 1).
- **Multi-Edge Deduplication Rule**: When upserting relationships, uniqueness checking must key on `(Source, Target, Relation, case_id)` rather than `(Source, Target, Relation)` alone to avoid overwriting prior case appearances for that hearing session.

### 8. Phase 3 Analytical Toolkit & CLI Capabilities
- **Operational Daily Board vs Historical Portfolio**:
  - *Daily Board (`graph-daily-board`, `graph-counsel-cases`)*: Queries direct `APPEARED_IN` edges on the specific hearing date to show only advocates assigned on that day.
  - *Historical Portfolio (`graph-counsel-portfolio`)*: Traverses persistent `(counsel)-[:REPRESENTS]->(case)` relations across all lifetime matters.
- **Judge Caseload Aggregator (`graph-judge-bench`)**: Aggregates hearing sessions, cases heard, and date breakdowns for a judge.
- **Precedent & Statute Traversal (`graph-precedents`)**: Explores multi-hop citations (`CITES_PRECEDENT`) and statutory provisions (`INVOKES_STATUTE`).
- **Interactive Cypher Runner (`graph-query`)**: Direct openCypher CLI execution.
- **Visual Graph Exporter (`graph-export`)**: Exports graph topology in JSON node-link, Graphviz DOT, or GEXF.

### 9. Phase 2: PDF Order Triplet Extraction & Precedent Citation Network
- **The Challenge**: Cause lists only capture operational hearing schedules. Binding ratios, statutory provisions, and landmark Supreme Court citations live in unstructured order PDFs.
- **Deterministic Regex Layer ($0$ Token Tax)**:
  - `StatuteParser` extracts Indian environmental acts (`NGT Act 2010`, `Water Act 1974`, `Air Act 1981`, `Environment Protection Act 1986`, `Solid Waste Management Rules 2016`, `Constitution Article 21`) including compound section references (e.g. *Section 14 and 15 of NGT Act*).
  - `NGTOrderParser` extracts coram judges (`CORAM: HON'BLE MR. JUSTICE...`), hearing dates, and Supreme Court citations (`Vellore Citizens Forum (1996) 5 SCC 647`, `M.C. Mehta v. Kamal Nath (1997) 1 SCC 388`, `T.N. Godavarman (1997) 1 SCC 1`).
- **Graph Linkage**:
  - Ingests `(Case)-[:INVOKES_STATUTE]->(Section)`
  - Ingests `(Case)-[:CITES_PRECEDENT]->(Precedent_Case)`
  - Ingests `(Case)-[:DELIVERED_BY]->(Judge)`
- **CLI Commands**:
  - `graph-extract-order <pdf_path> [--ingest]`
  - `graph-sync-orders [--dir <orders_dir>]`

### 10. Phase 4: FastAPI Graph Bridge & REST Service Layer
- **The Architecture**:
  - Exposes the embedded columnar Kùzu Knowledge Graph via high-speed, asynchronous HTTP REST endpoints.
  - Lifecycle management via FastAPI `lifespan` guarantees a single shared, thread-safe `LegalGraphStore` instance per worker process.
  - Full CORS enabling (`allow_origins=["*"]`) for web dashboards, visualizers (Cytoscape.js), and Open-NotebookLM GraphRAG backends.
- **REST Endpoints Architecture**:
  - `GET /health` & `GET /`: Health check and API discovery.
  - `GET /api/graph/stats`: Real-time node and relationship metrics.
  - `GET /api/graph/daily-board`: Courtroom cause list board with item numbers, full counsels, and judges.
  - `GET /api/graph/counsel/{name}/cases`: Date-range appearance schedule.
  - `GET /api/graph/counsel/{name}/clashes`: Multi-courtroom clash detection.
  - `GET /api/graph/counsel/{name}/portfolio`: Lifetime counsel representation analytics.
  - `GET /api/graph/judge/{name}/caseload`: Judge bench caseload and hearings.
  - `GET /api/graph/case/{case_id}/timeline`: Chronological listing timeline with interval gaps.
  - `GET /api/graph/case/{case_id}/precedents`: Multi-hop precedent citations and statutory sections.
  - `POST /api/graph/query`: Raw openCypher query executor (`{"query": "MATCH (n) RETURN n"}`).
  - `GET /api/graph/export`: Full graph topology export (JSON node-link, DOT, GEXF).
  - `POST /api/orders/extract-file` & `POST /api/orders/extract-path`: Order PDF parsing and triplet extraction.
  - `POST /api/orders/sync`: Batch order directory synchronization.
### 11. Phase 5: Hybrid GraphRAG Retriever (Vector + Kùzu Graph Expansion)
- **The Problem Solved**:
  - Semantic vector search retrieves textual similarity but misses multi-hop precedent citations and statutory networks.
  - Frontier LLMs without structured ontological grounding hallucinate penalty provisions and misattribute judgments.
- **Dual-Channel Retrieval Architecture**:
  1. *Dense/Sparse Passage Retrieval*: Semantic TF-IDF + cosine search across judicial order paragraph chunks.
  2. *Multi-Hop Graph Expansion*: Resolves candidate case IDs in Kùzu DB and traverses 1-to-2 hop neighborhoods (`INVOKES_STATUTE`, `CITES_PRECEDENT`, `DELIVERED_BY`).
  3. *Grounded Context Assembly*: Generates a markdown-structured context block with verified statutory provisions, landmark Supreme Court citations, coram bench members, and text passages.
  4. *Rule 11 Supervisory Wrapper*: Core retrieval and grounded context assembly is 100% deterministic ($0$ LLM token tax), with optional supervisory LLM synthesis.
- **REST Endpoints & CLI**:
  - `POST /api/rag/retrieve`: Hybrid passage retrieval + graph expansion.
  - `POST /api/rag/ask`: Grounded legal question answering.
  - `python projects/LawNidhi/cli.py graph-rag "<query>" [--top-k 3] [--synthesize]`

### 12. Phase 6: Hierarchical Graph Summarization & Interactive Web UI
- **Unsupervised Community Detection**:
  - Employs NetworkX modularity-based greedy partitioning (`GraphClusterEngine`) to detect cohesive legal sub-networks (macro thematic clusters) across 918+ nodes.
  - Computes degree centrality, hub nodes, dominant entity types, and statutory concentrations per community.
- **Modern Dark-Mode Web Dashboard**:
  - Built with Vanilla JS & custom glassmorphism design system (`Inter`, `Outfit`, `JetBrains Mono`).
  - **Interactive Cytoscape.js Canvas**: Node coloring by ontology type, force-directed COSE physics, search & auto-zoom, and slide-out node inspector drawer.
  - **Live Daily Courtroom Board**: Cause list browsing with courtroom filters and advocate highlights.
  - **Hybrid GraphRAG Search Interface**: Live query portal returning grounded synthesis and Knowledge Graph citations side-by-side.
  - **Thematic Cluster Explorer**: Macro-community cards with member counts and statutory focus.
- **Service Integration**:
  - Mounted directly in FastAPI at `/ui` and `/` (`http://localhost:8000/ui`).
  - CLI: `python projects/LawNidhi/cli.py graph-communities [--min-size 2]`

### 13. Phase 7: Agentic Legal Co-Counsel (Autonomous ReAct Legal Researcher)
- **Architecture & Workspace Rules**:
  - Strict ReAct loop safety limit (`max_loops <= 12`, Rule 1) preventing token burn.
  - Deterministic Tool Registry (Rule 11) interfacing with Kùzu DB, Precedent lineager, Hybrid GraphRAG, and Counsel clash detector.
  - Generates structured Case Briefs (`CaseBrief` Pydantic model) and autonomous research opinions.
- **Service & UI Integration**:
  - REST API: `POST /api/agent/chat` and `POST /api/agent/brief`.
  - Web UI: Dedicated **Legal Co-Counsel** tab in the Single-Page Application displaying live step-by-step reasoning trajectories alongside final briefs.
  - CLI: `python projects/LawNidhi/cli.py ask "<query>" [--max-loops 10] [--verbose]`

---

## 📚 Related Workspace Documentation Links
- **[knowledge_graph_guide.md](file:///home/navin/work/AI/docs/guides/knowledge_graph_guide.md)** — Comprehensive guide on Knowledge Graphs, GraphRAG architecture, schemas, and workspace project applicability.
- **[ai_glossary_and_definitions.md](file:///home/navin/work/AI/docs/ai_glossary_and_definitions.md)** — Master technical glossary & definitions (FLOPs, Prefill/Generation, VRAM math, MoE, BPE, ReAct, SOPs).
- **[model_landscape_and_comparison_guide.md](file:///home/navin/work/AI/docs/guides/model_landscape_and_comparison_guide.md)** — Comprehensive LLM & SLM model features, parameter scaling, MoE vs Dense, and ecosystem comparison.
- **[slm_architecture_and_usage_guide.md](file:///home/navin/work/AI/docs/guides/slm_architecture_and_usage_guide.md)** — Comprehensive SLM guide covering architecture, VRAM formulas, CPU dynamics, and code implementation.
- **[ai_ecosystem_concepts.md](file:///home/navin/work/AI/docs/ai_ecosystem_concepts.md#L281-L297)** — Section H: Small Language Models (SLMs) & Edge Inference.
- **[model_routing_guide.md](file:///home/navin/work/AI/docs/guides/model_routing_guide.md)** — Model routing strategies combining SLMs and frontier LLMs.
- **[structured_outputs_guide.md](file:///home/navin/work/AI/docs/guides/structured_outputs_guide.md)** — Schema enforcement & GBNF grammar-guided decoding.


