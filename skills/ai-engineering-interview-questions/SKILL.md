```markdown
---
name: ai-engineering-interview-questions
description: Comprehensive cheat sheet and study guide for AI Engineering interview questions covering LLMs, RAG, agents, fine-tuning, quantization, and more.
triggers:
  - help me prepare for an AI engineering interview
  - what are common LLM interview questions
  - explain RAG interview topics
  - AI agent interview preparation
  - fine-tuning and quantization interview questions
  - LLMOps interview questions and answers
  - prompt engineering interview prep
  - vector database interview questions
---

# AI Engineering Interview Questions Skill

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

## What This Project Does

[ai-engineering-interview-questions](https://github.com/amitshekhariitbhu/ai-engineering-interview-questions) is a curated, community-maintained cheat sheet of AI Engineering interview questions and answers. It covers all major topics an AI/ML/LLM engineer needs for interviews at AI-focused companies, targeting roles such as:

- AI Engineer / Gen AI Engineer
- LLM Engineer / Agentic AI Engineer
- AI Solutions Architect
- MLOps / LLMOps Engineer
- Applied AI Engineer

Topics span: LLM Fundamentals, Prompt Engineering, RAG, AI Agents, Fine-Tuning, Vector Databases, AI System Design, LLMOps, Evaluation, AI Safety, Multi-Modal AI, and Infrastructure.

---

## Installation / Setup

This is a Markdown reference repository — no installation required. Clone or bookmark it for study.

```bash
# Clone the repo locally for offline study
git clone https://github.com/amitshekhariitbhu/ai-engineering-interview-questions.git
cd ai-engineering-interview-questions

# Browse the main README
cat README.md

# Or open in your editor
code README.md
```

---

## Topic Coverage Map

Use this map to navigate interview prep by role focus:

| Role Focus | Key Sections |
|---|---|
| LLM Engineer | LLM Fundamentals, Prompt Engineering, Fine-Tuning |
| RAG Engineer | RAG, Vector Databases & Embeddings, AI System Design |
| Agentic AI Engineer | AI Agents, MCP, Prompt Engineering (ReAct) |
| MLOps/LLMOps | LLMOps and Production AI, AI Infrastructure |
| Applied AI / Full-Stack | All sections + Coding & Practical Implementation |

---

## Core Concept Summaries

### LLM Fundamentals

```
Key Concepts:
- Transformer architecture: encoder-only, decoder-only, encoder-decoder
- Self-attention: Q (Query), K (Key), V (Value) matrices
- Multi-head attention vs Grouped-Query Attention (GQA)
- Tokenization: BPE, WordPiece, SentencePiece
- Positional encoding (absolute, learned, RoPE)
- KV Cache: speeds up autoregressive inference by caching past K/V
- Mixture of Experts (MoE): sparse routing to expert sub-networks
- Flash Attention: memory-efficient attention with IO-aware tiling
- Context window: maximum tokens the model can process at once
- Temperature, Top-k, Top-p: controls for text generation randomness
```

**Quick answer — Self-Attention:**
```
Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) * V

- Q, K, V are learned linear projections of the input
- Scaled dot-product prevents vanishing gradients in softmax
- Multi-head: run H attention heads in parallel, concatenate outputs
```

**Quick answer — KV Cache:**
```
During autoregressive generation:
- Without cache: recompute K, V for ALL previous tokens each step → O(n²)
- With KV cache: store K, V for previous tokens, only compute new token → O(n)
- Trade-off: memory grows linearly with sequence length
```

---

### RAG (Retrieval-Augmented Generation)

**Architecture:**
```
[User Query]
     │
     ▼
[Embedding Model] → Query Vector
     │
     ▼
[Vector DB Search] → Top-K Chunks
     │
     ▼
[Re-ranker] (optional) → Reordered Chunks
     │
     ▼
[LLM] + [System Prompt] + [Retrieved Context] + [User Query]
     │
     ▼
