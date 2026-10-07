# 🧭 AI & Agentic Topics: Master Revision Index

A clean, hierarchical quick-revision checklist of core topics and subtopics (max 1–2 levels). Use this index for rapid review, knowledge auditing, and tracking active vs. upcoming AI and agentic learning domains.

*(For systems engineering, distributed systems, and backend topics, see [system_architecture_topics_index.md](file:///home/navin/work/AI/docs/system_architecture_topics_index.md)).*

---

## 🤖 1. Agent Frameworks & Multi-Agent Systems

### CrewAI
- **Agent**: Role, Goal, Backstory, Tools, LLM config, `allow_delegation`, `max_iter`
- **Task**: Description, Expected Output, Agent binding, Context dependencies, `output_pydantic`
- **Crew**: Agent pool, Task queue, Verbosity, Cache, Token usage metrics
- **Process**: `Process.sequential`, `Process.hierarchical` (Manager Agent)
- **Tools**: `@tool` decorator, `BaseTool`, Input Schema (`args_schema`)

### LangGraph
- **Core Primitives**: `StateGraph`, `State` (`TypedDict` / Pydantic), `START`, `END`
- **Nodes**: Pure update functions `(state) -> dict`
- **Edges**: Normal edges (`add_edge`), Conditional edges (`add_conditional_edges`)
- **Reducers**: State merging (e.g. `Annotated[list, add]`, `operator.add`)
- **Checkpointers & Persistence**: `MemorySaver`, `SqliteSaver`, `PostgresSaver`
- **Human-in-the-loop**: Breakpoints (`interrupt_before`, `interrupt_after`), State rewind & fork

### AutoGen (Microsoft)
- **Agents**: `ConversableAgent`, `UserProxyAgent`, `AssistantAgent`
- **Orchestration**: Two-agent chat, `GroupChat`, `GroupChatManager`
- **Code Execution**: Local command line, Docker sandbox execution

### Next Candidates (Multi-Agent & Frameworks)
> *PydanticAI, smolagents (Hugging Face), Letta / MemGPT, LlamaIndex Workflows, Semantic Kernel*

---

## ⛓️ 2. Core Orchestration & Application Layers

### LangChain
- **LCEL (Expression Language)**: Pipe syntax (`prompt | model | parser`)
- **Runnables**: `RunnableLambda`, `RunnableParallel`, `RunnablePassthrough`, `RunnableBranch`
- **Prompts**: `ChatPromptTemplate`, `MessagesPlaceholder`, Few-shot prompt templates
- **Output Parsers**: `PydanticOutputParser`, `StrOutputParser`, `JsonOutputParser`
- **Chains**: RetrievalQA, HistoryAwareRetriever, Document loaders

### Protocol & Standards
- **Model Context Protocol (MCP)**: MCP Host, MCP Client, MCP Server, Resources, Tools, Prompts
- **OpenAI Tool Calling Spec**: Function definition schema, JSON Schema validation, Tool call resolution

### Next Candidates (Orchestration & Control)
> *DSPy (compiled prompt signatures), Outlines (structured regex/grammar generation), Guidance, Instructor*

---

## 🔍 3. Observability, Evaluation & Tracing

### LangSmith
- **Tracing**: Run tree, Latency breakdown, Token usage per node/span
- **Datasets**: Few-shot collections, Golden test sets
- **Evaluators**: Correctness, Groundedness, Hallucination, RAG Triad
- **Feedback & Annotations**: Thumbs up/down, Human-in-the-loop tags

### LLM Tracing & Evaluation Tools
- **Arize Phoenix**: OpenTelemetry traces, Evals, Embedding drift analysis
- **Langfuse**: Open-source tracing, Prompt versioning, Latency monitoring
- **Ragas**: Faithfulness, Answer Relevance, Context Recall, Context Precision
- **DeepEval**: LLM-as-a-Judge metric test suites, Unit-test integration

### Next Candidates (Observability & Ops)
> *OpenTelemetry (OTel Semantic Conventions for GenAI), TruLens, Weave (W&B), Promptfoo*

---

## 🕸️ 4. Graph Databases & Knowledge Graph RAG (GraphRAG)

### Graph Databases
- **Neo4j**:
  - Cypher query language (`MATCH`, `MERGE`, `WHERE`, `RETURN`)
  - APOC library & procedures
  - Graph Data Science (GDS): PageRank, Node2Vec, Community Detection
- **Kùzu DB**:
  - Embedded columnar graph engine (DuckDB-style)
  - Cypher support, Structured vertex & edge tables
- **Alternative Graph Engines**:
  - Memgraph (In-memory, Cypher-compatible)
  - FalkorDB (Redis-based Graph Engine)
  - Amazon Neptune (Managed AWS property graph & RDF)
  - Oxigraph (Embedded SPARQL / RDF triple store)

### GraphRAG
- **Extraction**: Entity-Relation-Entity extraction (Triples: Head, Relation, Tail)
- **Entity Resolution**: Deduplication, Synonym merging, Coreference resolution
- **Hierarchical Summarization**: Microsoft GraphRAG pattern, Leiden community detection
- **Query Types**: Local Search (entity-centric), Global Search (corpus-wide macro summaries)

### Next Candidates (Graph & Semantics)
- Ontotext GraphDB, Apache Jena, Neo4j Vector Index, RDF/OWL Ontologies

---

## 📚 5. Vector Search, Retrieval & Semantic Cache

### Vector Stores
- **Qdrant**: HNSW, Payload filtering, Fast hybrid indexing
- **ChromaDB**: Lightweight embedded vector store
- **pgvector**: PostgreSQL vector extension, IVFFlat / HNSW indexes
- **LanceDB**: Serverless columnar vector store, zero-copy reads

### Search & Reranking Techniques
- **Dense Retrieval**: Vector cosine similarity, Dot product
- **Sparse Retrieval**: BM25, TF-IDF, SPLADE (learned sparse embeddings)
- **Hybrid Search**: Reciprocal Rank Fusion (RRF), Linear score combination
- **Reranking**: Cross-encoders, Cohere Rerank, BGE-Reranker
- **Semantic Caching**: GPTCache, Embedding threshold routing, Exact vs Fuzzy hit

---

## 🎙️ 6. Voice, Speech & Multimodal Audio

### Speech-to-Text (STT)
- **Whisper Ecosystem**: Faster-Whisper (CTranslate2), WhisperX (word alignment)
- **Hosted / Edge STT**: Deepgram Nova-2, Hugging Face Serverless Router
- **Voice Activity Detection (VAD)**: Silero VAD, WebRTC VAD

### Text-to-Speech (TTS)
- **Local Neural TTS**: Kokoro (82M ultra-fast TTS), XTTS-v2 (Voice cloning), Piper
- **Lightweight Fallbacks**: gTTS, Pyttsx3
- **Hosted Voice APIs**: ElevenLabs, Cartesia, OpenAI TTS

### Real-Time Streaming Audio
- **Protocols**: WebSockets, WebRTC, Server-Sent Events (SSE)
- **Audio Chunks**: PCM 16-bit 16kHz, Opus, WAV chunking, FFmpeg stream piping

---

## 🧠 7. Prompt Patterns, Reasoning & Agent Architectures

### Agent Reasoning Patterns
- **ReAct**: Interleaved Reasoning + Action loop
- **Plan-and-Solve**: Upfront goal decomposition $\to$ Stepwise execution
- **Reflection / Self-Correction**: Critic agent, Verification pass, Self-RAG
- **Router Pattern**: Semantic Router (Embedding distance), Classifier LLM, Regex/Keyword cache

### Memory Architectures
- **Short-Term Buffer**: Sliding conversation window, Summary memory
- **Long-Term Memory**:
  - Episodic Memory (Past interactions, conversation logs)
  - Semantic Memory (User facts, profile attributes)
  - Procedural Memory (Learned workflows, past tool execution traces)

---

## 🚀 8. Next Candidates & Frontier Topics (Backlog Pool)

A quick comma-separated list of topics to explore next:

> **Serving & Inference Engines**: vLLM, TensorRT-LLM, SGLang, Ollama, Llama.cpp, TGI  
> **Model Architectures**: Mixture of Experts (MoE), State Space Models (Mamba), Speculative Decoding  
> **Alignment & Fine-Tuning**: LoRA / QLoRA, DPO (Direct Preference Optimization), PPO, Unsloth, Axolotl  
> **Agent Sandboxes & Execution**: E2B Sandboxes, Docker microVMs, Modal, Fly.io edge workers  
> **Structured Generation**: Jsonformer, Guidance, Outlines, Instructor  
> **Multi-Modal Frontiers**: Vision-Language-Action (VLA), Omni models, Whisper-large-v3-turbo  
