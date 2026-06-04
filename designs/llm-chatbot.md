# LLM-Powered Chatbot

## Problem Statement
Design a basic chatbot system that accepts user messages, sends them to a Large Language Model (LLM), and streams back coherent responses - supporting multi-turn conversations with context retention.

---

## Requirements

### Functional
- Users can send messages and receive responses from an LLM
- System maintains conversation history for multi-turn context
- Responses are streamed back in real time
- Support both self-hosted and external LLM providers
- Augment responses with relevant context via RAG
- Filter unsafe or out-of-scope inputs and outputs via Guardrails

### Non-Functional
- Low latency first token response
- Scalable to handle multiple concurrent users
- Conversation history persisted across sessions

---

## High-Level Architecture

![LLM Chatbot Architecture](../images/chat-service/chat_service.png)

---

## Core Components

| Component | Responsibility |
|---|---|
| **Client** | Sends user messages and renders responses |
| **API Gateway** | Auth, rate limiting, and routes requests to the Chat Service |
| **Chat Service** | Central orchestrator - coordinates all components to produce a response |
| **Cache** | Stores frequently requested responses to reduce redundant LLM calls |
| **RAG** | Retrieves relevant context chunks to augment the prompt |
| **Vector DB** | Stores and indexes embeddings for semantic search used by RAG |
| **Guardrails** | Validates inputs and outputs against safety and policy rules |
| **Guardrails Model** | ML model that powers the Guardrails safety checks |
| **Model Router** | Selects the most appropriate LLM based on cost, latency, or task type |
| **Model Selection Model** | ML model that informs the routing decision |
| **Self-Hosted LLMs** | Internal LLMs for cost-sensitive or data-privacy-sensitive workloads |
| **External LLMs** | Third-party providers (OpenAI, Anthropic, etc.) for general-purpose tasks |

---

## Component Deep Dive

### Client
The client is the interface through which users interact with the chatbot. It is responsible for capturing user input, sending it to the backend, and rendering the streamed response. Clients can take several forms:

- **Web Browser** - a web app (React, Vue, etc.) that communicates with the backend over HTTP/WebSocket
- **Mobile App** - a native iOS or Android app, or a cross-platform app (React Native, Flutter) that integrates the chat UI
- **Embedded Chat Window** - a chat widget embedded inside another product (e.g. a customer support window, a docs site, or an IDE plugin)

### API Gateway
The API Gateway is the single entry point for all client requests. It sits between the client and the backend services, handling cross-cutting concerns so the Chat Service does not have to. Key responsibilities include:

- **Authentication & Authorization** - validates identity (JWT, API keys, OAuth tokens) and enforces access control before any request reaches the backend
- **Rate Limiting** - caps the number of requests per user or IP over a time window to prevent abuse and control LLM cost
- **DDoS Protection** - detects and drops anomalous traffic spikes before they overwhelm downstream services
- **Request Filtering** - rejects malformed, oversized, or disallowed requests early in the pipeline
- **Request Routing** - directs traffic to the appropriate backend service based on path, headers, or version
- **Load Balancing** - distributes incoming traffic evenly across multiple Chat Service instances to prevent hotspots and improve availability
- **SSL/TLS Termination** - handles HTTPS at the gateway layer so backend services can communicate over plain HTTP internally, reducing certificate management overhead
- **Circuit Breaking** - detects failing downstream services and stops forwarding requests to them, preventing cascading failures across the system
- **Retry Logic** - automatically retries failed requests to the Chat Service or LLM provider with exponential backoff to improve resilience against transient failures
- **Usage Metering** - tracks request and token consumption per user or tenant, enabling quota enforcement, billing, and cost attribution
- **IP Allowlist / Blocklist** - permits or denies traffic at the network level based on IP address before it reaches any application logic
- **Streaming Support** - since LLM responses are streamed token-by-token, the gateway maintains long-lived connections (HTTP/2 server-sent events or WebSocket) and proxies the stream directly to the client without buffering the full response

**Technologies:**
- *Open Source* - Kong, NGINX, Traefik, Envoy, Tyk
- *Managed / Closed Source* - AWS API Gateway, Google Apigee, Azure API Management, Cloudflare API Shield