[Response]
```

**Key RAG Components:**
```
1. Document Ingestion
   - Load → Parse → Chunk → Embed → Store in Vector DB

2. Chunking Strategies
   - Fixed-size: simple, predictable, may break context
   - Recursive: respects document structure (headers, paragraphs)
   - Semantic: groups semantically similar sentences together

3. Retrieval
   - Dense (vector similarity): semantic search via embeddings
   - Sparse (BM25/TF-IDF): keyword matching
   - Hybrid: combine both with RRF (Reciprocal Rank Fusion)

4. Re-ranking
   - Cross-encoder model scores query-document pairs more accurately
   - More expensive but improves precision of top results

5. Generation
   - Inject retrieved context into LLM prompt
   - Ground responses to prevent hallucination
```

**Agentic RAG:**
```
Standard RAG: single retrieval → generate
Agentic RAG:  agent decides WHEN and WHAT to retrieve
              - multi-hop: retrieves multiple times
              - self-query: reformulates query based on initial results
              - tool use: retrieval is one of many agent tools
```

---

### AI Agents and Agentic Systems

**Agent Components:**
```
┌─────────────────────────────────────────────────────┐
│                    AI Agent                         │
│                                                     │
│  [LLM Brain] ←→ [Memory] ←→ [Tools/Actions]        │
│       │                                             │
│  [Planning] (ReAct, CoT, Tree-of-Thought)           │
└─────────────────────────────────────────────────────┘

Memory types:
- In-context (short-term): conversation history in prompt
- External (long-term): vector DB, key-value store
- Episodic: past interaction summaries
- Semantic: knowledge base

Tool types:
- Web search, code execution, API calls
- RAG retrieval, database queries
- Other agents (multi-agent systems)
```

**ReAct Pattern (Reasoning + Acting):**
```
Thought: I need to find the current stock price of AAPL.
Action: search("AAPL stock price today")
Observation: AAPL is trading at $213.45
Thought: I have the answer.
Final Answer: AAPL is currently trading at $213.45.
```

**MCP (Model Context Protocol):**
```
MCP standardizes how LLMs interact with external tools/data sources.
- Server: exposes tools, resources, prompts
- Client: LLM application that consumes MCP servers
- Transport: stdio (local) or HTTP/SSE (remote)

Use case: connect Claude/GPT to your database, filesystem, APIs
via a standardized protocol instead of custom integrations.
```

---

### Prompt Engineering Patterns

```python
# Zero-shot
prompt = "Classify the sentiment: 'The product is amazing!'"

# Few-shot
prompt = """
Classify sentiment as positive/negative/neutral.

Input: "Great experience!" → positive
Input: "Terrible service." → negative
Input: "It was okay."     → neutral
Input: "I loved every moment!" → 
"""

# Chain-of-Thought (CoT)
prompt = """
Q: Roger has 5 balls. He buys 2 more cans of 3 balls each. How many does he have?
A: Roger starts with 5. Buys 2 cans × 3 balls = 6 balls. Total = 5 + 6 = 11.

Q: The cafeteria had 23 apples. They used 20 to make lunch and bought 6 more. How many?
A:
"""

# Structured output
prompt = """
Extract entities from the text and return valid JSON only.
Schema: {"people": [], "organizations": [], "locations": []}

Text: "Elon Musk founded SpaceX in Hawthorne, California."
"""

# System prompt for role
system = "You are an expert Python developer. Be concise. If you don't know, say so."
```

---

### Fine-Tuning

```
When to Fine-Tune vs RAG vs Prompt Engineering:

Prompt Engineering  → Try first. Zero cost, fast iteration.
RAG                 → When model needs external/updated knowledge.
Fine-Tuning         → When you need style/behavior change, 
                       consistent format, or domain-specific capability
                       that prompting can't achieve.

