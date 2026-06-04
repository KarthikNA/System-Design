# LLM-Powered Chatbot

## Problem Statement
Design a basic chatbot system that accepts user messages, sends them to a Large Language Model (LLM), and streams back coherent responses — supporting multi-turn conversations with context retention.

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
| **Chat Service** | Central orchestrator — coordinates all components to produce a response |
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

- **Web Browser** — a web app (React, Vue, etc.) that communicates with the backend over HTTP/WebSocket
- **Mobile App** — a native iOS or Android app, or a cross-platform app (React Native, Flutter) that integrates the chat UI
- **Embedded Chat Window** — a chat widget embedded inside another product (e.g. a customer support window, a docs site, or an IDE plugin)

---

## Data Flow

1. User sends a message from the client
2. API Gateway authenticates the request and forwards it to the Chat Service
3. Chat Service checks the Cache for an existing response to the query
4. If a cache miss, Guardrails validates the input against safety and policy rules
5. Chat Service queries RAG, which performs a semantic search against the Vector DB to fetch relevant context
6. Chat Service builds the final prompt (system prompt + retrieved context + conversation history + user message)
7. Model Router selects the appropriate LLM (self-hosted or external) using the Model Selection Model
8. The selected LLM generates a response, which is streamed back to the Chat Service
9. Guardrails validates the LLM output before it is returned
10. Chat Service streams the response back to the client via the API Gateway
11. Response is stored in Cache for future reuse

---

## Trade-offs & Considerations

- **RAG vs. fine-tuning** — RAG is easier to keep up to date but adds retrieval latency; fine-tuning bakes knowledge into the model but requires retraining cycles
- **Model routing** — routing to cheaper or self-hosted models reduces cost but may affect response quality for complex queries
- **Guardrails latency** — running safety checks on both input and output adds round-trip overhead; async or batched checks can mitigate this
- **Cache hit rate** — caching works well for repetitive queries but has low utility in open-ended conversational settings
- **Context window limit** — older messages must be truncated or summarized as conversations grow long
- **Streaming vs. REST** — WebSockets give a better UX for streaming but add connection management complexity
