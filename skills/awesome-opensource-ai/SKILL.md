```markdown
---
name: awesome-opensource-ai
description: Curated guide to the best open-source AI projects, models, tools, and infrastructure across the full ML stack
triggers:
  - show me open source AI tools
  - what are the best open source LLMs
  - recommend open source ML frameworks
  - find open source alternatives to closed AI APIs
  - what open source models should I use for my project
  - help me pick an open source inference engine
  - what are good open source RAG tools
  - open source AI stack for production
---

# Awesome Open Source AI

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

A curated reference for open-source AI models, libraries, infrastructure, and developer tools spanning the full ML/LLM stack — from training frameworks to production deployment.

---

## What This Resource Covers

The [awesome-opensource-ai](https://github.com/alvinunreal/awesome-opensource-ai) list organizes the open-source AI ecosystem into 14 categories:

1. Core Frameworks & Libraries
2. Open Foundation Models
3. Inference Engines & Serving
4. Agentic AI & Multi-Agent Systems
5. Retrieval-Augmented Generation (RAG) & Knowledge
6. Generative Media Tools
7. Training & Fine-tuning Ecosystem
8. MLOps / LLMOps & Production
9. Evaluation, Benchmarks & Datasets
10. AI Safety, Alignment & Interpretability
11. Specialized Domains
12. User Interfaces & Self-hosted Platforms
13. Developer Tools & Integrations
14. Resources & Learning

---

## Quick Decision Guide by Use Case

### "I need to run an LLM locally"

| Need | Recommended Tool |
|------|-----------------|
| Simple local chat | [Ollama](https://github.com/ollama/ollama) |
| Max performance inference | [llama.cpp](https://github.com/ggerganov/llama.cpp) or [vLLM](https://github.com/vllm-project/vllm) |
| OpenAI-compatible API | [LocalAI](https://github.com/mudler/LocalAI) or [LM Studio](https://lmstudio.ai) |
| Production serving | [vLLM](https://github.com/vllm-project/vllm) or [TGI](https://github.com/huggingface/text-generation-inference) |

### "I need to train or fine-tune a model"

| Need | Recommended Tool |
|------|-----------------|
| LoRA/QLoRA fine-tuning | [Unsloth](https://github.com/unslothai/unsloth) or [LLaMA-Factory](https://github.com/hiyouga/LLaMA-Factory) |
| Full training at scale | [DeepSpeed](https://github.com/microsoft/DeepSpeed) + [Megatron-LM](https://github.com/NVIDIA/Megatron-LM) |
| Quick experiments | [Hugging Face Transformers](https://github.com/huggingface/transformers) + [Accelerate](https://github.com/huggingface/accelerate) |

### "I need to build a RAG pipeline"

| Need | Recommended Tool |
|------|-----------------|
| Full RAG framework | [LlamaIndex](https://github.com/run-llama/llama_index) or [Haystack](https://github.com/deepset-ai/haystack) |
| Vector store | [Chroma](https://github.com/chroma-core/chroma), [Qdrant](https://github.com/qdrant/qdrant), or [Weaviate](https://github.com/weaviate/weaviate) |
| Embeddings model | [sentence-transformers](https://github.com/UKPLab/sentence-transformers) |

### "I need to build an AI agent"

| Need | Recommended Tool |
|------|-----------------|
| General agents | [LangChain](https://github.com/langchain-ai/langchain) or [LlamaIndex Workflows](https://github.com/run-llama/llama_index) |
| Multi-agent orchestration | [AutoGen](https://github.com/microsoft/autogen) or [CrewAI](https://github.com/joaomdmoura/crewAI) |
| Code agents | [OpenHands](https://github.com/All-Hands-AI/OpenHands) or [SWE-agent](https://github.com/princeton-nlp/SWE-agent) |

---

## Model Selection Guide

### Open LLMs by Size & Use Case

```
Small (1B–7B) — Edge, mobile, low-resource:
  - Phi-4-Mini (Microsoft) — best reasoning per parameter
  - Gemma 3 2B/7B (Google) — strong efficiency
  - Qwen3.5-3B/7B — excellent multilingual

Medium (8B–30B) — Balanced production use:
  - Llama 4 8B — general purpose workhorse
  - Qwen3.5-14B — coding + math
  - Mistral Small — multilingual, tool use

Large (70B+) — Max capability open:
  - Llama 4 405B — frontier open model
  - DeepSeek-V3.2 (MoE 671B active 37B) — math/reasoning
  - Qwen3.5-72B — top open coding/math

