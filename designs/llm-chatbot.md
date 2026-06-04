# LLM-Powered Chatbot

## 1. Problem Statement

Design a production-grade, multi-tenant conversational system that accepts user messages, augments them with relevant knowledge and conversational context, generates coherent and safe responses using a Large Language Model (LLM), and streams the output back to the client in real time across multiple turns.

**In Scope:**
- Multi-turn conversational interaction with stateful session memory
- Retrieval-Augmented Generation (RAG) over internal knowledge sources
- Routing between self-hosted and external LLMs based on cost, latency, capability, and data sensitivity
- Content moderation (input and output) via dedicated Guardrails
- Real-time token streaming to the client
- Tool use / function calling for actions beyond pure text generation

**Non-Goals (explicitly out of scope for this design):**
- Multi-modal (image, audio, video) input or output - text-only for v1
- Voice (ASR/TTS) interfaces
- Agentic multi-step orchestration beyond a single tool-calling loop (full agent design is a separate document)
- Model training, fine-tuning pipelines, or RLHF infrastructure
- Knowledge ingestion pipeline internals (covered separately; this design assumes embeddings are populated)

---

## 2. Glossary of Acronyms

Definitions for acronyms used throughout this document.

| Acronym | Expansion |
|---|---|
| **ANN** | Approximate Nearest Neighbour |
| **API** | Application Programming Interface |
| **ASR** | Automatic Speech Recognition |
| **AWQ** | Activation-aware Weight Quantization |
| **BM25** | Best Matching 25 (probabilistic sparse ranking function) |
| **CDN** | Content Delivery Network |
| **DDoS** | Distributed Denial of Service |
| **DLP** | Data Loss Prevention |
| **DSR** | Data Subject Request (GDPR) |
| **GDPR** | General Data Protection Regulation |
| **GGUF** | GPT-Generated Unified Format (quantization file format) |
| **GPTQ** | Generative Pretrained Transformer Quantization |
| **HIPAA** | Health Insurance Portability and Accountability Act |
| **HNSW** | Hierarchical Navigable Small World (graph-based ANN index) |
| **HyDE** | Hypothetical Document Embeddings |
| **JWT** | JSON Web Token |
| **KNN** | K-Nearest Neighbours |
| **KV Cache** | Key-Value cache (transformer attention state) |
| **LLM** | Large Language Model |
| **MCP** | Model Context Protocol (Anthropic open standard for tool/context integration) |
| **MoE** | Mixture of Experts |
| **mTLS** | Mutual Transport Layer Security |
| **OIDC** | OpenID Connect |
| **PII** | Personally Identifiable Information |
| **QPS** | Queries Per Second |
| **RAG** | Retrieval-Augmented Generation |
| **RLHF** | Reinforcement Learning from Human Feedback |
| **RPS** | Requests Per Second |
| **SDK** | Software Development Kit |
| **SLA** | Service Level Agreement (contractual) |
| **SLI** | Service Level Indicator (measured signal) |
| **SLO** | Service Level Objective (internal target) |
| **SSE** | Server-Sent Events |
| **TGI** | Text Generation Inference (HuggingFace serving) |
| **TPOT** | Time Per Output Token |
| **TPS** | Tokens Per Second |
| **TTFT** | Time To First Token |
| **TTL** | Time To Live |
| **TTS** | Text-To-Speech |
| **VRAM** | Video Random Access Memory (GPU memory) |
| **WAF** | Web Application Firewall |

---

## 3. Requirements

### 3.1 Functional
- Users can send messages and receive responses from an LLM
- System maintains conversation history for multi-turn context with stateful session memory
- Responses are streamed back token-by-token in real time
- Support both self-hosted and external LLM providers, with dynamic routing
- Augment responses with relevant context via RAG over a private knowledge base
- Support tool use / function calling for actions that require external data or side effects
- Filter unsafe, out-of-scope, or policy-violating inputs and outputs via Guardrails
- Provide message-level citations linking responses back to source documents
- Support graceful fallback when any downstream component (LLM, RAG, cache) is degraded

### 3.2 Non-Functional (with concrete targets)

| Dimension | Target |
|---|---|
| **Availability** | 99.9% monthly (≈ 43 minutes downtime / month) for the chat API |
| **TTFT (p50)** | ≤ 800ms end-to-end (client to first visible token) |
| **TTFT (p95)** | ≤ 1.5s end-to-end |
| **TPOT (steady-state)** | ≤ 50ms per token at p95 (sustained ≥ 20 tokens/sec perceived) |
| **Throughput** | 10,000 concurrent active sessions per region, autoscaling beyond |
| **Session durability** | Conversation history retained for 30 days hot, 1 year cold storage |
| **Cache hit rate** | ≥ 25% of qualifying read-only queries (FAQ-shaped traffic) |
| **Guardrails latency** | ≤ 50ms p95 added per check (input + output) |
| **RAG retrieval latency** | ≤ 200ms p95 (embed + ANN + rerank) |
| **Compliance** | SOC 2 Type II, GDPR data-resident routing, optional HIPAA-compliant deployment |
| **Cost ceiling** | Per-conversation cost tracked and alertable at a per-tenant level |

### 3.3 Constraints & Assumptions
- Multi-tenant from day one; tenant isolation at the data, configuration, and cost-attribution layer
- Operating across at least two regions (active-active or active-passive depending on tier)
- The knowledge base is pre-embedded and updated by an external ingestion pipeline
- LLM provider APIs may have variable latency and rate limits; the system must absorb this variability
- All user data is treated as confidential by default; sensitive data classification triggers self-hosted routing

---

## 4. High-Level Architecture

![LLM Chatbot Architecture](../images/chat-service/chat_service.png)

The system is organised into four logical planes:

- **Edge plane** - Client + API Gateway. Handles connection management, identity, traffic shaping, streaming.
- **Orchestration plane** - Chat Service. Owns the request lifecycle, session state, prompt assembly, and component coordination.
- **Augmentation plane** - Cache, RAG, Vector DB, Guardrails. Improves quality, reduces cost, and enforces safety.
- **Inference plane** - Model Router, Self-Hosted LLMs, External LLMs. Performs the actual generation.

The Chat Service is the only component aware of the entire request lifecycle; all other components are stateless or domain-scoped, which keeps them independently scalable, testable, and replaceable.

---

## 5. Core Components

| Component | Responsibility |
|---|---|
| **Client** | Captures user input, renders streamed responses |
| **API Gateway** | Edge concerns - auth, rate limiting, streaming proxy, observability |
| **Chat Service** | Orchestrator - session state, prompt construction, component coordination |
| **Cache** | Semantic cache to skip the LLM for similar prior queries |
| **RAG** | Retrieves grounding context from a knowledge base |
| **Vector DB** | Stores and indexes dense (and optionally sparse) embeddings |
| **Guardrails** | Input/output content moderation and policy enforcement |
| **Guardrails Model** | Classifier(s) backing Guardrails decisions |
| **Model Router** | Selects target LLM based on signals |
| **Model Selection Model** | Lightweight classifier informing routing |
| **Self-Hosted LLMs** | Open-weight models served on internal GPU infrastructure |
| **External LLMs** | Third-party hosted models (OpenAI, Anthropic, Google, etc.) |
| **Session Store** | Hot conversation state (Redis) + durable history (PostgreSQL) |

---

## 6. Component Deep Dive

### 6.1 Client

The client is the interface through which users interact with the chatbot. It captures input, manages the streaming connection, renders tokens as they arrive, and emits client-side telemetry. Its design directly affects perceived latency, which is often more important than backend latency.