### Cache
The cache sits in front of the LLM and stores previous responses to avoid redundant and costly model calls. Rather than simple exact-match caching, this system uses **semantic caching** - meaning it can serve cached responses not just for identical queries but also for queries that are semantically similar.

**How it works:**
- When a new request comes in, the Chat Service generates an embedding of the user's query and performs a **vector similarity search** against previously cached query embeddings stored in Redis
- If a sufficiently similar query is found (above a configurable similarity threshold), the cached response is returned directly without hitting the LLM
- If no match is found (cache miss), the request proceeds to the LLM; once a response is generated, both the query embedding and the response are stored in Redis to populate the cache for future requests

**Similarity Score Thresholds:**

Similarity is typically measured using **cosine similarity**, which produces a score between 0 and 1 - where 1 is an exact match and 0 is completely unrelated. The threshold determines how close a query needs to be to a cached query to be considered a hit:

- **High threshold (e.g. ≥ 0.95)** - only near-identical queries are served from cache; safer and more accurate but results in a lower cache hit rate
- **Medium threshold (e.g. 0.85 – 0.95)** - catches paraphrased or slightly reworded versions of the same question; good balance for most use cases
- **Low threshold (e.g. < 0.85)** - more cache hits but higher risk of returning a response that does not actually match the user's intent

The right threshold depends on the application - a FAQ-style bot can tolerate a lower threshold safely, whereas an open-ended assistant should err on the higher side to avoid serving stale or mismatched responses.

**Embedding Models:**

The quality of semantic caching is directly tied to the embedding model used. The query at cache-write time and cache-lookup time must use the same model:

- *Open Source* - `all-MiniLM-L6-v2` (384 dims, fast and lightweight), `all-mpnet-base-v2` (768 dims, higher accuracy) via Sentence Transformers
- *Closed Source / Managed* - OpenAI `text-embedding-3-small` (1536 dims) and `text-embedding-3-large` (3072 dims), Cohere Embed, Google `text-embedding-gecko`

Smaller dimension models are faster and cheaper to store; larger dimension models capture more semantic nuance and tend to produce better similarity scores.

**Data Structure in Redis:**

Each cache entry is stored as a **Redis Hash** alongside an indexed vector field, using the **RediSearch** module:

- `query` - the original query text
- `response` - the full LLM response
- `embedding` - the query vector stored as a `VECTOR` field (float32 array)
- `created_at` - timestamp of when the entry was cached
- `ttl` - expiry set on the key to evict stale responses automatically

The vector index uses either a **FLAT** index (brute-force, suitable for small datasets) or an **HNSW** (Hierarchical Navigable Small World) index for approximate nearest-neighbour search at scale - HNSW trades a small amount of recall accuracy for significantly faster lookup times.

In a Redis Cluster, typical search latencies are:
- **FLAT index** - O(n) brute-force scan; ~1–10ms for datasets up to ~100K vectors, degrades linearly beyond that
- **HNSW index** - O(log n) approximate search; ~1–5ms even on millions of vectors, making it the practical choice for production workloads

Both figures exclude the network round-trip to the Redis Cluster (~0.1–1ms on a local network), which is negligible compared to the LLM call latency (typically hundreds of milliseconds to a few seconds).

**Operations Overview:**

1. **Embed** - incoming query is passed through the embedding model to produce a fixed-dimension vector
2. **KNN Search** - RediSearch runs a K-nearest neighbour query against all stored vectors, returning the top match(es) with their cosine similarity scores
3. **Threshold Check** - if the top match score meets or exceeds the configured threshold, the cached response is returned immediately
4. **Cache Miss Path** - query is forwarded to the LLM; on response, the query text, response, and embedding are written to a new Redis Hash entry and indexed
5. **TTL Expiry** - Redis automatically evicts entries past their TTL, keeping the cache fresh and preventing stale responses from being served

**Technologies:**
- *Cache Store* - Redis with the RediSearch module (part of Redis Stack) for vector indexing and KNN search

### RAG & Vector DB
**Retrieval-Augmented Generation (RAG)** is a technique that enhances LLM responses by injecting relevant external knowledge into the prompt at query time. Instead of relying solely on what the model learned during training, RAG fetches up-to-date, domain-specific context from a knowledge base - making the system more accurate, grounded, and easier to maintain without retraining the model.