Fine-Tuning Methods:
┌─────────────────────────────────────────────────────┐
│ Full Fine-Tuning: update ALL weights                │
│   Pro: maximum flexibility                          │
│   Con: very expensive, catastrophic forgetting risk │
├─────────────────────────────────────────────────────┤
│ LoRA (Low-Rank Adaptation):                         │
│   Freeze base model, train low-rank adapter matrices│
│   W' = W + BA  where rank(B,A) << rank(W)           │
│   Pro: ~1% of parameters, swappable adapters        │
├─────────────────────────────────────────────────────┤
│ QLoRA: LoRA on quantized (4-bit) base model         │
│   Pro: fine-tune 70B model on single consumer GPU   │
├─────────────────────────────────────────────────────┤
│ RLHF: Reward model + PPO to align with preferences  │
│ DPO: Direct Preference Optimization (simpler RLHF)  │
└─────────────────────────────────────────────────────┘
```

---

### Quantization

```
Quantization: reduce model weight precision to save memory/speed up inference

FP32  → 4 bytes/param  (full precision training)
FP16  → 2 bytes/param  (standard inference)
INT8  → 1 byte/param   (post-training quantization, minimal quality loss)
INT4  → 0.5 bytes/param (aggressive, used in GGUF/llama.cpp, QLoRA)

Methods:
- PTQ (Post-Training Quantization): quantize after training, no retraining
- QAT (Quantization-Aware Training): simulate quantization during training
- GGUF: format for llama.cpp local inference (Q4_K_M, Q5_K_M, Q8_0)
- GPTQ: weight-only PTQ optimized for GPU inference
- AWQ: Activation-aware Weight Quantization, preserves salient weights

Rule of thumb:
- Q8: near lossless, 2× memory savings
- Q4_K_M: good quality/size balance for local models
- Q2: noticeable degradation, only for extreme memory constraints
```

---

### Vector Databases and Embeddings

```
Similarity Metrics:
- Cosine similarity: angle between vectors (most common for text)
- Dot product: magnitude + angle (used when vectors are normalized)
- Euclidean (L2): geometric distance (used in image embeddings)

ANN (Approximate Nearest Neighbor) Algorithms:
- HNSW (Hierarchical Navigable Small World): fast, high recall, default in most DBs
- IVF (Inverted File Index): cluster-based, good for large datasets
- LSH (Locality Sensitive Hashing): older, less accurate

Popular Vector DBs:
┌──────────────┬───────────────────────────────────────────┐
│ Pinecone     │ Managed, serverless, production-ready     │
│ Weaviate     │ Open-source, hybrid search built-in       │
│ Qdrant       │ Open-source, Rust-based, high performance │
│ Chroma       │ Open-source, great for prototyping        │
│ pgvector     │ PostgreSQL extension, familiar SQL         │
│ Milvus       │ Open-source, cloud-native, large scale    │
└──────────────┴───────────────────────────────────────────┘

Embedding Models:
- text-embedding-3-small/large (OpenAI): strong general purpose
- text-embedding-ada-002 (OpenAI): legacy, widely used
- all-MiniLM-L6-v2 (sentence-transformers): fast, free, local
- BAAI/bge-large-en-v1.5: strong open-source option
- nomic-embed-text: good open-source, long context
```

---

### LLMOps and Production AI

```
Key Production Concerns:

1. Latency
   - Use streaming (SSE/WebSocket) for perceived speed
   - KV cache, speculative decoding
   - Smaller/quantized models for latency-critical paths
   - Caching repeated queries (semantic cache with embedding similarity)

2. Cost
   - Prompt compression (remove redundant tokens)
   - Cache frequent prompts/responses
   - Route simple queries to smaller/cheaper models
   - Batch requests where possible

3. Reliability
   - Fallback chains (primary model → fallback model)
   - Retry with exponential backoff on rate limits
   - Circuit breakers for downstream dependencies

4. Observability
   - Log: prompt, response, latency, token count, model version
   - Trace multi-step agent runs end-to-end
   - Monitor: hallucination rate, toxicity, refusal rate
   - Tools: LangSmith, Helicone, Langfuse, Phoenix (Arize)