**Form factors:**
- **Web Browser** - SPA built with React, Vue, Svelte, or Next.js, communicating over HTTP/2 or WebSocket; Server-Sent Events (SSE) is the simplest protocol for unidirectional token streams
- **Mobile App** - native iOS (Swift) or Android (Kotlin), or cross-platform via React Native, Flutter, or Kotlin Multiplatform; long-lived streaming connections must survive backgrounding and network transitions
- **Embedded Chat Window** - a widget injected into another product (support portal, docs site, IDE plugin, Slack/Teams app) via an SDK; expected to coexist with the host application's auth and styling

**Client responsibilities beyond rendering:**
- **Optimistic UI** - immediately echo the user's message before the backend acknowledges, so the interaction feels instant
- **Stream resumption** - handle dropped connections by resuming the in-progress response using a message ID and offset
- **Client-side telemetry** - emit `time-to-first-byte`, `tokens-per-second perceived`, and connection errors back to the observability pipeline; backend latency alone does not capture the user's experience
- **Markdown / code rendering** - LLM responses commonly contain markdown, syntax-highlighted code, and tables; the client should render these incrementally as tokens stream in without layout thrash
- **Input throttling and abort** - allow the user to cancel an in-flight generation, which propagates an HTTP/2 stream cancel to the backend so the LLM call is aborted (saving cost)

---

### 6.2 API Gateway

The API Gateway is the single entry point for all client requests. It sits between the client and the backend services, handling cross-cutting concerns so the Chat Service does not have to.

**Responsibilities:**
- **Authentication & Authorization** - validates identity (JWT, OIDC, OAuth 2.1, mTLS for service-to-service); enforces tenant- and role-scoped access control
- **Rate Limiting** - token bucket or leaky bucket per user, tenant, and IP; differentiates between request count and LLM token consumption (the latter is the more important cost driver)
- **DDoS Protection** - integrates with an upstream WAF or CDN (Cloudflare, AWS Shield, Akamai) to absorb volumetric attacks before they reach origin
- **Request Filtering** - rejects malformed, oversized, or disallowed payloads early; protects against payload-based exploits
- **Request Routing** - directs traffic by path, header, or version (`/v1/chat`, `/v2/chat`); supports header-based tenant routing
- **Load Balancing** - distributes traffic across Chat Service instances; for streaming, sticky session affinity is preferred to preserve in-flight connections
- **SSL/TLS Termination** - handles HTTPS at the edge so internal traffic can be mTLS or plain HTTP within the service mesh
- **Circuit Breaking** - opens the circuit on downstream failures to prevent cascading outages
- **Retry Logic** - retries idempotent requests with exponential backoff and jitter; non-idempotent operations (anything that mutates session state) must not be silently retried
- **Usage Metering** - meters requests and token consumption per user/tenant for quota enforcement, billing, and cost attribution
- **IP Allowlist / Blocklist** - network-level filtering before any application logic runs
- **Streaming Support** - LLM responses are streamed token-by-token; the gateway maintains long-lived HTTP/2 or WebSocket connections and proxies the stream without buffering. HTTP/2 SSE is the modern default - simpler than WebSockets and works through most CORS/proxy stacks unchanged

**AI-specific Gateway capabilities (worth calling out):**
A new class of "AI Gateway" products (Portkey, Kong AI Gateway, Helicone, LangSmith, Cloudflare AI Gateway) add LLM-aware features beyond a traditional gateway:
- Provider abstraction (call OpenAI, Anthropic, Google through one schema)
- Automatic fallback across providers when one is rate-limited or down
- Token-level cost accounting and budget enforcement
- Prompt and response logging with redaction
- Built-in caching at the gateway tier
- Request/response transformation between provider schemas

For greenfield deployments, an AI Gateway often replaces or augments the traditional API Gateway.

**Technologies:**
- *Open Source* - Kong, NGINX, Traefik, Envoy, Tyk; Kong AI Gateway, LiteLLM Proxy, Portkey OSS
- *Managed* - AWS API Gateway, Google Apigee, Azure API Management, Cloudflare API Shield, Cloudflare AI Gateway, Helicone, Portkey Cloud

---

### 6.3 Cache

Caching is the single most impactful cost lever in an LLM system. The cache sits in front of the LLM and serves prior responses for semantically equivalent queries, avoiding redundant model calls. This design uses Redis with vector search for **semantic caching**, complementing the **provider-side prompt caching** offered by external LLM providers.

**Two layers of caching to be aware of:**

1. **Application-level semantic cache (this section)** - Redis vector index keyed on query embedding; returns previously generated full responses
2. **Provider-level prompt cache** - Anthropic's prompt caching, OpenAI cached input tokens, Google Gemini context caching; reduces input token cost when the same prompt prefix (e.g., system prompt + RAG context) is sent repeatedly. The Chat Service should structure prompts to maximise prefix reuse (system prompt + stable context first, volatile content last)

These layers compose: semantic cache short-circuits the entire request, while provider-side caching reduces cost on requests that do reach the LLM.

**Semantic Cache - How it works:**
1. On a new request, the Chat Service embeds the query
2. RediSearch runs a KNN query against stored embeddings using cosine similarity
3. If the top match exceeds the configured threshold, the cached response is streamed back immediately
4. On miss, the request proceeds to the LLM; the (query embedding, response, metadata) tuple is written back to the cache

**Similarity Score Thresholds:**

Cosine similarity yields a score in `[0, 1]`. The threshold dials the trade-off between accuracy and hit rate:

- **High threshold (≥ 0.95)** - near-identical queries only; high precision, low hit rate
- **Medium threshold (0.85 – 0.95)** - catches paraphrased variants; sensible default for most production deployments
- **Low threshold (< 0.85)** - aggressive reuse; risks returning answers that miss the user's actual intent

There is no universal correct value - tune empirically using a labelled validation set of (query, intended-answer) pairs and measure precision-at-threshold per tenant or vertical.

**Embedding Models (2026-current):**

The embedding model used for the cache should match (or be compatible with) the model used for RAG to simplify infra. Same model required between write and read.

- *Open Source* - `BGE-M3` (multi-lingual, hybrid dense+sparse+ColBERT), `nomic-embed-text-v1.5` (Matryoshka, variable dims), `mxbai-embed-large-v1`, `NV-Embed-v2` (NVIDIA), Sentence Transformers (`all-mpnet-base-v2`, `all-MiniLM-L6-v2`)
- *Managed* - OpenAI `text-embedding-3-large` (3072 dims, supports Matryoshka), Cohere `embed-v4` (3072 dims, multi-modal), Voyage AI `voyage-3-large`, Google `gemini-embedding-001`

Matryoshka Representation Learning is worth highlighting: a single model produces vectors that can be truncated to lower dimensions (e.g., 3072 → 512) with graceful quality degradation - useful when storage or memory is the binding constraint.

**Data Structure in Redis:**

Each cache entry is a Redis Hash with an indexed VECTOR field via RediSearch (Redis 8 / Redis Stack):

- `query` - original query text (for audit and debugging)
- `response` - full LLM response (or pointer to object store for large responses)
- `embedding` - query vector (FLOAT32 array, dimension matches embedding model)
- `tenant_id` - tenant scope (entries are partitioned by tenant)
- `model_version` - LLM model version that generated the response (invalidate on model change)
- `created_at` - timestamp
- `ttl` - Redis-managed expiry

**Index choice:**
- **FLAT** - O(n) brute-force scan; acceptable up to ~100K vectors per tenant
- **HNSW** - O(log n) approximate search; ~1-5ms p95 over millions of vectors; the practical choice for production. Tune `M` (graph connectivity) and `efConstruction` / `efRuntime` to trade recall against latency