Coding Specialists:
  - Qwen2.5-Coder-32B — #1 open coding
  - DeepSeek-Coder-V2 — MoE coding powerhouse
  - StarCoder2-15B — 600+ languages, transparent

Vision-Language:
  - Qwen2.5-VL-72B — top open VLM
  - InternVL 2.5 — charts, OCR, video
  - LLaVA-Next — most popular/documented
```

---

## Core Framework Examples

### PyTorch — Basic Training Loop

```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader

# Define model
class SimpleNet(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Dropout(0.1),
            nn.Linear(hidden_dim, output_dim)
        )
    
    def forward(self, x):
        return self.layers(x)

model = SimpleNet(784, 256, 10).to("cuda")
optimizer = torch.optim.AdamW(model.parameters(), lr=3e-4)
criterion = nn.CrossEntropyLoss()

# Training loop
for epoch in range(10):
    for batch_x, batch_y in dataloader:
        batch_x, batch_y = batch_x.to("cuda"), batch_y.to("cuda")
        
        optimizer.zero_grad()
        logits = model(batch_x)
        loss = criterion(logits, batch_y)
        loss.backward()
        optimizer.step()
```

### Hugging Face Transformers — Load & Inference

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

model_id = "meta-llama/Llama-3.1-8B-Instruct"

tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    torch_dtype=torch.bfloat16,
    device_map="auto",  # auto-distributes across available GPUs
)

messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Explain gradient descent in simple terms."},
]

input_ids = tokenizer.apply_chat_template(
    messages,
    add_generation_prompt=True,
    return_tensors="pt"
).to(model.device)

with torch.inference_mode():
    outputs = model.generate(
        input_ids,
        max_new_tokens=512,
        temperature=0.7,
        do_sample=True,
    )

response = tokenizer.decode(outputs[0][input_ids.shape[-1]:], skip_special_tokens=True)
print(response)
```

### Hugging Face Accelerate — Multi-GPU Training

```python
from accelerate import Accelerator
from transformers import AutoModelForSequenceClassification, AutoTokenizer
from torch.utils.data import DataLoader
import torch

accelerator = Accelerator(mixed_precision="bf16")

model = AutoModelForSequenceClassification.from_pretrained("bert-base-uncased", num_labels=2)
optimizer = torch.optim.AdamW(model.parameters(), lr=2e-5)

# Accelerate handles device placement, mixed precision, distributed training
model, optimizer, train_dataloader = accelerator.prepare(
    model, optimizer, train_dataloader
)

for epoch in range(3):
    for batch in train_dataloader:
        outputs = model(**batch)
        loss = outputs.loss
        accelerator.backward(loss)
        optimizer.step()
        optimizer.zero_grad()

# Save — handles unwrapping DistributedDataParallel automatically
accelerator.wait_for_everyone()
unwrapped = accelerator.unwrap_model(model)
unwrapped.save_pretrained("./output", save_function=accelerator.save)
```

---

## Inference Engine Examples

### vLLM — Production OpenAI-Compatible Server

```bash
# Install
pip install vllm

# Start server (OpenAI-compatible)
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --dtype bfloat16 \
  --tensor-parallel-size 2 \
  --max-model-len 8192 \
  --port 8000
```

```python
# Use with OpenAI client
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"  # vLLM doesn't require auth by default
)

response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[{"role": "user", "content": "Write a Python function to reverse a string."}],
    temperature=0.7,
    max_tokens=512,
)
print(response.choices[0].message.content)
```

### Ollama — Local Model Management

```bash
# Install (macOS/Linux)
curl -fsSL https://ollama.ai/install.sh | sh

# Pull and run models
ollama pull llama3.1:8b
ollama pull qwen2.5-coder:14b
ollama pull mistral:7b

# Interactive chat
ollama run llama3.1:8b

# Serve API (default port 11434)
ollama serve
```

```python
import ollama

# Simple generation
response = ollama.chat(
    model="llama3.1:8b",
    messages=[{"role": "user", "content": "What is RAG in AI?"}]
)
print(response["message"]["content"])

# Streaming
for chunk in ollama.chat(
    model="qwen2.5-coder:14b",
    messages=[{"role": "user", "content": "Write a FastAPI CRUD app"}],
    stream=True
):
    print(chunk["message"]["content"], end="", flush=True)

# Embeddings
embedding = ollama.embeddings(
    model="nomic-embed-text",
    prompt="Represent this document for retrieval:"
)
vector = embedding["embedding"]  # list of floats
```

