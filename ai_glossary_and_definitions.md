# AI & LLM Technical Glossary & Key Definitions

This document serves as a centralized, easy-to-study reference guide for core technical terms, mathematical concepts, hardware metrics, architectural patterns, and execution dynamics across Artificial Intelligence, Large Language Models (LLMs), Small Language Models (SLMs), and Agentic Systems.

---

## 🔤 Master Category Directory

- [1. Core Compute & Hardware Metrics](#1-core-compute--hardware-metrics)
- [2. Model Architecture & Parameter Dynamics](#2-model-architecture--parameter-dynamics)
- [3. Tokenization & Context Window Mechanics](#3-tokenization--context-window-mechanics)
- [4. Inference & Execution Phases](#4-inference--execution-phases)
- [5. Agentic AI & Systems Terminology](#5-agentic-ai--systems-terminology)
- [6. Alphabetical Quick Reference Table](#6-alphabetical-quick-reference-table)

---

## 1. Core Compute & Hardware Metrics

### FLOPs (Floating-Point Operations)
- **Definition:** The total number of individual floating-point mathematical calculations (multiplications and additions) required to perform a forward pass through a neural network layer.
- **Why It Matters:** Indicates raw computational work. Self-attention layers scale quadratically ($O(N^2)$) in FLOPs relative to prompt token sequence length $N$.
- **TFLOPS / PFLOPS:** TeraFLOPs ($10^{12}$ operations per second) or PetaFLOPs ($10^{15}$ operations per second), measuring hardware compute speed.

### Compute-Bound vs. Memory-Bandwidth Bound
- **Compute-Bound:** An execution phase where the bottleneck is the GPU's raw arithmetic processing capability (Tensor Cores). Occurs during the **Prefill Phase** when processing all prompt tokens simultaneously ($GEMM$ matrix multiplication).
- **Memory-Bandwidth Bound:** An execution phase where the bottleneck is the rate at which model weights can be read from GPU VRAM into compute registers (GB/s or TB/s). Occurs during the sequential **Generation Phase** (1 output token per step).

### VRAM (Video RAM) & High Bandwidth Memory (HBM)
- **Definition:** Dedicated high-speed GPU memory used to store model weights, active $KV$ caches, intermediate activation tensors, and CUDA buffers.
- **Formula for Weight Footprint:** $\text{VRAM}_{\text{Weights}} = P \times b \times 1.15$ (where $P$ is billions of parameters and $b$ is precision size in bytes, e.g., 2 bytes for FP16, 0.55 bytes for 4-bit Q4_K_M).

### SRAM (Static RAM / GPU L2 Cache)
- **Definition:** Ultra-fast, low-capacity cache located directly inside the GPU silicon next to Tensor Cores. Used to store active weights during matrix multiplications.

### TTFT (Time-To-First-Token)
- **Definition:** The total latency (in milliseconds) from when a user submits a prompt until the model outputs its very first token.
- **Key Determinant:** Directly driven by the prefill phase latency and prompt token count $N$.

### Throughput (Tokens per Second - t/s)
- **Definition:** The rate at which the model generates text completion tokens after the first token has been produced. Driven by VRAM memory bandwidth.

---

## 2. Model Architecture & Parameter Dynamics

### Transformer Model Classifications: Functional Types
Modern neural models are broadly categorized into **5 primary architectural types** based on attention mechanism and operational objective:

1. **Decoder-Only Models (e.g., Llama 3, Gemini, GPT-4o, DeepSeek, Qwen):**
   - *Attention:* Causal / Autoregressive masking (Token $N$ attends only to past tokens $1\dots N-1$).
   - *Output:* Autoregressive next-token generation (text, code, reasoning `<think>` traces).
   - *Use Case:* Conversational agents, reasoning, multi-step code generation, tool calling.
2. **Encoder-Only Models (e.g., BERT, RoBERTa, DeBERTa):**
   - *Attention:* Bi-directional (Token $N$ attends to all tokens before and after it simultaneously).
   - *Output:* Contextual representations, classification logits, or sequence labels.
   - *Use Case:* Text classification, Named Entity Recognition (NER), sentiment analysis.
3. **Encoder-Decoder / Seq2Seq Models (e.g., T5, BART, Whisper):**
   - *Attention:* Bi-directional Encoder processes input; Causal Decoder generates output via cross-attention.
   - *Output:* Sequence-to-sequence translation or transcription.
   - *Use Case:* Machine translation, abstractive summarization, audio speech-to-text.
4. **Embedding Models (e.g., OpenAI `text-embedding-3`, BGE-M3, Nomic-Embed):**
   - *Mechanism:* Specialized Encoders or Decoders with mean/CLS pooling fine-tuned using Contrastive Loss.
   - *Output:* High-dimensional dense vectors (e.g., 768 or 1536 floats representing semantic meaning).
   - *Use Case:* RAG vector search, document similarity, semantic clustering.
5. **Multimodal / Vision-Language Models (LMMs) (e.g., Gemini 1.5/3.6, Claude 3.5, Qwen2-VL):**
   - *Mechanism:* Cross-modal projectors combining Vision/Audio Encoders (SigLIP/Whisper) with LLM Decoder backbones.
   - *Output:* Multimodal comprehension and text/action generation from images, video, PDFs, or audio.

### Dense Architecture vs. Mixture-of-Experts (MoE)
- **Dense Architecture:** Every model weight parameter is active and evaluated during the forward pass for every single token (e.g., Llama 3.3 70B runs all 70B parameters per token).
- **Mixture-of-Experts (MoE):** Utilizes router layers to dynamically direct tokens to a small subset of specialized sub-networks ("experts"). Example: DeepSeek V3 has **671B total parameters**, but only **37B active parameters** are used per token.

### Active Parameters vs. Total Parameters
- **Total Parameters:** The aggregate size of all weights stored in model memory.
- **Active Parameters:** The subset of weights actually computed during a single token's forward pass. In MoE models, Active Parameters dictate inference FLOPs, while Total Parameters dictate VRAM footprint.

### Hidden Dimension ($H$) & Layer Count ($L$)
- **Hidden Dimension ($H$):** The vector embedding size representing token state inside the Transformer (e.g., $H = 4096$ in 7B models, $H = 8192$ in 70B models).
- **Layer Count ($L$):** The number of stacked Transformer blocks (e.g., 32 layers in 7B models, 80 layers in 70B models).

### LLM Parameter Accounting: What Adds Up to the Total Parameter Count?
A standard decoder-only LLM (like Llama, Qwen, or Mistral) builds its total parameters from **3 distinct component layers**:

1. **Vocabulary Embedding Layers (Input & Output):**
   - **Input Embedding (`tok_embeddings`):** Maps token IDs ($0\dots V-1$) to hidden vectors $[V \times H] \implies V \cdot H$ parameters.
   - **Output Projection (`lm_head`):** Maps hidden vectors back to logits $[H \times V] \implies H \cdot V$ parameters.
   - *Subtotal:* $\text{Params}_{\text{Vocab}} = 2 \cdot (V \cdot H)$

2. **The $L$ Stacked Transformer Layers (Repeated $L$ Times):**
   - **Multi-Head Self-Attention (Q, K, V, O Projections):**
     - $W_Q, W_K, W_V, W_O$ matrices $\implies \approx 4 \cdot H^2$ parameters per layer.
   - **Feed-Forward Network (SwiGLU MLP):**
     - $W_{\text{gate}}, W_{\text{up}}, W_{\text{down}}$ matrices $\implies 3 \cdot (H \cdot d_{\text{ffn}}) \approx 8 \cdot H^2$ parameters per layer (where $d_{\text{ffn}} \approx \frac{8}{3} H$).
   - *Subtotal per Layer:* $\approx 12 \cdot H^2$ parameters.
   - *Subtotal for $L$ Layers:* $\text{Params}_{\text{Layers}} = L \cdot \left(4 H^2 + 3 H d_{\text{ffn}}\right) \approx 12 \cdot L \cdot H^2$

3. **Normalization Scales (RMSNorm):**
   - 2 scale vectors per layer plus final norm $\implies (2L + 1) \cdot H$ parameters (~0.01% of total).

$$\mathbf{\text{Grand Total Params}} = \underbrace{2 \cdot V \cdot H}_{\text{Vocab Embeddings}} + \underbrace{L \cdot \left(4 H^2 + 3 H d_{\text{ffn}}\right)}_{\text{Stacked Layers (Attention + FFN)}} + \underbrace{(2L + 1) \cdot H}_{\text{Layer Norms}}$$

*Example (7B Model, e.g. Llama 2 7B: $V=32k, H=4096, L=32, d_{\text{ffn}}=11008$):*
- Vocab: $0.26\text{B}$ + Attention: $2.15\text{B}$ + FFN: $4.33\text{B}$ + Norms: $0.0002\text{B} = \mathbf{6.74\text{ Billion Parameters}}$.

### Quantization (FP16, INT8, Q4_K_M, GGUF, AWQ)
- **Definition:** Compressing neural network weights by reducing numerical precision (e.g., converting 16-bit floating point FP16 to 4-bit integer Q4).
- **Benefits:** Cuts VRAM footprint by 60–75% and reduces memory bandwidth requirements, enabling 7B–14B models to run locally on consumer GPUs and CPUs.
- **In-Depth Guide:** For complete mathematical foundations, algorithms, and conversion workflows, see [model_quantization_guide.md](file:///home/navin/work/AI/docs/guides/model_quantization_guide.md).


### Distillation
- **Definition:** Training a smaller "student" model (e.g., Qwen 7B) using outputs, probability distributions, or reasoning chains generated by a larger "teacher" model (e.g., DeepSeek R1).

---

## 3. Tokenization & Context Window Mechanics

### Token & Tokenizer
- **Token:** The discrete integer ID representing a sub-word, character, or syntax chunk processed by neural networks.
- **Tokenizer:** The software algorithm (BPE, SentencePiece, Tiktoken) that splits raw text strings into token IDs and vice versa.

### Vocabulary Size ($V$)
- **Definition:** The total dictionary size of unique tokens supported by a model's tokenizer (e.g. 32k in Llama 2, 128k in Llama 3, 151k in Qwen 2.5, 200k in GPT-4o).
- **The Vocabulary Tax Formula:** $\text{Params}_{\text{Vocab}} = 2 \times (H \times V)$ (where $H$ is hidden dimension and $V$ is vocabulary size).
- **Why the factor 2?** The vocabulary parameters are split across **two separate matrices**:
  1. **Input Embedding Matrix (`tok_embeddings`):** Shape $[V \times H] \to V \times H$ parameters (converts input token IDs into hidden vectors).
  2. **Output Unembedding Matrix (`lm_head`):** Shape $[H \times V] \to H \times V$ parameters (projects hidden vectors back into token logits).
  - *Total Parameters:* $(V \times H) + (H \times V) = \mathbf{2 \times (H \times V)}$. *(Note: Models using "Weight Tying" share these matrices, reducing the factor to 1).*

### Byte-Fallback
- **Definition:** A tokenizer mechanism that encodes any unlisted or rare Unicode character as raw UTF-8 byte tokens ($0\dots255$), guaranteeing **100% string coverage** without crashing.

### Context Window & Needle-in-a-Haystack (NIAH)
- **Context Window:** The maximum total token sequence length ($N_{\text{prompt}} + N_{\text{completion}}$) a model can accept in a single session (e.g., 128K to 2M+ tokens).
- **Needle-in-a-Haystack (NIAH):** A benchmark testing a model's ability to retrieve a specific isolated fact embedded deep within a massive context window.

### Prompt Caching / Prefix Caching
- **Definition:** Storing pre-computed Key-Value ($KV$) attention states in memory for exact prompt prefixes. Eliminates redundant prefill FLOPs, cutting TTFT and billing costs by 50–90%.

---

## 4. Inference & Execution Phases

### Prefill Phase (Prompt Ingestion)
- **Definition:** The initial pass where all input prompt tokens ($N$) are processed simultaneously in parallel.
- **Math & Hardware:** Compute-bound $GEMM$ operations scaling quadratically ($O(N^2)$) in self-attention FLOPs.

### Generation Phase (Autoregressive Completion)
- **Definition:** The sequential step-by-step token generation loop ($1 \dots N_{\text{out}}$).
- **Math & Hardware:** Memory-bandwidth bound. Each generated token requires reading all model weights from VRAM into compute units.

### Self-Attention & $O(N^2)$ Scaling
- **Definition:** The Transformer mechanism where every token computes pairwise dot-product alignment scores against all other tokens in the sequence.
- **Quadratic Formula:** Attention FLOPs scale as $2 \times N^2 \times H$. Compressing prompt length $N$ by 33% cuts quadratic attention FLOPs by 55.5%.

### Inference-Time Compute Scaling (`<think>` Reasoning Tokens)
- **Definition:** Allowing a model (e.g., DeepSeek R1, OpenAI o1) to generate intermediate hidden reasoning steps (`<think>...</think>`) before returning a final response, trading extra generation FLOPs for higher accuracy on complex math and code tasks.

### KV Cache (Key-Value Cache)
- **Definition:** Memory storage that preserves calculated Key ($K$) and Value ($V$) attention tensors across turns, avoiding redundant matrix multiplication during multi-turn chats.

---

## 5. Agentic AI & Systems Terminology

### ReAct Loop (Reasoning + Acting)
- **Definition:** An agent execution pattern that alternates between **Thought** (reasoning), **Action** (invoking a sandboxed tool), and **Observation** (reading tool output) in a loop until a goal is met.

### Declarative SOP (Standard Operating Procedure)
- **Definition:** External markdown/YAML files defining behavioral rules and step-by-step guidelines that are dynamically injected into an agent's prompt context based on task intent.

### Grammar-Guided Decoding (GBNF / JSON Schema)
- **Definition:** Middleware (vLLM, llama.cpp) that constrains logit sampling at runtime to guarantee 100% compliance with strict JSON or type schemas.

### Model Context Protocol (MCP)
- **Definition:** An open standard ("USB-C for AI") enabling agents to connect seamlessly to external tools, databases, and local file systems via uniform schemas.

### RAG (Retrieval-Augmented Generation)
- **Definition:** Dynamically retrieving relevant document chunks or SOP guidelines from a vector/text index and injecting them into the prompt before model generation.

---

## 6. Alphabetical Quick Reference Table

| Term | Category | Key Formula / Definition | Primary Impact |
| :--- | :--- | :--- | :--- |
| **BPE** | Tokenization | Byte-Pair Encoding algorithm | Sub-word segmentation |
| **Compute-Bound** | Hardware | Bottlenecked by TFLOPS capacity | Governs Prefill Phase speed |
| **Dense Model** | Architecture | 100% weights active per token | High VRAM & predictable latency |
| **FLOPs** | Compute | $2 \times N^2 \times H$ (Self-attention) | Measures arithmetic work |
| **GBNF** | Inference | Backus-Naur Form grammar constraint | Guarantees structured JSON |
| **KV Cache** | Memory | Stores $K, V$ attention tensors | Prevents re-computing past turns |
| **Memory-Bound** | Hardware | Bottlenecked by VRAM bandwidth | Governs Generation speed (t/s) |
| **MoE Model** | Architecture | Dynamic routing to active experts | Low per-token compute cost |
| **NIAH** | Context | Needle-In-A-Haystack benchmark | Tests long-context recall accuracy |
| **Prefix Caching** | Memory | Reuses pre-computed $KV$ states | 50–90% cost & latency reduction |
| **Prefill Phase** | Inference | Parallel processing of input prompt | Computes $O(N^2)$ self-attention |
| **ReAct** | Agentic AI | Reasoning $\to$ Action $\to$ Observation | Foundational agent execution loop |
| **SOP** | Agentic AI | Standard Operating Procedure | Declarative prompt behavioral guide |
| **TTFT** | Performance | Time-To-First-Token latency | Initial user responsiveness metric |
| **Vocabulary Size ($V$)** | Tokenization | Total unique tokens in dictionary | Larger $V$ = Fewer tokens per prompt |