**Eviction policies:**

Redis offers `allkeys-lru`, `allkeys-lfu`, `volatile-lru`, `volatile-ttl`. For semantic cache, `volatile-lru` with explicit TTLs is the safest default - prevents accumulation of stale entries while keeping hot queries warm.

**Latency in a Redis Cluster:**
- FLAT - ~1-10ms for up to ~100K vectors; degrades linearly
- HNSW - ~1-5ms across millions of vectors
- Network RTT - ~0.1-1ms; negligible against LLM latency

**Cache invalidation:**

Whenever upstream knowledge changes (RAG corpus updated, system prompt revised, model version pinned), affected entries must be invalidated. Practical approaches:
- Tag entries with `(model_version, prompt_template_hash, corpus_version)`; invalidate by tag when any element changes
- Aggressive short TTLs (hours to a day) for high-churn knowledge bases
- Explicit cache busting on knowledge ingestion pipeline completion

**Technologies:**
- *Cache Store* - Redis (8.x+) with RediSearch / vector module, Redis Enterprise, AWS ElastiCache for Redis, Google Memorystore, Upstash Redis
- *Alternatives* - Momento (serverless cache with vector), KeyDB

---

### 6.4 RAG & Vector DB

**Retrieval-Augmented Generation (RAG)** injects relevant external knowledge into the prompt at query time, grounding the LLM in factual, up-to-date, domain-specific context rather than relying purely on parametric memory.

**Why RAG matters:**
- Reduces hallucinations by grounding the response in retrieved facts
- Enables answering over private or proprietary data the LLM was never trained on
- Decouples knowledge from the model - update the corpus without retraining
- Enables source attribution and citations - critical for trust and auditability
- Significantly cheaper than fine-tuning for keeping knowledge current

**RAG Pipeline - Two Phases:**

*Ingestion (offline, idempotent):*
1. Source documents (PDFs, wikis, CRM records, code, transcripts) are normalised
2. **Chunking** - split into retrievable units. Modern strategies:
   - *Fixed-size chunking* - simple, ~256-512 tokens per chunk
   - *Semantic chunking* - splits at natural boundaries (headings, paragraphs, semantic shifts)
   - *Contextual chunking* (Anthropic, 2024) - prepends an LLM-generated context summary to each chunk before embedding, dramatically improving retrieval recall on long documents
   - *Late chunking* (Jina, 2024) - chunk the embeddings rather than the text; the embedding model sees the full document context but produces per-chunk vectors
3. Each chunk is embedded and stored with metadata (`source`, `document_id`, `chunk_index`, `tenant_id`, `permissions`, `timestamp`)

*Retrieval (online, per query):*
1. **Query rewriting** - the raw user query may be ambiguous or terse; an LLM or rule-based rewriter expands or clarifies it. Variants include:
   - *HyDE (Hypothetical Document Embeddings)* - generate a hypothetical answer first, then embed and retrieve against that
   - *Multi-query expansion* - generate N paraphrases and union the retrieved sets
   - *Step-back prompting* - generate a more abstract query to find higher-level context
2. **Hybrid retrieval** - run both dense (vector) and sparse (BM25/SPLADE) retrieval in parallel; fuse results via Reciprocal Rank Fusion (RRF). Hybrid consistently outperforms pure dense search, especially on rare named entities, codes, and exact-match queries
3. **Metadata filtering** - apply tenant scope, permissions, recency, source filters before or during retrieval
4. **Reranking** - the top-K (e.g., K=50) candidates are re-scored by a more expensive cross-encoder reranker (Cohere Rerank v3.5, BGE Reranker, Jina Reranker, mxbai-rerank) which jointly encodes query+document for higher precision. Retain top-N (e.g., N=5-10) after reranking
5. **Context assembly** - retrieved chunks are formatted with citations and inserted into the prompt

**Advanced RAG Patterns (2026-current):**
- **Agentic RAG** - the LLM iteratively decides what to retrieve, in multiple rounds, rather than a single pass
- **Graph RAG** (Microsoft) - augments vector retrieval with a knowledge graph to capture entity relationships and enable multi-hop reasoning
- **ColBERT / multi-vector retrieval** - represents each document as multiple vectors with late interaction; higher retrieval quality at the cost of storage
- **Self-querying / metadata generation** - the LLM extracts structured filters from the user query before retrieval

**Embedding Models (2026-current):**

Same selection as the semantic cache; the same model should be used to keep the embedding infrastructure simple. The same embedding model must be used at ingestion and retrieval time - mixing models produces incomparable vectors and requires a full re-embedding migration to fix.

**Vector DB Technologies:**

| Tier | Options | Notes |
|---|---|---|
| Open Source, self-hosted | Qdrant, Weaviate, Milvus, ChromaDB, Vespa | Qdrant and Milvus are the most production-mature for high-QPS workloads; Vespa excels at hybrid search at scale |
| Postgres-native | pgvector, pgvectorscale (Timescale) | Best when you already run Postgres and want operational simplicity; pgvectorscale closes the performance gap with dedicated vector DBs |
| Managed | Pinecone, Zilliz Cloud, Weaviate Cloud, Qdrant Cloud, Vespa Cloud, MongoDB Atlas Vector Search, AWS OpenSearch with k-NN | Fully managed - no operational overhead but vendor lock-in and per-vector cost |
| Embedded | LanceDB, ChromaDB (embedded), DuckDB-VSS | Suitable for edge or single-tenant deployments |

**RAG Latency Budget:**

| Step | Approximate Time (p95) |
|---|---|
| Query rewriting (if used) | 100-300ms (extra LLM call) |
| Query embedding | 20-100ms (API); 5-20ms (local) |
| Hybrid retrieval (dense + sparse) | 10-50ms |
| Reranking (top-50 → top-10) | 30-150ms |
| Chunk fetch & prompt assembly | 10-30ms |
| **Total RAG pipeline** | **~80-300ms p95 without rewrite, 200-600ms with rewrite** |

Whether to run query rewriting is itself a routing decision - skip for short, clear queries; enable for ambiguous or multi-turn queries.

---

### 6.5 Guardrails

The Guardrails layer is a dedicated content moderation service powered by one or more classification models that sits at two critical points in the pipeline - **before** the request reaches the LLM and **after** the LLM generates a response. It is the system's safety net.

**Why this layer is non-negotiable:**

LLMs are inherently probabilistic and can be coerced into producing harmful, off-topic, or confidential content. Without dedicated moderation, the system is only as safe as the base model's own (often inconsistent) safety training. Guardrails provide an explicit, auditable, and independently tunable safety boundary that:
- Protects users from harmful LLM outputs
- Protects the system from adversarial inputs designed to exploit it
- Gives operators full control over what the chatbot will and will not engage with
- Builds user trust through predictable, on-policy behaviour
- Provides regulators an auditable trail of safety decisions

**Input Guardrails (request → LLM):**

- **Prompt Injection** - detects attempts to override the system prompt via embedded instructions (*"Ignore previous instructions..."*, indirect injection via tool outputs or retrieved documents). This is the most important threat to address; indirect injection via RAG-retrieved content is particularly dangerous
- **Jailbreaking** - identifies adversarial framing (roleplay, hypotheticals, obfuscation, encoded payloads) intended to bypass safety training
- **Topic Moderation** - classifies user intent against the application's allowed scope; out-of-scope requests get a structured refusal
- **PII Detection** - detects and optionally redacts personal data before logging or sending to external providers (especially relevant when the Router cannot route to a self-hosted model)
- **Toxicity / Harmful Intent** - blocks content seeking instructions for harm (CBRN, self-harm, illegal activity)
- **Secret / Credential Leakage** - detects API keys, tokens, passwords pasted in queries (and optionally in responses)