### llama.cpp — CPU/GPU Inference

```bash
# Build
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
make -j$(nproc)                    # CPU only
make LLAMA_CUDA=1 -j$(nproc)      # NVIDIA GPU
make LLAMA_METAL=1 -j$(nproc)     # Apple Silicon

# Download a GGUF model (e.g. from HuggingFace)
# Then run:
./llama-cli -m ./models/llama-3.1-8b-instruct.Q4_K_M.gguf \
  -p "You are a helpful assistant." \
  --chat-template llama3 \
  -n 512 \
  --temp 0.7

# Start OpenAI-compatible server
./llama-server -m ./models/llama-3.1-8b-instruct.Q4_K_M.gguf \
  --host 0.0.0.0 --port 8080 \
  -ngl 35  # layers to offload to GPU
```

---

## RAG Pipeline Examples

### LlamaIndex — Complete RAG Setup

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.llms.ollama import Ollama
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

# Configure models
Settings.llm = Ollama(model="llama3.1:8b", request_timeout=120.0)
Settings.embed_model = HuggingFaceEmbedding(
    model_name="BAAI/bge-small-en-v1.5"
)

# Load documents
documents = SimpleDirectoryReader("./data").load_data()

# Build index
index = VectorStoreIndex.from_documents(
    documents,
    show_progress=True
)

# Persist index
index.storage_context.persist(persist_dir="./storage")

# Query
query_engine = index.as_query_engine(similarity_top_k=5)
response = query_engine.query("What are the main findings?")
print(response)
```

### Chroma — Vector Store

```python
import chromadb
from chromadb.utils import embedding_functions

# Initialize client (persistent)
client = chromadb.PersistentClient(path="./chroma_db")

# Use sentence-transformers embeddings
ef = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="all-MiniLM-L6-v2"
)

collection = client.get_or_create_collection(
    name="documents",
    embedding_function=ef,
    metadata={"hnsw:space": "cosine"}
)

# Add documents
collection.add(
    documents=[
        "PyTorch is a machine learning framework.",
        "LangChain helps build LLM applications.",
        "Vector databases store embeddings for similarity search.",
    ],
    ids=["doc1", "doc2", "doc3"],
    metadatas=[
        {"source": "ml_docs", "category": "framework"},
        {"source": "llm_docs", "category": "framework"},
        {"source": "db_docs", "category": "database"},
    ]
)

# Query
results = collection.query(
    query_texts=["how do I train neural networks?"],
    n_results=2,
    where={"category": "framework"}  # optional metadata filter
)

for doc, score in zip(results["documents"][0], results["distances"][0]):
    print(f"Score: {1 - score:.3f} | {doc[:80]}...")
```

---

## Agentic AI Examples

### LangChain — ReAct Agent with Tools

```python
from langchain_community.llms import Ollama
from langchain.agents import create_react_agent, AgentExecutor
from langchain.tools import tool
from langchain import hub

llm = Ollama(model="llama3.1:8b")

@tool
def search_docs(query: str) -> str:
    """Search internal documentation for information."""
    # Replace with your actual search logic
    return f"Documentation results for: {query}"

@tool  
def run_python(code: str) -> str:
    """Execute Python code and return the output."""
    import io, contextlib
    output = io.StringIO()
    try:
        with contextlib.redirect_stdout(output):
            exec(code, {})
        return output.getvalue() or "Code executed successfully (no output)"
    except Exception as e:
        return f"Error: {str(e)}"

tools = [search_docs, run_python]
prompt = hub.pull("hwchase17/react")

agent = create_react_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True, max_iterations=5)

result = executor.invoke({
    "input": "Search for how to use pandas groupby, then write a code example."
})
print(result["output"])
```

### AutoGen — Multi-Agent Conversation

```python
import autogen

config_list = [{
    "model": "llama3.1:8b",
    "base_url": "http://localhost:11434/v1",
    "api_key": "ollama",
}]

llm_config = {"config_list": config_list, "temperature": 0.7}

# Create agents
assistant = autogen.AssistantAgent(
    name="Assistant",
    llm_config=llm_config,
    system_message="You are a helpful AI. Solve tasks step by step."
)

code_reviewer = autogen.AssistantAgent(
    name="CodeReviewer", 
    llm_config=llm_config,
    system_message="You review code for bugs, security issues, and best practices."
)