**How RAG helps the system:**
- Reduces hallucinations by grounding the LLM's response in retrieved facts
- Enables the chatbot to answer questions about private or proprietary data the LLM was never trained on
- Keeps knowledge current - the Vector DB can be updated independently without touching the model
- Allows citations and source attribution by surfacing which documents informed the response

**RAG Walkthrough:**

RAG operates in two phases - an offline ingestion phase and an online retrieval phase:

*Ingestion (offline):*
1. Source documents (PDFs, wikis, databases, etc.) are split into smaller **chunks** (typically 256–512 tokens each)
2. Each chunk is passed through an **embedding model** to produce a vector representation
3. The vector and its associated metadata (source, chunk text, document ID) are stored in the **Vector DB**

*Retrieval (online, at query time):*
1. The user's query is embedded using the same model used during ingestion
2. A **KNN / ANN search** is run against the Vector DB to retrieve the top-K most semantically similar chunks
3. Retrieved chunks are ranked and the most relevant ones are injected into the prompt as context
4. The LLM generates a response conditioned on both the conversation history and the retrieved context

**How RAG enriches the output:**

Without RAG, the LLM responds purely from parametric memory - what it learned during training - which can be outdated or simply absent for domain-specific topics. With RAG, the prompt explicitly contains relevant facts, leading to:
- More specific and accurate answers grounded in real source material
- Fewer confident but wrong (hallucinated) responses
- The ability to handle internal knowledge bases, product documentation, or real-time data the model has never seen

**Embedding Models:**
- *Open Source* - `all-MiniLM-L6-v2` (384 dims, fast), `all-mpnet-base-v2` (768 dims, higher accuracy) via Sentence Transformers, `BGE-M3` (multi-lingual, strong retrieval performance)
- *Closed Source / Managed* - OpenAI `text-embedding-3-small` (1536 dims), `text-embedding-3-large` (3072 dims), Cohere Embed v3, Google `text-embedding-gecko`

The same embedding model must be used at both ingestion and retrieval time - mixing models produces incomparable vectors and degrades retrieval quality significantly.

**Vector DB Technologies:**
- *Open Source* - Qdrant, Weaviate, Milvus, ChromaDB, pgvector (PostgreSQL extension)
- *Managed / Closed Source* - Pinecone, Zilliz (managed Milvus), Weaviate Cloud

**Approximate RAG Latency:**

| Step | Approximate Time |
|---|---|
| Query embedding | ~20–100ms (API-based models); ~5–20ms (local models) |
| Vector search (ANN) | ~5–50ms depending on dataset size and index type |
| Chunk fetch & assembly | ~10–30ms |
| **Total RAG pipeline** | **~50–200ms end-to-end** |

This adds overhead compared to a direct LLM call, but is substantially faster than retraining or fine-tuning and well within acceptable latency budgets given that LLM generation itself typically takes 500ms–several seconds.

### Guardrails
The Guardrails layer is a dedicated content moderation service powered by a classification model that sits at two critical points in the pipeline - **before** the request reaches the LLM and **after** the LLM generates a response. It acts as the system's safety net, ensuring neither end of the conversation can be weaponised or misused.

**Why this layer is absolutely necessary:**

LLMs are powerful but inherently unpredictable - they can be manipulated into producing harmful content, leaking confidential information, or going wildly off-topic. Without a dedicated moderation layer, the system is only as safe as the base model's own (often inconsistent) safety training. Guardrails provide an explicit, auditable, and independently tunable safety boundary that:
- Protects users from harmful or offensive LLM outputs
- Protects the system from adversarial inputs designed to exploit the model
- Gives operators full control over what the chatbot will and will not engage with
- Builds user trust by ensuring the product behaves predictably and responsibly

**Input Guardrails** *(request → LLM)*

Input guardrails intercept the user's message before it is sent to the LLM. Their job is to ensure that no malicious, manipulative, or out-of-scope content reaches the model:

- **Prompt Injection** - detects attempts to override or hijack the system prompt by embedding instructions inside user messages (e.g. *"Ignore all previous instructions and..."*). The model is prevented from acting on injected directives that conflict with the system's intended behaviour
- **Jailbreaking** - identifies attempts to bypass the model's safety training through adversarial phrasing, roleplay scenarios, or obfuscation techniques designed to make the model produce content it would otherwise refuse
- **Topic Moderation** - classifies the user's intent and flags or blocks messages that fall outside the application's defined scope (e.g. a customer support bot should not engage with requests for illegal content or unrelated political discussions)
- **PII Detection** - identifies and redacts personally identifiable information in the input before it is logged or forwarded, protecting user privacy

**Output Guardrails** *(LLM → user)*

Output guardrails inspect the LLM's response before it is returned to the client. Even a well-behaved model can occasionally produce unsafe content - output guardrails ensure nothing harmful ever surfaces to the end user:

- **Toxicity & Hate Speech Detection** - flags and blocks responses containing offensive, discriminatory, or harmful language
- **Hallucination & Factual Grounding Check** - optionally verifies that the response is consistent with the retrieved RAG context, catching cases where the model fabricates information
- **PII Leakage Detection** - ensures the model has not surfaced sensitive data (e.g. from RAG-retrieved documents) that the user should not have access to
- **Policy Compliance** - checks that the response adheres to organisational, legal, or regulatory requirements (e.g. financial or medical disclaimers)

**How Guardrails improve user experience and trust:**

Beyond safety, a well-tuned Guardrails layer directly improves the quality of the user experience:
- Users receive consistent, on-topic responses - the chatbot does not go off the rails or engage with adversarial edge cases
- Transparent refusals (e.g. *"I can't help with that"*) are more trustworthy than model-generated deflections that may vary wildly in quality
- Operators can tune guardrail policies over time without retraining the underlying LLM, making it easy to adapt to new risks or compliance requirements
- Audit logs from the guardrails layer provide a clear paper trail for reviewing flagged interactions, which is essential in regulated industries

**Technologies:**
- *Open Source* - NeMo Guardrails (NVIDIA), Guardrails AI, LlamaGuard (Meta), Perspective API
- *Managed / Closed Source* - Azure Content Safety, AWS Comprehend, OpenAI Moderation API

### Model Router & Model Selection
The Model Router is responsible for deciding which LLM should handle a given request. Rather than hardwiring every request to a single model, the router dynamically selects the most appropriate one based on a combination of signals - balancing cost, latency, capability, and data sensitivity. The routing decision is informed by a **Model Selection Model**, a lightweight classifier that evaluates the incoming request and recommends a model tier.

**Why dynamic routing matters:**

Not every query needs the most powerful (and expensive) model. A simple FAQ lookup does not warrant the same compute as a complex multi-step reasoning task. Dynamic routing allows the system to:
- Serve simple queries with cheaper, faster models to reduce cost and latency
- Reserve high-capability models for complex or ambiguous requests
- Route sensitive or private data to self-hosted models to meet data residency or compliance requirements
- Gracefully fall back to an alternative model if the primary is unavailable or rate-limited

**Routing Criteria:**

The Model Selection Model evaluates each request across several signals to determine the best routing target:

- **Task complexity** - classifies the query as simple (factual lookup, short answer) or complex (reasoning, code generation, multi-step tasks) and routes accordingly
- **Query intent / task type** - identifies the nature of the request (summarisation, code, translation, conversation) and maps it to a model best suited for that task
- **Cost** - cheaper models (e.g. smaller open-source or quantised models) are preferred for lower-complexity requests to control spend
- **Latency requirements** - time-sensitive interactions (e.g. live chat) may prefer a faster model even at the cost of some quality
- **Data sensitivity** - requests containing PII or confidential data are flagged and routed to self-hosted models to avoid sending sensitive content to third-party APIs
- **Context length** - requests with long conversation histories or large RAG payloads are routed to models with larger context windows

**Routing Strategies:**

- **Rule-based routing** - simple, deterministic rules (e.g. *"if token count > 4000, route to model X"*); fast and predictable but inflexible
- **ML-based routing** - a lightweight classifier (the Model Selection Model) scores the request and selects a model; more adaptive but introduces a small classification overhead (~10–30ms)
- **Hybrid** - rule-based filters handle obvious cases (data sensitivity, context length) while the ML classifier handles nuanced task complexity decisions

**Self-Hosted vs External LLMs for Model Routing:**