**Output Guardrails (LLM → user):**

- **Toxicity & Hate Speech Detection** - blocks responses with offensive, discriminatory, or harmful content
- **Hallucination & Factual Grounding Check** - verifies the response is grounded in the retrieved RAG context (e.g., via entailment models or LLM-as-judge); flags fabricated claims
- **PII Leakage Detection** - ensures the model has not surfaced sensitive data from RAG or training corpus
- **Policy Compliance** - checks adherence to required disclaimers (financial, medical, legal), formatting requirements, or brand voice
- **Output schema enforcement** - for structured outputs (JSON, function calls), validate against schema and either retry or repair on failure

**Streaming-aware moderation:**

A subtle but important point: when the response streams token-by-token, output guardrails cannot wait for the full response. Production systems use one of:
- **Token-window buffering** - moderate over sliding windows (e.g., last 50 tokens); higher latency but lower risk
- **Sentence-boundary moderation** - emit moderated tokens up to the last completed sentence
- **Streaming classifier** - dedicated low-latency classifier evaluating partial output incrementally
- **Post-hoc verification with rollback** - emit tokens optimistically; if a violation is detected mid-stream, cut the stream and return a structured refusal (user experience trade-off)

**Technologies (2026-current):**
- *Open Source* - NeMo Guardrails (NVIDIA), Guardrails AI, Llama Guard 3 (Meta), Granite Guardian (IBM), Prompt Guard (Meta), Microsoft Presidio (PII), PyRIT (red-teaming)
- *Managed / Closed Source* - Azure AI Content Safety, AWS Bedrock Guardrails, OpenAI Moderation API, Google Cloud Model Armor, Lakera Guard, Robust Intelligence, Protect AI

**Audit and tunability:**

Every guardrail decision should be logged with: the input, the classifier scores, the decision, the policy version, and the timestamp. This enables:
- Continuous tuning of thresholds based on false positive / negative rates
- Compliance reporting for regulated industries
- Red-team / blue-team workflows for security teams

---

### 6.6 Model Router & Model Selection

The Model Router decides which LLM handles a given request. Rather than hardcoding every request to a single model, the router dynamically balances cost, latency, capability, and data sensitivity. The decision is informed by a **Model Selection Model** - a lightweight classifier that scores the request and recommends a model tier.

**Why dynamic routing matters:**
- Not every query needs a frontier model; FAQ-shaped queries can be answered by a small fast model at 1/100th the cost
- Sensitive or region-restricted data must be routed to a compliant deployment
- Frontier models periodically face outages and rate limits; fallback routing preserves availability
- Different models excel at different tasks (code, math, multilingual, long-context); routing on intent improves quality

**Routing Signals:**

- **Task complexity** - simple (lookup, summary) vs. complex (reasoning, multi-step, code)
- **Query intent / task type** - code, math, translation, summarisation, conversation, retrieval QA
- **Cost budget** - per-tenant or per-conversation cost ceilings
- **Latency requirements** - user-facing interactive vs. background or asynchronous
- **Data sensitivity** - PII flags or tenant policy steering to self-hosted models
- **Context length** - very long prompts force selection of long-context models
- **Tool use requirements** - some smaller models do not reliably call tools or produce structured output

**Routing Strategies:**

- **Rule-based routing** - deterministic predicates (`token_count > 4000 → long-context model`); fast (~1ms), predictable, easy to audit; the workhorse for hard constraints (data residency, context length)
- **ML-based routing** - a lightweight classifier (DistilBERT, fastText, embedding-based KNN) trained on labelled (query, best-model) pairs; ~10-30ms overhead; adaptive but requires ongoing label collection
- **Cascade routing** - try a cheap model first; if confidence is low (or the response fails a verifier), escalate to a more capable model; reduces average cost but adds tail latency
- **Speculative routing** - issue requests to multiple models in parallel and use whichever returns first or scores highest; trades cost for latency reliability
- **Hybrid** - rules for hard constraints, ML for nuanced quality decisions; the most common production pattern

**Self-Hosted vs. External LLMs:**

Self-hosted models (Llama 4, Mistral, Qwen 2.5, DeepSeek V3) run on your own GPU infrastructure - data never leaves your environment, latency is predictable, and per-token cost is fixed by hardware utilisation rather than provider pricing. They are the right choice for sensitive workloads, regulated industries, and high-volume traffic where amortised infrastructure cost beats per-token billing. The trade-off is operational burden: hosting, scaling, capacity planning, GPU procurement, model versioning, and 24/7 on-call. External providers (OpenAI GPT-5 family, Anthropic Claude Opus 4 / Sonnet 4.5, Google Gemini 2.5 Pro) offer state-of-the-art capabilities with zero infrastructure burden in exchange for per-token pricing and third-party data exposure.

**Data residency** is a particularly significant driver. Regulations such as GDPR, HIPAA, the EU AI Act, and country-specific data sovereignty laws may mandate that certain data never crosses regional boundaries or leaves the organisation's infrastructure. The Model Router must be aware of data classification and routing policy - sensitive or region-restricted requests must always go to a compliant deployment regardless of cost or capability preferences. This is a non-negotiable rule, not an optimisation.

**Technologies:**
- *Open Source / Self-Hostable Routing* - LiteLLM, OpenRouter (proxy + routing), Portkey OSS, RouteLLM
- *Managed Routing* - Martian, Not Diamond, Unify, Portkey, Helicone, Anyscale

---

### 6.7 LLM Serving - Self-Hosted & External Models

Once the Router selects a model, the request goes to the inference layer for generation. This is the most computationally intensive and latency-dominant part of the pipeline.

**Self-Hosted Models (2026-current):**

Open-weight model families currently used in production:

- *Llama 4 (Meta)* - latest open-weight family from Meta, native MoE architecture, multimodal-capable, very strong general-purpose model with long-context support
- *Mistral Large 2 / Mixtral / Pixtral (Mistral AI)* - efficient MoE designs; Mixtral provides strong performance at a fraction of dense-model compute
- *Qwen 2.5 / 3 (Alibaba)* - state-of-the-art on coding, math, and multilingual benchmarks; available across a wide size range (0.5B to 72B)
- *DeepSeek V3 / R1 (DeepSeek)* - DeepSeek V3 is a highly efficient MoE model; R1 is a strong reasoning-focused open-weight model competitive with closed frontier models on math and code
- *Gemma 3 (Google)* - lightweight, efficient models optimised for on-device and edge inference
- *Phi-4 (Microsoft)* - small (~14B) but punches well above its weight on reasoning benchmarks

**Serving infrastructure:**

- *vLLM* - the de facto standard high-throughput serving framework; PagedAttention for efficient KV cache management, continuous batching, prefix caching, speculative decoding
- *SGLang* - competitive alternative with strong structured-output and RadixAttention (KV-cache reuse) support
- *TGI (Text Generation Inference)* - HuggingFace's production server; streaming, continuous batching, tensor parallelism
- *TensorRT-LLM (NVIDIA)* - highest throughput on NVIDIA hardware with kernel-level optimisations; significant compile-time overhead
- *LMDeploy* - efficient serving with strong quantization support
- *Ollama* - lightweight runner ideal for development and low-traffic deployments
- *Triton Inference Server (NVIDIA)* - enterprise-grade multi-model, multi-framework serving

**Hardware:**

