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

## 📚 Related Workspace Documentation Links
- **[ai_glossary_and_definitions.md](file:///home/navin/work/AI/docs/ai_glossary_and_definitions.md)** — Master technical glossary & definitions (FLOPs, Prefill/Generation, VRAM math, MoE, BPE, ReAct, SOPs).
- **[model_landscape_and_comparison_guide.md](file:///home/navin/work/AI/docs/guides/model_landscape_and_comparison_guide.md)** — Comprehensive LLM & SLM model features, parameter scaling, MoE vs Dense, and ecosystem comparison.
- **[slm_architecture_and_usage_guide.md](file:///home/navin/work/AI/docs/guides/slm_architecture_and_usage_guide.md)** — Comprehensive SLM guide covering architecture, VRAM formulas, CPU dynamics, and code implementation.
- **[ai_ecosystem_concepts.md](file:///home/navin/work/AI/docs/ai_ecosystem_concepts.md#L281-L297)** — Section H: Small Language Models (SLMs) & Edge Inference.
- **[model_routing_guide.md](file:///home/navin/work/AI/docs/guides/model_routing_guide.md)** — Model routing strategies combining SLMs and frontier LLMs.
- **[structured_outputs_guide.md](file:///home/navin/work/AI/docs/guides/structured_outputs_guide.md)** — Schema enforcement & GBNF grammar-guided decoding.