user_proxy = autogen.UserProxyAgent(
    name="User",
    human_input_mode="NEVER",
    max_consecutive_auto_reply=5,
    code_execution_config={"work_dir": "workspace", "use_docker": False},
)

# Group chat
groupchat = autogen.GroupChat(
    agents=[user_proxy, assistant, code_reviewer],
    messages=[],
    max_round=10
)
manager = autogen.GroupChatManager(groupchat=groupchat, llm_config=llm_config)

user_proxy.initiate_chat(
    manager,
    message="Write a Python script that scrapes headlines from a news RSS feed and summarizes them."
)
```

---

## Fine-tuning Examples

### Unsloth — Fast LoRA Fine-tuning

```python
from unsloth import FastLanguageModel
from trl import SFTTrainer
from transformers import TrainingArguments
from datasets import load_dataset

# Load model with Unsloth optimizations (2x faster, 60% less VRAM)
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/Meta-Llama-3.1-8B-Instruct",
    max_seq_length=2048,
    dtype=None,  # auto-detect
    load_in_4bit=True,
)

# Add LoRA adapters
model = FastLanguageModel.get_peft_model(
    model,
    r=16,               # LoRA rank
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    lora_alpha=16,
    lora_dropout=0,
    bias="none",
    use_gradient_checkpointing="unsloth",
    random_state=42,
)

dataset = load_dataset("yahma/alpaca-cleaned", split="train")

trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=dataset,
    dataset_text_field="text",
    max_seq_length=2048,
    args=TrainingArguments(
        per_device_train_batch_size=4,
        gradient_accumulation_steps=4,
        num_train_epochs=1,
        learning_rate=2e-4,
        fp16=True,
        output_dir="./output",
        save_steps=100,
        logging_steps=10,
    ),
)

trainer.train()

# Save LoRA weights
model.save_pretrained("./lora_model")
tokenizer.save_pretrained("./lora_model")

# Optionally merge and export to GGUF
model.save_pretrained_gguf("./gguf_model", tokenizer, quantization_method="q4_k_m")
```

---

## MLOps Examples

### MLflow — Experiment Tracking

```python
import mlflow
import mlflow.pytorch
from mlflow.models import infer_signature

mlflow.set_experiment("llm-fine-tuning")

with mlflow.start_run(run_name="llama3-lora-v1"):
    # Log hyperparameters
    mlflow.log_params({
        "model": "llama3.1-8b",
        "lora_rank": 16,
        "learning_rate": 2e-4,
        "epochs": 3,
        "batch_size": 4,
    })
    
    # Log metrics during training
    for step, loss in enumerate(training_losses):
        mlflow.log_metric("train_loss", loss, step=step)
    
    mlflow.log_metric("eval_perplexity", 12.4)
    mlflow.log_metric("eval_bleu", 0.38)
    
    # Log artifacts
    mlflow.log_artifact("./lora_model", artifact_path="model")
    mlflow.log_artifact("./training_config.yaml")
    
    # Tag the run
    mlflow.set_tags({
        "task": "instruction-tuning",
        "dataset": "alpaca-cleaned",
        "framework": "unsloth",
    })

# Query runs programmatically
runs = mlflow.search_runs(
    experiment_names=["llm-fine-tuning"],
    filter_string="metrics.eval_perplexity < 15",
    order_by=["metrics.eval_perplexity ASC"],
)
print(runs[["run_id", "params.model", "metrics.eval_perplexity"]].head())
```

---

## Common Patterns

### Pattern 1: Local LLM with Fallback

```python
import os
from openai import OpenAI

def get_llm_client(prefer_local: bool = True):
    """Returns OpenAI-compatible client, preferring local vLLM/Ollama."""
    if prefer_local:
        try:
            client = OpenAI(
                base_url=os.getenv("LOCAL_LLM_URL", "http://localhost:11434/v1"),
                api_key="local"
            )
            # Test connection
            client.models.list()
            return client, os.getenv("LOCAL_MODEL", "llama3.1:8b")
        except Exception:
            pass
    
    # Fallback to OpenAI
    return OpenAI(api_key=os.environ["OPENAI_API_KEY"]), "gpt-4o-mini"