| GPU | VRAM | Notes |
|---|---|---|
| NVIDIA H100 | 80 GB | Production workhorse 2023-2025 |
| NVIDIA H200 | 141 GB | Higher memory bandwidth, fits 70B in fp16 on a single GPU |
| NVIDIA B100 / B200 | 192 GB | Blackwell generation - significantly higher throughput |
| AMD MI300X / MI325X | 192 GB / 256 GB | Strong fp16 perf at competitive price; ROCm ecosystem maturing |
| AWS Trainium / Inferentia2 | varies | Cost-efficient for AWS-resident workloads |
| Google TPU v5p / Trillium | varies | Strong for Gemini and JAX-based workloads |
| Apple Silicon (MLX) | unified | Mac M-series for dev / single-user inference |

**Inference-time optimisations (essential to call out):**

- **Quantization** - GPTQ, AWQ, GGUF, INT8, INT4, FP8. A 70B model in fp16 needs ~140 GB; quantised to INT4 fits in ~35 GB on a single H100 at a modest quality cost
- **Continuous batching** - dynamically batch concurrent requests at the token level for much higher throughput than naive request-level batching
- **PagedAttention / KV cache management** - allocate KV cache in pages to avoid memory fragmentation and enable efficient sharing
- **Prefix caching / RadixAttention** - reuse KV cache across requests that share a common prompt prefix (system prompt, RAG context); huge cost win for the common case where prefixes are stable
- **Speculative decoding** - a smaller draft model proposes multiple tokens; the larger target model verifies in parallel. Often 2-3x throughput improvement with no quality loss
- **Tensor / pipeline / expert parallelism** - split a model across multiple GPUs to serve larger models or improve throughput

**External Models (2026-current):**

Major providers and their flagship models:

- *OpenAI* - GPT-5 family (frontier reasoning), GPT-5 mini (cost-efficient), o-series reasoning models (extended thinking for complex problems), GPT-5 Turbo (balanced)
- *Anthropic* - Claude Opus 4 (top-tier coding and reasoning, long-context), Claude Sonnet 4.5 (best balance of capability and speed for production workloads), Claude Haiku 4 (low-latency, cost-efficient)
- *Google* - Gemini 2.5 Pro (deep reasoning, very large context window up to 2M tokens), Gemini 2.5 Flash (fast and cost-efficient), Gemini 2.5 Flash-Lite (cheapest, smallest)
- *DeepSeek* - DeepSeek V3 and R1 via API; competitive on price-performance
- *Mistral AI* - Mistral Large 2, Mistral Small 3 (cost-efficient)
- *Cohere* - Command R+ (optimised for RAG and enterprise retrieval), Command A
- *xAI* - Grok 4 (frontier reasoning with X-platform integration)

**Provider-side features to leverage:**
- **Prompt caching** - Anthropic, OpenAI, and Google all offer caching of input tokens for repeated prompt prefixes; up to ~90% cost reduction and significant latency improvement on cached portions. The Chat Service should structure prompts to maximise prefix reuse
- **Batch API** - asynchronous batch endpoints (OpenAI, Anthropic) at 50% discount for non-interactive workloads (background summarisation, eval runs)
- **Structured outputs** - native JSON Schema constrained decoding (OpenAI, Anthropic tools, Google function calling) replaces brittle regex parsing
- **Tool use / function calling** - first-class structured tool invocations; combined with MCP (Model Context Protocol) for standardised tool integration

**Key considerations for external models:**
- **TTFT** - varies by provider, region, and load; typically 200ms-2s+
- **Rate limits** - per-minute token and request limits; production systems need queue management and retry budgets
- **Cost** - per-input and per-output token; output tokens are typically 3-5x more expensive than input tokens. Long contexts (RAG payloads, conversation history) compound cost; provider-side prompt caching is essential at scale
- **Model versioning** - providers periodically deprecate model versions; pin model versions and plan migrations
- **Provider dependence** - SLA and pricing are at the provider's discretion; the Router's fallback paths are the only mitigation

---

### 6.8 Chat Service

The Chat Service is the central orchestrator and the brain of the system. It is the only component aware of the entire request lifecycle - it talks to the API Gateway, Cache, Guardrails, RAG, Model Router, session store, and observability pipeline.

**Session & Conversation Management:**

Every user interaction belongs to a session. The Chat Service:
- Retrieves conversation history from the session store (Redis for hot, PostgreSQL for durable)
- Tracks running token count and triggers truncation or summarisation when approaching context limits
- Persists session metadata (user ID, tenant ID, session ID, timestamps, model used, total tokens, cost) for observability and billing
- Supports conversation resumption across device or connection changes via session ID

**Prompt Construction:**

Assembling the final prompt is a deliberate composition, not concatenation. The order matters both for quality and for provider-side prompt caching (stable prefixes maximise cache hits):

1. **System prompt** (most stable) - persona, scope, hard rules, tool descriptions
2. **RAG context** (per-query but cacheable in some cases) - retrieved chunks with citation markers
3. **Conversation history** (grows over time) - prior turns, trimmed or summarised to fit budget
4. **Current user message** (volatile)
5. **Tool definitions / response schema** - if using tool use or structured outputs

Prompt templates are versioned and stored as code artefacts - never inlined into the service. Every prompt change is a deployable change, eligible for rollout, A/B testing, and rollback.

**Orchestration Flow:**

1. Receive request from the API Gateway
2. Load session history (parallel with cache lookup)
3. Semantic cache lookup - on hit, stream cached response and return
4. Input Guardrails on the user message
5. RAG retrieval (can be parallelised with input guardrails to save latency)
6. Construct final prompt
7. Dispatch to Model Router → selected LLM
8. Stream tokens, applying output guardrails in a windowed fashion
9. Persist new message pair to session store; write to semantic cache
10. Emit telemetry (latency per stage, tokens, model, guardrail outcomes, cost)

**Tool Use / Function Calling (MCP):**

Modern chatbots need to do more than generate text - they call APIs, query databases, search the web, and take actions. The Chat Service supports tool use via:
- **Tool registry** - declarative tool definitions (name, JSON schema, description) injected into the prompt
- **Tool call detection** - parse structured tool calls from the LLM output
- **Tool execution** - call the underlying API/function with appropriate auth scoping
- **Result injection** - feed tool results back into the conversation for a follow-up generation
- **MCP (Model Context Protocol)** - Anthropic's open standard for tool/context integration. MCP servers expose tools, resources, and prompts that any MCP-aware client (Claude Desktop, IDEs, Chat Services) can connect to. Adopting MCP standardises tool integration and decouples tool implementation from the Chat Service

A single tool-use loop typically runs 1-3 iterations; agentic systems run many more. This design supports a bounded loop (max iterations configurable per tenant) - true open-ended agents are out of scope.

**Context Window Management:**

- **Sliding window** - retain most recent N turns; simple, cheap, lossy
- **Summarisation** - compress older turns into a brief summary that is prepended to context; preserves continuity at the cost of an extra LLM call
- **Hierarchical memory** - recent turns verbatim + summary of older turns + long-term memory pinned facts
- **Token counting** - use the target model's tokenizer (tiktoken for OpenAI, the model's own tokenizer otherwise) to ensure prompts never exceed limits

**Streaming:**

The Chat Service does not buffer the full response. It opens a streaming connection to the LLM (SSE for most providers) and pipes tokens to the client via the API Gateway in real time. This drives perceived latency: the user sees the response begin within hundreds of milliseconds even for long generations. Streaming concerns include:
- Cancellation propagation (user clicks stop → cancel LLM call → release GPU capacity)
- Output guardrails on partial output (see 6.5)
- Reconnection / resume semantics (message ID + token offset)

**Error Handling & Fallbacks:**