Self-hosted models (e.g. LLaMA 3, Mistral, Falcon, Qwen) run on your own infrastructure, which means data never leaves your environment - making them the right choice for sensitive workloads, regulated industries, or scenarios with strict data residency requirements. They offer lower and more predictable latency since there is no external API hop, and costs are fixed regardless of usage volume. The trade-off is operational overhead: the team is responsible for hosting, scaling, and keeping models up to date. External providers (e.g. OpenAI GPT-4o, Anthropic Claude, Google Gemini) on the other hand offer state-of-the-art capabilities with zero infrastructure burden and a simple pay-per-token model - but every request sends data to a third-party, which may be unacceptable depending on compliance constraints.

**Data residency** is a particularly significant driver of routing decisions. Regulations such as GDPR, HIPAA, or country-specific data sovereignty laws may mandate that certain data never crosses regional boundaries or leaves the organisation's own infrastructure entirely. In such cases, the Model Router must be aware of data classification and routing rules - ensuring requests flagged as sensitive or region-restricted are always directed to a compliant self-hosted deployment, regardless of cost or capability considerations.

**Technologies:**
- *Open Source Routing Frameworks* - LiteLLM (unified API across providers), RouteLLM, OpenRouter
- *Self-Hosted Inference* - vLLM, Ollama, TGI (Text Generation Inference by HuggingFace), Triton Inference Server
- *External LLM Providers* - OpenAI, Anthropic, Google Vertex AI, Cohere, Mistral AI

### LLM Serving - Self-Hosted & External Models
Once the Model Router has made its decision, the request is forwarded to the selected model for final inference - the step where the actual response is generated. This is the most computationally intensive and latency-dominant part of the entire pipeline.

**Self-Hosted Models:**

Self-hosted models are open-weight models deployed and served on your own infrastructure. They give full control over the model, the hardware, and the data - nothing leaves your environment.

Popular model families:
- *LLaMA 3 (Meta)* - one of the strongest open-weight model families; available in 8B, 70B, and 405B parameter sizes; well-suited for general conversation, reasoning, and instruction following
- *Mistral / Mixtral (Mistral AI)* - highly efficient models; Mixtral uses a Mixture-of-Experts (MoE) architecture that delivers strong performance at lower compute cost compared to dense models of equivalent quality
- *Qwen 2.5 (Alibaba)* - strong multilingual support; competitive on coding and reasoning benchmarks; available in a wide range of sizes
- *Falcon (TII)* - efficient open-weight models, particularly well-suited for lower-resource deployments
- *Gemma 2 (Google)* - lightweight, fast models optimised for on-device and edge inference

Serving infrastructure for self-hosted models:
- *vLLM* - high-throughput inference server with PagedAttention for efficient KV cache management; the de facto standard for production self-hosted LLM serving
- *TGI (Text Generation Inference)* - HuggingFace's production inference server with streaming, continuous batching, and tensor parallelism support
- *Ollama* - lightweight local model runner; ideal for development and low-traffic deployments
- *Triton Inference Server (NVIDIA)* - enterprise-grade serving framework optimised for GPU workloads, supports multi-model and multi-framework deployments

Key hardware considerations:
- Model size dictates minimum GPU VRAM requirements (e.g. a 70B model in fp16 needs ~140GB VRAM, typically requiring multiple A100s or H100s)
- Quantisation techniques (GPTQ, AWQ, GGUF) can reduce memory footprint significantly - a 70B model quantised to 4-bit fits in ~35GB VRAM at a modest quality trade-off
- Tensor parallelism splits a single model across multiple GPUs to serve larger models or improve throughput

**External Models:**

External models are accessed via API and are fully managed by the provider - no infrastructure to operate, but every request is a network call to a third party.

Major providers and their flagship models:
- *OpenAI* - GPT-4o (fast, multimodal, strong reasoning), o1 / o3 (extended thinking for complex reasoning tasks), GPT-4o mini (cost-efficient for simpler tasks)
- *Anthropic* - Claude Opus 4 (top-tier reasoning and long-context tasks), Claude Sonnet 4 (best balance of capability and speed), Claude Haiku 3.5 (low latency, cost-efficient)
- *Google* - Gemini 2.5 Pro (strong reasoning, very large context window), Gemini 2.0 Flash (fast and cost-efficient)
- *Mistral AI* - Mistral Large (strong multilingual and coding), Mistral Small (efficient, low cost)
- *Cohere* - Command R+ (optimised for RAG and enterprise retrieval workloads)