client, model = get_llm_client()
```

### Pattern 2: Embeddings + Similarity Search (No Vector DB)

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer("BAAI/bge-small-en-v1.5")

def build_index(texts: list[str]) -> np.ndarray:
    return model.encode(texts, normalize_embeddings=True)

def search(query: str, corpus_embeddings: np.ndarray, texts: list[str], top_k: int = 5):
    query_emb = model.encode([query], normalize_embeddings=True)
    scores = (query_emb @ corpus_embeddings.T)[0]
    top_indices = np.argsort(scores)[::-1][:top_k]
    return [(texts[i], float(scores[i])) for i in top_indices]

# Usage
texts = ["doc 1 content", "doc 2 content", "doc 3 content"]
embeddings = build_index(texts)
results = search("my query", embeddings, texts)
```

### Pattern 3: Structured Output with Pydantic

```python
from pydantic import BaseModel
from transformers import pipeline
import json

class CodeReview(BaseModel):
    has_bugs: bool
    severity: str  # "low" | "medium" | "high" | "critical"
    issues: list[str]
    suggestions: list[str]

def review_code(code: str, llm_pipeline) -> CodeReview:
    prompt = f"""Review this code and respond with ONLY valid JSON matching this schema:
{CodeReview.model_json_schema()}

Code to review:
```python
{code}
```"""
    
    output = llm_pipeline(prompt, max_new_tokens=512)[0]["generated_text"]
    
    # Extract JSON from output
    json_start = output.rfind("{")
    json_end = output.rfind("}") + 1
    json_str = output[json_start:json_end]
    
    return CodeReview.model_validate_json(json_str)
```

---

## Troubleshooting

### CUDA Out of Memory

```python
# Reduce memory usage:

# 1. Use 4-bit quantization
from transformers import BitsAndBytesConfig
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_use_double_quant=True,
)
model = AutoModelForCausalLM.from_pretrained(model_id, quantization_config=bnb_config)

# 2. Enable gradient checkpointing
model.gradient_checkpointing_enable()

# 3. Use smaller batch + gradient accumulation
# Instead of batch_size=32, use batch_size=4, grad_accum=8

# 4. Clear cache between operations
import gc
torch.cuda.empty_cache()
gc.collect()
```

### vLLM Slow First Response

```bash
# Pre-warm the model after startup
curl -s http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "meta-llama/Llama-3.1-8B-Instruct", "prompt": "hi", "max_tokens": 1}'
```

### Hugging Face Download Issues

```bash
# Use environment variables for auth and caching
export HUGGING_FACE_HUB_TOKEN="your_token_here"   # use env var, not hardcoded
export HF_HOME="/path/to/large/disk/.cache/huggingface"
export HF_HUB_OFFLINE=1  # use cached files only (after download)

# Download model files explicitly
huggingface-cli download meta-llama/Llama-3.1-8B-Instruct \
  --local-dir ./models/llama3.1-8b \
  --include "*.safetensors" "*.json" "tokenizer*"
```

### Ollama Model Not Found

```bash
# List available models
ollama list

# Search for models
ollama search llama

# Pull specific version/quantization
ollama pull qwen2.5-coder:14b-instruct-q4_K_M

# Check running status
ollama ps
```

---

## Environment Setup

```bash
# Minimal ML environment
conda create -n ai-dev python=3.11
conda activate ai-dev

pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
pip install transformers accelerate datasets peft trl
pip install vllm                          # production inference
pip install llama-index chromadb          # RAG
pip install langchain langchain-community # agents
pip install mlflow                        # experiment tracking
pip install sentence-transformers         # embeddings
pip install unsloth                       # fast fine-tuning

# Environment variables (add to .env or shell profile)
export HUGGING_FACE_HUB_TOKEN="${HUGGING_FACE_HUB_TOKEN}"
export OPENAI_API_KEY="${OPENAI_API_KEY}"           # if using OpenAI fallback
export ANTHROPIC_API_KEY="${ANTHROPIC_API_KEY}"     # if using Anthropic
export HF_HOME="${HF_HOME:-~/.cache/huggingface}"
export TRANSFORMERS_CACHE="${HF_HOME}/hub"
```

---

## Key Resources

- **Awesome List**: https://github.com/alvinunreal/awesome-opensource-ai
- **Hugging Face Hub**: https://huggingface.co/models (model downloads)
- **Ollama Library**: https://ollama.com/library (curated GGUF models)
- **Open LLM Leaderboard**: https://huggingface.co/spaces/HuggingFaceH4/open_llm_leaderboard
- **LMSYS Chatbot Arena**: https://chat.lmsys.org (human preference rankings)
- **Papers With Code**: https://paperswithcode.com/sota (benchmark tracking)
```