- LLM unavailable / rate-limited → Model Router falls back to alternate provider
- RAG retrieval times out → proceed without RAG context (with a confidence note)
- Guardrails fail open vs. closed - sensitive deployments fail closed (return a refusal); low-risk deployments may fail open to preserve availability
- Streaming connection drops → in-memory write of completed tokens; client can resume

**Multi-tenancy:**

The Chat Service is the natural enforcement point for tenant isolation:
- Tenant-scoped session stores (partitioned by tenant ID)
- Tenant-scoped cache keys
- Tenant-scoped RAG corpora (filter by `tenant_id` at query time)
- Tenant-scoped guardrail policies
- Tenant-scoped model routing rules
- Tenant-scoped cost and rate limits

**Technologies:**
- *Orchestration Frameworks* - LangGraph (LangChain), LlamaIndex, Semantic Kernel (Microsoft), Haystack, DSPy (declarative prompting)
- *MCP* - Anthropic MCP SDKs (Python, TypeScript), reference MCP servers
- *Custom Services* - FastAPI (Python), Node.js, Go - preferred when team wants framework-free orchestration
- *Session Store* - Redis (hot context), PostgreSQL (durable), DynamoDB or Spanner for global multi-region
- *Streaming* - SSE for unidirectional token streams (default), WebSocket for bidirectional (e.g., voice or tool callbacks)

---

## 7. Data Flow

1. **User sends a message** - the client (web, mobile, or embedded widget) captures the user's input and sends it to the backend over HTTP/2 with SSE, along with a session identifier and an idempotency key.

2. **API Gateway processes the request** - SSL/TLS termination, authentication (JWT/OIDC), rate-limit and IP checks, payload validation. On success, the request is forwarded to the Chat Service and a streaming connection is held open back to the client.

3. **Chat Service loads session context** - conversation history is fetched from Redis (recent turns) and PostgreSQL (durable history if needed). Token counts are computed; truncation or summarisation runs if the window is full.

4. **Semantic cache lookup** - the query is embedded and a KNN search is run against the tenant-scoped Redis vector index. A hit above threshold short-circuits the pipeline - the cached response is streamed back and the request ends here.

5. **Input Guardrails** - on cache miss, the user message is sent through the input Guardrails layer (prompt injection, jailbreaking, topic, PII, secrets). Flagged messages return a structured refusal and the request stops.

6. **RAG retrieval** - the Chat Service runs hybrid retrieval (dense + sparse) against the tenant-scoped Vector DB, applies metadata filters (permissions, recency, tenant), reranks the top-K candidates with a cross-encoder, and selects the top-N for prompt injection. Steps 5 and 6 can be parallelised - input guardrails do not depend on RAG.

7. **Prompt construction** - the final prompt is assembled with stable elements first (system prompt, tool definitions, RAG context) and volatile elements last (history, user message). Token count is verified against the target model's limit.

8. **Model Router selects the LLM** - rules apply first (data sensitivity, context length, hard constraints), then the Model Selection Model scores the request for nuanced quality decisions. The selected provider's API client is invoked, with retry and fallback policies attached.

9. **LLM generates and streams the response** - the model generates token by token. The Chat Service consumes the stream and forwards it downstream while simultaneously buffering for output guardrail windows.

10. **Output Guardrails (streaming)** - as tokens stream in, the output Guardrails layer evaluates sliding windows or sentence boundaries. If a violation is detected mid-stream, the stream is cut and a safe fallback response is returned.

11. **Response streamed to the client** - clean tokens flow through the API Gateway to the client via SSE in real time. The user sees the response begin appearing within the first 500-800ms (TTFT target).

12. **Session and cache updated, telemetry emitted** - on completion, the message pair is persisted to Redis (hot) and PostgreSQL (durable). The query embedding and response are written to the semantic cache with appropriate TTL and version tags. Structured logs, metrics, traces, and cost/token telemetry are emitted to the observability pipeline.

**Parallelism summary:** Steps 5 (input guardrails) and 6 (RAG) run in parallel. Step 3 (history load) overlaps with step 4 (cache lookup). Step 12's telemetry and persistence happen asynchronously and do not block the response stream to the user.

---

## 8. Observability & Monitoring

LLM systems have unique observability needs - in addition to traditional service metrics (latency, errors, throughput), we must observe model behaviour (token usage, quality, drift, guardrail outcomes, cost). The Chat Service is the natural emission point for end-to-end request-level telemetry.

### 8.1 Logs

Structured JSON logs per request, correlated by `request_id`, `session_id`, `tenant_id`, `user_id` (hashed):

- Incoming request metadata (timestamp, route, source IP class, user agent)
- Per-stage timings (auth, cache lookup, guardrails in, RAG, prompt build, LLM TTFT, LLM total, guardrails out)
- Cache hit/miss with similarity score
- RAG: top-K document IDs, retrieval scores, rerank scores, retrieved chunk sizes
- Model routing: chosen model, confidence, fallback chain (if any)
- Guardrails: classifier scores, decision, policy version
- LLM: input tokens, output tokens, cost, finish reason
- Errors: stack trace, retry count, downstream service

**Logging stack:** Structured logs to a sink such as OpenTelemetry Collector → Loki / Elasticsearch / Datadog Logs / Splunk; redact PII at the edge before persistence.

**Prompt & response logging:** Full prompts and responses should be logged conditionally (sampled, or always-on for flagged tenants), with PII redaction and tenant-scoped access control. This is non-negotiable for debugging quality issues but is a privacy-sensitive surface - treat with the same care as a credentials log.

### 8.2 Metrics

Time-series metrics exposed via Prometheus, OpenTelemetry, or equivalent. Standard service metrics (RED - Rate, Errors, Duration) plus LLM-specific:

**Service-level:**
- `chat_requests_total` (by tenant, route, status)
- `chat_errors_total` (by tenant, error class, downstream component)
- `chat_request_duration_seconds` (histogram, by stage)

**LLM-specific:**
- `llm_ttft_seconds` (histogram, by model, provider)
- `llm_tpot_seconds` (histogram, by model)
- `llm_input_tokens_total` (by tenant, model)
- `llm_output_tokens_total` (by tenant, model)
- `llm_cost_usd_total` (by tenant, model)
- `llm_provider_errors_total` (by provider, error class - distinguishes rate limit, timeout, model overload)

**RAG:**
- `rag_retrieval_duration_seconds` (histogram, by stage)
- `rag_top1_score` (histogram - tracks corpus drift)
- `rag_rerank_uplift` (histogram - difference between dense top-1 and post-rerank top-1)

**Cache:**
- `cache_hit_ratio` (by tenant)
- `cache_similarity_at_hit` (histogram)
- `cache_size_entries` (gauge, by tenant)

**Guardrails:**
- `guardrails_decisions_total` (by direction, category, decision, policy version)
- `guardrails_classifier_score` (histogram, by category)
- `guardrails_latency_seconds` (histogram)

**Routing:**
- `routing_decisions_total` (by chosen_model, signal)
- `routing_fallback_total` (by primary_model, fallback_model, reason)

### 8.3 Traces

Distributed tracing via OpenTelemetry; one trace per user request, spanning the API Gateway, Chat Service, Guardrails, RAG, Cache, and LLM call. Spans include:

- `gateway.auth`, `gateway.rate_limit`
- `chat.load_session`, `chat.cache_lookup`, `chat.guardrails_in`
- `rag.embed`, `rag.dense_search`, `rag.sparse_search`, `rag.rerank`
- `chat.prompt_build`, `chat.route_model`
- `llm.generate` (with `model`, `provider`, `input_tokens`, `output_tokens`, `ttft`)
- `chat.guardrails_out` (per window)
- `chat.persist`, `chat.cache_write`