5. Evaluation
   - Automated: LLM-as-judge, embedding similarity, exact match
   - Human: preference ratings, thumbs up/down
   - RAG metrics: Faithfulness, Answer Relevancy, Context Precision/Recall
   - Tools: RAGAS, DeepEval, PromptFoo
```

---

## Common Interview Scenario Questions & Answers

### "Your LLM hallucinates. How do you fix it?"
```
1. Add RAG: ground responses in retrieved source documents
2. Prompt: "Only answer based on the provided context. If unsure, say 'I don't know.'"
3. Constrain output: use structured output (JSON schema) to limit free-form text
4. Lower temperature: reduce randomness in generation
5. Add citations: force model to cite sources, easier to verify
6. Evaluate: use faithfulness metric (RAGAS) to detect hallucinations automatically
7. Fine-tune: on (question, grounded-answer) pairs if domain-specific
```

### "Your RAG system retrieves irrelevant chunks. How do you improve retrieval?"
```
1. Improve chunking: use semantic or recursive chunking instead of fixed-size
2. Improve embeddings: use a better/domain-fine-tuned embedding model
3. Add hybrid search: combine BM25 + vector search with RRF
4. Add re-ranker: cross-encoder to re-score top-K results
5. Query expansion: use LLM to generate multiple query variants, retrieve for each
6. Metadata filtering: filter by date, source, category before vector search
7. HyDE (Hypothetical Document Embeddings): generate a hypothetical answer, embed it
```

### "Your agent loops infinitely. How do you fix it?"
```
1. Add max_iterations limit with hard stop
2. Add a "stuck detector": if last N actions are identical, break
3. Use structured output for agent decisions to prevent malformed tool calls
4. Add explicit "FINISH" action the agent must call to terminate
5. Improve planning prompt to include stopping conditions
6. Log and trace each step with LangSmith/Langfuse to debug the loop cause
```

### "Context window is too short for your documents. How do you handle it?"
```
1. Chunking + RAG: don't put full doc in context, retrieve only relevant chunks
2. Summarization: recursive summarization of long documents
3. Use a model with larger context (128K, 1M token models)
4. Map-reduce: process chunks independently, combine results
5. Sliding window: process with overlap, aggregate answers
```

---

## Practical Code Examples

### Simple RAG Pipeline (Python)

```python
import os
from openai import OpenAI
import chromadb
from chromadb.utils import embedding_functions

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

# Setup vector store
chroma_client = chromadb.Client()
openai_ef = embedding_functions.OpenAIEmbeddingFunction(
    api_key=os.environ["OPENAI_API_KEY"],
    model_name="text-embedding-3-small"
)
collection = chroma_client.create_collection(
    name="docs",
    embedding_function=openai_ef
)

# Ingest documents
documents = [
    "RAG combines retrieval with generation to reduce hallucinations.",
    "Fine-tuning adapts a pretrained model on domain-specific data.",
    "Quantization reduces model size by lowering weight precision.",
]
collection.add(
    documents=documents,
    ids=[f"doc_{i}" for i in range(len(documents))]
)

def rag_query(question: str, n_results: int = 2) -> str:
    # Retrieve
    results = collection.query(query_texts=[question], n_results=n_results)
    context = "\n".join(results["documents"][0])

    # Generate
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": "Answer based only on the provided context. "
                           "If the answer is not in the context, say 'I don't know.'"
            },
            {
                "role": "user",
                "content": f"Context:\n{context}\n\nQuestion: {question}"
            }
        ]
    )
    return response.choices[0].message.content

print(rag_query("What is quantization?"))
```

### ReAct Agent Pattern (Python)

```python
import os
import json
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

# Define tools
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_web",
            "description": "Search the web for current information",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Search query"}
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "Perform mathematical calculations",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string", "description": "Math expression"}
                },
                "required": ["expression"]
            }
        }
    }
]

def execute_tool(name: str, args: dict) -> str:
    if name == "search_web":
        # Replace with real search implementation
        return f"Search results for '{args['query']}': [mock result]"
    elif name == "calculate":
        return str(eval(args["expression"]))  # noqa: use safely in production
    return "Tool not found"