Key considerations for external models:
- **Latency** - time-to-first-token (TTFT) varies widely by provider and model tier; typically ranges from 200ms to 2s+ depending on load
- **Rate limits** - providers enforce per-minute token and request limits; production systems must implement retry logic and queue management to handle bursts
- **Cost** - billed per input and output token; costs can scale significantly with context length, especially when RAG payloads or long conversation histories are included in every request
- **Model versioning** - providers periodically update or retire model versions; the system must handle version pinning and migration gracefully to avoid unexpected behaviour changes

### Chat Service
The Chat Service is the central orchestrator and the brain of the entire system. It is the only component that communicates with every other layer - the API Gateway, Cache, RAG, Guardrails, and Model Router - and is responsible for coordinating them in the right sequence to produce a coherent, safe, and contextually aware response. No other component has awareness of the full request lifecycle; the Chat Service holds it all together.

**Session & Conversation Management:**

Every user interaction belongs to a session. The Chat Service is responsible for maintaining the state of that session across turns:
- On each incoming request, it retrieves the conversation history for the session from the session store (Redis for recent context, persistent DB for full history)
- It tracks the running token count of the conversation and decides when to truncate older messages or trigger a summarisation pass to stay within the model's context window
- Session metadata (user ID, session ID, timestamps, model used) is persisted to allow resuming conversations across connections or devices

**Prompt Construction:**

One of the Chat Service's most critical responsibilities is assembling the final prompt that gets sent to the LLM. This is not just appending the user's message - it is a deliberate composition of multiple inputs:
1. **System prompt** - defines the assistant's persona, tone, scope, and any hard rules (e.g. "You are a customer support agent for X. Do not discuss competitors.")
2. **RAG context** - relevant document chunks retrieved from the Vector DB, injected as grounding context before the conversation
3. **Conversation history** - prior turns in the session, managed and trimmed to fit within the context window budget
4. **User message** - the current user input, appended last

The order and formatting of these blocks meaningfully affect output quality. The Chat Service applies prompt templates that are versioned and tunable independently of the model.

**Orchestration Flow:**

The Chat Service drives the full request lifecycle in a defined sequence:
1. Receive request from the API Gateway
2. Load session history from the store
3. Check the semantic cache - return immediately if a hit is found
4. Run input Guardrails on the user message - reject or flag if needed
5. Trigger RAG retrieval in parallel where possible to minimise latency
6. Construct the final prompt from all assembled inputs
7. Forward to the Model Router, which selects and calls the appropriate LLM
8. Stream tokens back from the LLM as they arrive
9. Run output Guardrails on the response stream before forwarding to the client
10. Persist the new message pair to the session store and cache

**Context Window Management:**

As conversations grow, the accumulated history can exceed the model's context window limit. The Chat Service handles this through:
- **Sliding window** - retains only the most recent N turns and drops the oldest
- **Summarisation** - compresses older turns into a short summary that is prepended to the context, preserving continuity without consuming the full token budget
- **Token counting** - tracks tokens per message using the same tokeniser as the target model to ensure the assembled prompt never exceeds the limit before dispatch

**Streaming:**

The Chat Service does not wait for the full LLM response before forwarding to the client. It opens a streaming connection to the LLM and pipes tokens through to the client via the API Gateway in real time - using Server-Sent Events (SSE) or WebSocket. This dramatically reduces perceived latency since the user sees the response begin appearing within the first few hundred milliseconds, even for long outputs.

**Error Handling & Fallbacks:**

The Chat Service is responsible for graceful degradation when things go wrong:
- If the primary LLM is rate-limited or unavailable, it retries with exponential backoff or falls back to an alternative model via the Model Router
- If RAG retrieval fails, the request proceeds without context augmentation rather than failing entirely
- If Guardrails flag an input or output, the Chat Service returns a structured refusal message rather than an empty or broken response

**Observability:**

Since the Chat Service touches every component, it is the natural place to emit structured logs, metrics, and traces for the entire request lifecycle - including latency per stage, model used, token counts, cache hit/miss, and guardrail outcomes. This data feeds dashboards, alerting, and cost attribution.