Trace backends: Jaeger, Tempo, Datadog APM, Honeycomb, New Relic.

### 8.4 LLM-Specific Observability Platforms

A new category of LLM observability tools provides purpose-built support beyond generic APM:

- *Open Source* - Langfuse, OpenLLMetry (Traceloop), Phoenix (Arize), Helicone
- *Managed* - LangSmith (LangChain), Helicone, Portkey, Datadog LLM Observability, Arize Phoenix Cloud, Braintrust

These tools add prompt-version-aware tracing, response evaluation dashboards, cost breakdowns by tenant/model, and integration with eval pipelines.

### 8.5 Cost Observability

LLM systems can accrue cost in surprising, fast-moving ways. Cost telemetry is first-class:

- Cost per request, attributed to tenant, model, and feature path
- Cost per session and per conversation
- Cost per cached vs. non-cached request (justifies caching investment)
- Budget burn rate per tenant, alertable
- Anomaly detection on cost-per-request (catches prompt regressions that bloat token counts)

---

## 9. Alerting & SLOs

### 9.1 Service Level Indicators (SLIs)

The measured signals that we care about:

- **Availability SLI** - successful chat requests / total chat requests (excluding client errors)
- **TTFT SLI** - fraction of requests where TTFT ≤ threshold (e.g., 800ms p95)
- **TPOT SLI** - fraction of requests where steady-state token rate ≥ threshold
- **Response completion SLI** - fraction of requests that complete without stream interruption
- **Cache hit SLI** - cache hits / cacheable requests
- **Guardrails false-positive SLI** - human-reviewed false positives / total blocks (target lower bound)
- **Quality SLI** - LLM-as-judge or human-eval score on a sampled audit set

### 9.2 Service Level Objectives (SLOs)

Internal targets backed by error budgets:

| SLO | Target | Window |
|---|---|---|
| Chat API availability | 99.9% | 30 days rolling |
| TTFT p95 ≤ 1.5s | 99% of requests | 30 days rolling |
| End-to-end success | 99.5% | 30 days rolling |
| Guardrails added latency p95 ≤ 50ms | 99% | 7 days rolling |
| RAG retrieval p95 ≤ 200ms | 99% | 7 days rolling |
| Quality score ≥ X (LLM-as-judge) | 95% of sampled requests | 7 days rolling |

Each SLO has an error budget. Burn rate alerts fire on fast (5m) and slow (1h, 6h) burn windows per Google SRE practice.

### 9.3 Alerting Strategy

Two tiers of alerts:

**Page (wake the on-call):**
- Multi-burn-rate SLO violations indicating budget will be exhausted within the window
- Total chat API outage (health check failures across all regions)
- All LLM providers returning errors (routing has no viable target)
- Critical security: prompt injection success rate spike, mass PII leakage detected
- Cost burst: tenant or system-wide cost exceeds budget by >2x in a 5-minute window

**Ticket (next business hour):**
- Cache hit rate degradation
- Quality score degradation on the eval set
- Single LLM provider degradation (auto-fallback in place)
- Slow burn rate (likely to exhaust monthly budget if unchecked)
- Knowledge base staleness signals

**Anti-pattern to avoid:** alerting on every classifier flag or every model error. These should drive dashboards and tickets, not pages. Alerts must be actionable and reversible.

### 9.4 Runbook Discipline

Each alert is linked to a runbook with:
- The metric being measured and its threshold
- Likely causes (provider outage, deploy regression, traffic spike, corpus update)
- Diagnostic commands and dashboards
- Mitigation steps (failover, rollback, route override)
- Escalation contacts

Synthetic monitoring runs canary conversations end-to-end every minute from multiple regions; this catches whole-system regressions that component-level metrics miss.

---

## 10. Evaluation & Quality

Quality cannot be measured by uptime alone. LLM systems require continuous evaluation across multiple dimensions.

### 10.1 Offline Evals

Run before every deploy and on every prompt or model change:

- **Golden datasets** - curated (input, expected-output) pairs per task and per tenant; track exact-match, semantic similarity, and BLEU/ROUGE where appropriate
- **Regression suites** - known-hard cases that previously broke and must continue to pass
- **Adversarial sets** - prompt-injection and jailbreak attempts; verify guardrails block them
- **RAG eval** - retrieval precision/recall@K against labelled relevant documents; faithfulness (response grounded in retrieved chunks); answer correctness

**Eval frameworks (2026-current):** LangSmith, Braintrust, Arize Phoenix, Ragas, DeepEval, OpenAI Evals, Helicone, Patronus AI.

### 10.2 Online Evals & A/B Testing

- Shadow traffic - the new prompt/model runs in parallel with production; responses are compared but only production responses are returned to users
- A/B testing - traffic is split between variants; aggregate metrics (CSAT, completion rate, follow-up rate, cost) compared
- Canary rollout - new versions exposed to 1% → 10% → 100% with automatic rollback on quality regression

### 10.3 LLM-as-Judge

Use a strong model (often a frontier model from a different provider than the production one, to avoid same-model bias) to grade response quality on dimensions like correctness, helpfulness, groundedness, safety, and tone. LLM-as-judge correlates well with human judgement at a fraction of the cost - but must be calibrated against periodic human review.

### 10.4 Continuous Quality Monitoring

- Sample 1-5% of production traffic for ongoing automated grading
- Flag low-scoring responses for human review
- Track quality scores per: tenant, model, prompt version, RAG corpus version, time
- Detect drift - if quality drops without a deploy, investigate corpus updates, traffic mix changes, or provider model updates

---

## 11. Security

### 11.1 Data Protection

- **In transit** - TLS 1.3 everywhere; mTLS for service-to-service inside the mesh
- **At rest** - encryption with customer-managed keys (KMS, HSM-backed) for session stores, vector DB, and logs
- **PII minimisation** - detect and redact PII at the edge (Microsoft Presidio or equivalent) before logging or routing to external providers
- **Right to deletion (GDPR DSR)** - data deletion APIs that propagate to all stores including the semantic cache and vector embeddings (re-ingestion required for affected documents)

### 11.2 Threat Model

Specific to LLM systems:

- **Prompt injection (direct and indirect)** - the primary application-layer threat. Indirect injection via RAG-retrieved documents is the most dangerous variant - an attacker who can write to your knowledge base can hijack your assistant. Mitigations: content-source provenance, instruction isolation, never executing tools based on instructions found in retrieved content without user confirmation
- **Data exfiltration via responses** - the LLM is tricked into revealing system prompts, other users' data, or RAG context for documents the user shouldn't access. Mitigations: strict tenant scoping on RAG queries, output guardrails for sensitive data classes, system prompt isolation
- **Model denial of service** - adversaries craft expensive queries (large context, complex reasoning) to drive up cost or saturate capacity. Mitigations: rate limits in tokens (not just requests), per-tenant cost ceilings, complexity-aware routing
- **Tool abuse** - LLM persuaded to call tools maliciously. Mitigations: scoped tool credentials, human-in-the-loop for sensitive tools, audit logs for all tool invocations
- **Supply chain** - compromised model weights, dependency vulnerabilities in serving infra. Mitigations: model signing, SBOM tracking, vulnerability scanning

### 11.3 Audit & Compliance

- Immutable audit logs for guardrail decisions, routing decisions, and tool invocations
- Access control: who can see which conversations, who can change prompts, who can deploy models
- Compliance certifications: SOC 2, ISO 27001, GDPR, HIPAA (if applicable), EU AI Act high-risk classifications
- Red-team exercises on a regular cadence using tools like Microsoft PyRIT, Garak (NVIDIA), Promptfoo red-team