def run_agent(user_query: str, max_iterations: int = 5) -> str:
    messages = [
        {"role": "system", "content": "You are a helpful AI agent. Use tools when needed."},
        {"role": "user", "content": user_query}
    ]

    for _ in range(max_iterations):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,
            tool_choice="auto"
        )
        msg = response.choices[0].message

        if msg.tool_calls:
            messages.append(msg)
            for tool_call in msg.tool_calls:
                result = execute_tool(
                    tool_call.function.name,
                    json.loads(tool_call.function.arguments)
                )
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })
        else:
            return msg.content  # Final answer

    return "Max iterations reached."

print(run_agent("What is 1234 * 5678?"))
```

### Structured Output with JSON Schema

```python
import os
from pydantic import BaseModel
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

class Entity(BaseModel):
    name: str
    type: str  # person, organization, location

class ExtractionResult(BaseModel):
    entities: list[Entity]
    summary: str

response = client.beta.chat.completions.parse(
    model="gpt-4o-mini",
    messages=[
        {
            "role": "user",
            "content": "Extract entities: 'Sam Altman leads OpenAI, headquartered in San Francisco.'"
        }
    ],
    response_format=ExtractionResult,
)

result = response.choices[0].message.parsed
print(result.entities)   # [Entity(name='Sam Altman', type='person'), ...]
print(result.summary)
```

### Evaluating RAG with RAGAS

```python
import os
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

# Prepare evaluation dataset
eval_data = {
    "question": ["What is RAG?"],
    "answer": ["RAG stands for Retrieval-Augmented Generation..."],
    "contexts": [["RAG combines retrieval systems with language models..."]],
    "ground_truth": ["Retrieval-Augmented Generation is a technique..."]
}

dataset = Dataset.from_dict(eval_data)

result = evaluate(
    dataset=dataset,
    metrics=[faithfulness, answer_relevancy, context_precision],
)
print(result)
# {'faithfulness': 0.95, 'answer_relevancy': 0.87, 'context_precision': 0.90}
```

---

## Study Strategy for Interviews

```
Week 1: Foundations
  - LLM Fundamentals (Transformers, attention, tokenization)
  - Prompt Engineering patterns
  - RAG architecture

Week 2: Advanced Topics  
  - AI Agents, MCP, multi-agent systems
  - Fine-tuning (LoRA, QLoRA, RLHF, DPO)
  - Vector databases, embeddings, similarity search

Week 3: Production & Design
  - LLMOps, monitoring, evaluation
  - AI System Design (design a chatbot, RAG system, agent)
  - AI Safety, responsible AI

Week 4: Practice
  - Code a RAG pipeline from scratch
  - Code a simple agent with tool use
  - Mock system design interviews
  - Review scenario-based questions
```

---

## Troubleshooting Common Issues

| Issue | Solution |
|---|---|
| LLM ignores instructions | Use system prompt, structured output schemas, add examples |
| Hallucinations | Add RAG, citation requirement, lower temperature |
| Slow inference | KV cache, streaming, smaller model, batching |
| High token costs | Prompt compression, semantic caching, model routing |
| RAG retrieves wrong chunks | Better chunking, hybrid search, re-ranker |
| Agent loops | max_iterations, explicit FINISH action, better prompting |
| Context too long | Chunking, summarization, larger context model |
| Prompt injection | Input sanitization, separate system/user contexts |
| Reward hacking (RLHF) | Better reward model, constitutional AI, diverse evaluation |

---

## Key Resources

- **Repo**: https://github.com/amitshekhariitbhu/ai-engineering-interview-questions
- **Blog**: https://outcomeschool.substack.com
- **Course**: https://outcomeschool.com/program/ai-and-machine-learning
- **Video — Core concepts**: https://www.youtube.com/watch?v=lnfWvX66FUk
- **Author**: [@amitiitbhu](https://twitter.com/amitiitbhu)
```