**Technologies:**
- *Orchestration Frameworks* - LangChain, LlamaIndex, Semantic Kernel (provide pre-built abstractions for prompt management, RAG pipelines, memory, and tool use)
- *Custom Services* - FastAPI or Node.js for teams that prefer full control over the orchestration logic without framework abstractions
- *Session Store* - Redis (short-term context), PostgreSQL (full conversation history)
- *Streaming* - Server-Sent Events (SSE) or WebSocket for real-time token delivery to the client

---

## Data Flow

1. **User sends a message** - the client (web, mobile, or embedded widget) captures the user's input and sends it to the backend over HTTP or a WebSocket connection, along with a session identifier to associate the message with an ongoing conversation.

2. **API Gateway processes the request** - the gateway handles SSL/TLS termination, authenticates the request (JWT, API key, or OAuth token), checks rate limits and IP rules, and inspects the request for any obviously malformed or oversized payloads. If all checks pass, it routes the request to the Chat Service and keeps the connection open for streaming.

3. **Chat Service loads session context** - the Chat Service retrieves the conversation history for the session from Redis (recent turns) and the persistent DB (full history if needed). It counts the running token total and applies truncation or summarisation if the history approaches the model's context window limit.

4. **Semantic cache lookup** - the Chat Service embeds the incoming query using the configured embedding model and runs a KNN similarity search against previously cached query embeddings in Redis. If a match above the similarity threshold is found, the cached response is streamed back immediately to the client, skipping all downstream components entirely.

5. **Input Guardrails** - on a cache miss, the user message is passed through the input Guardrails layer. The Guardrails model checks for prompt injection attempts, jailbreaking patterns, out-of-scope topics, and PII. If the message is flagged, a structured refusal is returned to the client and the request goes no further.

6. **RAG retrieval** - the Chat Service embeds the query and dispatches a semantic search to the Vector DB to retrieve the top-K most relevant document chunks. Retrieved chunks are ranked by relevance score and the highest-scoring ones are selected to be injected as grounding context into the prompt.

7. **Prompt construction** - the Chat Service assembles the final prompt in a structured order: the system prompt (persona, scope, rules), followed by the RAG-retrieved context chunks, followed by the trimmed conversation history, and finally the user's current message. This fully composed prompt is what gets dispatched to the LLM.

8. **Model Router selects the LLM** - the Model Router evaluates the request using the Model Selection Model, scoring it across signals such as task complexity, query intent, token count, and data sensitivity. Based on this, it selects the most appropriate model - routing privacy-sensitive requests to a self-hosted model and complex or general-purpose requests to an external provider.

9. **LLM generates and streams the response** - the selected model receives the prompt and begins generating a response token by token. The Chat Service receives the token stream in real time and immediately begins forwarding it downstream, without waiting for the full response to complete.

10. **Output Guardrails** - as tokens stream in, the output Guardrails layer inspects the response for toxicity, hate speech, PII leakage, hallucinations inconsistent with the RAG context, and policy violations. If the output is flagged mid-stream, the stream is interrupted and a safe fallback message is returned to the client.

11. **Response streamed to the client** - clean tokens are forwarded through the API Gateway to the client in real time via SSE or WebSocket. The user sees the response begin appearing almost immediately after the first token is generated, minimising perceived latency.

12. **Session and cache updated** - once the full response has been delivered, the Chat Service persists the new message pair (user message + assistant response) to the session store and the persistent DB. The query embedding and response are also written to the Redis cache with a TTL, making them available for future semantic cache hits.

---

## Trade-offs & Considerations

- **RAG vs. fine-tuning** - RAG is easier to keep up to date but adds retrieval latency; fine-tuning bakes knowledge into the model but requires retraining cycles
- **Model routing** - routing to cheaper or self-hosted models reduces cost but may affect response quality for complex queries
- **Guardrails latency** - running safety checks on both input and output adds round-trip overhead; async or batched checks can mitigate this
- **Cache hit rate** - caching works well for repetitive queries but has low utility in open-ended conversational settings
- **Context window limit** - older messages must be truncated or summarized as conversations grow long
- **Streaming vs. REST** - WebSockets give a better UX for streaming but add connection management complexity