---

## 12. Scalability & Capacity Planning

The chat pipeline has very different scaling characteristics per component:

| Component | Scaling Bottleneck | Strategy |
|---|---|---|
| **API Gateway** | Concurrent connections | Horizontal scale; sticky sessions for streaming |
| **Chat Service** | CPU + memory (orchestration, prompt build) | Stateless horizontal scale; autoscale on CPU and active sessions |
| **Cache (Redis)** | Memory + RediSearch CPU | Redis Cluster with hash-slot sharding; tenant-aware sharding |
| **Vector DB** | Memory (vectors in RAM) + query CPU | Shard by tenant or hash; replicas for read scale |
| **Guardrails** | GPU inference | Co-locate with model serving GPU; batch evaluation |
| **Self-hosted LLM** | GPU memory + bandwidth | Hardest to scale - capacity planning critical |
| **Session Store (Postgres)** | Write throughput on hot tenants | Partition by tenant; read replicas; archive old conversations |

**Capacity planning for self-hosted LLMs is the hard problem.** GPU procurement has long lead times; spot capacity is unreliable for production. Practical approaches:

- Provision base capacity for steady-state p50 traffic
- Burst capacity via external providers (the Model Router auto-fallbacks when self-hosted is saturated)
- Continuous batching and prefix caching to extract more throughput from existing GPUs
- Multi-region active-active for resilience and latency; route to nearest region with capacity

**Multi-region considerations:**
- Session store replication strategy depends on consistency requirements (strong vs. eventual)
- Vector DB and cache typically have a per-region copy; ingestion writes propagate
- Routing should prefer in-region LLM endpoints to minimise cross-region latency

---

## 13. Trade-offs & Considerations

### 13.1 RAG vs. Fine-Tuning
RAG and fine-tuning are both ways to specialise a model for a domain, but they serve different needs. RAG keeps knowledge external and updatable - adding new documents to the Vector DB immediately improves responses without touching the model. Fine-tuning bakes knowledge into the model's weights, which produces faster inference (no retrieval step) and better stylistic alignment, but requires expensive retraining every time the knowledge changes. For most production chatbots, RAG is the default; fine-tuning is reserved for cases where response style, tone, or domain vocabulary needs deep alignment that prompting alone cannot achieve. A hybrid - RAG on top of a lightly fine-tuned model - is increasingly common for high-stakes deployments.

### 13.2 Semantic Cache Threshold - Accuracy vs. Hit Rate
The similarity threshold dials directly between response accuracy and cache efficiency. A high threshold (≥ 0.95) serves only genuinely equivalent queries; low hit rate, low risk. A low threshold (< 0.85) maximises reuse but risks returning a response that misses the user's intent. Tune empirically against a labelled validation set per tenant. Cached responses do not reflect knowledge updates after write time, so TTL management and version tagging are critical.

### 13.3 Guardrails - Safety vs. Latency
Every guardrail check adds latency. Running synchronous input and output checks adds two blocking round-trips to the critical path. For safety-critical deployments, this is non-negotiable. For lower-risk internal tools, output guardrails can run asynchronously (log-and-flag rather than block), trading safety for speed. Lighter classifier models or windowed evaluation of streaming output offer middle-ground options. The right balance depends entirely on the deployment's risk profile.

### 13.4 Model Routing - Quality vs. Cost vs. Latency
Routing sits at the intersection of three competing forces. Sending every request to the frontier model maximises quality but is prohibitively expensive. Routing everything to the cheapest model reduces cost but degrades responses on complex queries. Latency adds a third dimension - a fast mid-tier model often produces a better user experience than a slower frontier model. Misrouting - especially under-routing complex queries - is invisible in metrics until quality dashboards or user feedback catches it. Continuous evaluation is the only reliable guardrail.

### 13.5 Self-Hosted vs. External LLMs
Self-hosted models give full control over data, cost structure, and latency, but come with significant operational burden - GPU infrastructure, model versioning, scaling, capacity planning, on-call. External providers offload all of that in exchange for per-token cost, third-party data exposure, and dependence on provider availability and pricing. For most startups, external providers are the right default. For enterprises handling sensitive data or operating at very high token volumes, self-hosted becomes increasingly attractive. The most common production pattern is hybrid - external for general traffic, self-hosted for sensitive or high-volume workloads, with the Router enforcing the boundary.

### 13.6 Context Window Management - Truncation vs. Summarisation
Truncation (drop oldest messages) is cheap but loses context. Summarisation preserves continuity but adds latency (an extra LLM call) and risks information loss in the summary. Truncation is the simpler engineering choice; summarisation is the better experience. A hybrid - truncate by default, summarise when the session ages past a threshold - works well in practice. Provider-side prompt caching makes summarisation cheaper than it used to be, since the summary becomes a stable prefix that gets cached.

### 13.7 Streaming vs. Buffered Responses
Streaming dramatically reduces perceived latency but adds meaningful infrastructure complexity - long-lived connections, partial-output guardrails, mid-stream error handling. For conversational UX, streaming is almost always worth it. For API integrations that need structured outputs or downstream parsing, buffered responses are simpler and safer. Both can coexist if the API supports an opt-in `stream: true` flag.

### 13.8 Vector Index - Recall vs. Speed
FLAT guarantees finding true nearest neighbours but scans linearly; impractical beyond ~100K vectors. HNSW finds approximate nearest neighbours in O(log n) with 95-99% recall - the production standard at scale. The small recall gap means HNSW may occasionally miss a relevant chunk or cache entry; monitor retrieval quality as the index grows, and tune HNSW parameters (`M`, `efConstruction`, `efRuntime`) to dial recall vs. latency.

### 13.9 Embedding Model - Quality vs. Cost vs. Latency
All vector search quality depends on the embedding model. Larger models produce richer representations and higher retrieval quality but cost more per call and consume more storage. Smaller models are fast, cheap, and compact but miss subtle semantic similarities. Critically, the same model must be used at both ingestion and query time; switching requires re-embedding the entire corpus, which is a costly migration. Embedding model choice should be treated as a long-term infrastructure commitment. Matryoshka embeddings offer some escape - the same model produces variable-dimension vectors, so you can scale storage down without changing models.

### 13.10 Prompt Caching - Stability vs. Personalisation
Provider-side prompt caching offers up to 90% cost reduction and significant latency improvement on the cached prefix - but only when prefixes are stable. Structuring prompts for maximum cache hit (stable elements first, volatile elements last) sometimes conflicts with structuring prompts for maximum quality (recency-weighted, personalised). For most deployments, the cost savings justify the discipline; for highly personalised assistants, the structure can be relaxed selectively.

### 13.11 Build vs. Buy for Orchestration
LangGraph, LlamaIndex, Semantic Kernel, and DSPy provide pre-built abstractions for prompt management, RAG pipelines, memory, and tool use. They accelerate prototyping but their abstractions can become constraints at scale - debugging, performance tuning, and version pinning often become harder. Custom orchestration (FastAPI, Go) offers full control at the cost of building everything yourself. A common pattern is to start on a framework, then graduate to custom code once the system's specific requirements are clear and the abstractions start to cost more than they save.

### 13.12 MCP vs. Custom Tool Integration
MCP (Model Context Protocol) standardises how tools and resources are exposed to LLM-driven systems, removing the need to build a bespoke integration per model and per tool. The ecosystem is young but growing fast. Adopting MCP makes the Chat Service portable across models and clients but introduces a protocol dependency and currently lags some custom integrations on advanced features. For new builds in 2026, MCP is the default; for systems with deeply customised tool flows, custom integration may still win short term.
