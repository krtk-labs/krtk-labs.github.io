# Enterprise Conversational AI Architecture

> Reference architecture for an enterprise conversational AI and autonomous support platform.
>
> **Important:** This is a reference architecture for the type of system represented by the resume statement. It is not a claim about Atlassian's proprietary internal implementation. Exact model names and infrastructure should only be stated as factual if they were actually used.

## 1. Objective

The platform should:

1. Understand support requests.
2. Maintain conversation context.
3. Classify requests into known workflows.
4. Answer knowledge-based questions using RAG.
5. Execute deterministic support workflows.
6. Call enterprise systems through controlled tools.
7. Decide when autonomous resolution is safe.
8. Escalate ambiguous or high-risk requests to humans.
9. Handle very high request volumes and concurrent conversations.
10. Maintain predictable latency, reliability, security, and cost.

The key architectural principle is:

> **Separate AI reasoning from business execution and authorization.**

The LLM should reason and select from a constrained set of tools. It should never have unrestricted access to enterprise APIs.

---

## 2. High-Level Architecture

```text
                         ┌─────────────────────────┐
                         │        Customer         │
                         │ Portal / Slack / Teams  │
                         └────────────┬────────────┘
                                      │
                                      ▼
                         ┌─────────────────────────┐
                         │ CDN / Edge / WAF        │
                         └────────────┬────────────┘
                                      │
                                      ▼
                         ┌─────────────────────────┐
                         │ API Gateway             │
                         │ Auth / Rate Limiting    │
                         └────────────┬────────────┘
                                      │
                                      ▼
                    ┌──────────────────────────────────┐
                    │ Conversation Service             │
                    │ Session / State / History        │
                    └────────────────┬─────────────────┘
                                     │
                                     ▼
                    ┌──────────────────────────────────┐
                    │       AI Orchestrator             │
                    │                                  │
                    │ Intent → Context → Retrieval     │
                    │ → Reasoning → Tools → Response  │
                    └───────────────┬──────────────────┘
                                    │
             ┌──────────────────────┼────────────────────────┐
             │                      │                        │
             ▼                      ▼                        ▼
      ┌─────────────┐       ┌──────────────┐       ┌──────────────┐
      │ ML / AI     │       │ RAG Pipeline │       │ Tool Engine  │
      │ Inference   │       │              │       │              │
      └──────┬──────┘       └──────┬───────┘       └──────┬───────┘
             │                     │                      │
       ┌─────┴─────┐         ┌─────┴──────┐        ┌────┴────────┐
       │            │         │            │        │             │
       ▼            ▼         ▼            ▼        ▼             ▼
   Classifier   Embedding  Vector DB    Reranker  JSM APIs    External APIs
   Model        Model
       │
       ▼
  Model Gateway
       │
       ├──────────────┐
       ▼              ▼
   Small LLM       Large LLM
       │              │
       └──────┬───────┘
              ▼
       Response Validator
              │
              ▼
       Human / Customer
```

---

## 3. Why an AI/ML Inference Pipeline?

A simplistic architecture is:

```text
User → LLM → Answer
```

That is insufficient for an enterprise support platform.

A production inference path is closer to:

```text
User Request
     |
     v
Intent Classification
     |
     v
Entity Extraction
     |
     v
Conversation Context
     |
     v
Knowledge Retrieval
     |
     v
Reranking
     |
     v
Context / Prompt Construction
     |
     v
Model Routing
     |
     v
LLM Inference
     |
     v
Tool Execution
     |
     v
Response Validation
     |
     v
Autonomous Resolution / Human Escalation
```

Each stage solves a different problem.

The architecture deliberately avoids using an expensive large model for every operation.

---

## 4. Model Strategy

| Task | Candidate model type | Why |
|---|---|---|
| Intent classification | DistilBERT / MiniLM / small transformer | Fast, inexpensive, predictable |
| Entity extraction | Small transformer or structured-output LLM | Structured information |
| Embeddings | BGE / E5 / enterprise embedding model | Semantic retrieval |
| Reranking | BGE Reranker / cross-encoder | Better relevance |
| Simple FAQ | Small LLM | Lower cost and latency |
| Complex reasoning | Large LLM | Better reasoning |
| Safety classification | Small classifier / guardrail model | Fast policy gate |
| Response quality | Classifier or LLM evaluator | Detect poor/unsupported responses |

Exact model selection depends on:

- Data residency
- Licensing
- Quality
- Latency
- Cost
- Hardware availability
- Existing model platform
- Security requirements
- Vendor constraints

The application should therefore hide models behind a **Model Gateway**.

---

## 5. End-to-End Request Flow

Example:

> "I can't connect to VPN after changing my laptop."

```text
User
 |
 v
API Gateway
 |
 v
Conversation Service
 |
 v
AI Orchestrator
 |
 +--> Intent Classification
 |
 +--> Entity Extraction
 |
 +--> Conversation Context
 |
 +--> Knowledge Retrieval
 |
 +--> Reranking
 |
 +--> Model Routing
 |
 +--> LLM
 |
 +--> Tool Execution
 |
 +--> Response Validation
 |
 v
Response
```

---

## 6. Intent Classification

The system first determines the request category.

Potential intents:

```text
VPN problem
Password reset
Software access
Hardware request
Network problem
Account problem
General knowledge
Unknown
```

Example model:

```text
DistilBERT / MiniLM
```

Pipeline:

```text
User Message
     |
     v
Tokenizer
     |
     v
Transformer
     |
     v
Classification Head
     |
     v
Intent + Confidence
```

Example:

```json
{
  "intent": "VPN_CONNECTIVITY",
  "confidence": 0.96
}
```

### Why a small model?

Intent classification normally does not require complex reasoning.

A small model provides:

- Lower latency
- Lower cost
- High throughput
- Predictable behavior

A large LLM is reserved for tasks requiring reasoning.

---

## 7. Entity Extraction

Example:

```text
"I can't connect to GlobalProtect on my new Windows laptop."
```

Output:

```json
{
  "product": "GlobalProtect",
  "device": "Laptop",
  "operating_system": "Windows"
}
```

Possible implementations:

### Option A — NER model

A fine-tuned Named Entity Recognition model.

### Option B — Structured-output LLM

```text
Small LLM
   |
   v
JSON schema
   |
   v
Schema validation
```

Structured extraction is useful when support workflows evolve rapidly.

---

## 8. Conversation State

The AI must understand the complete conversation.

Example:

```text
User:
VPN doesn't work.

AI:
What operating system are you using?

User:
Windows.

AI:
Did this start after changing computers?

User:
Yes.
```

Maintain:

```text
Conversation ID
User ID
Tenant ID
Current intent
Extracted entities
Previous answers
Current workflow step
Ticket ID
Relevant context
Authentication state
Authorization state
```

Use Redis or an equivalent low-latency store for transient state.

Durable ticket/business data remains in the primary datastore.

---

## 9. Autonomous Resolution

Not every request should be autonomous.

The platform should make an explicit autonomy decision:

```text
                     Request
                        |
                        v
                Risk / Confidence
                        |
          ┌─────────────┴──────────────┐
          │                            │
       High confidence             Low confidence
       + low risk                       |
          |                             v
          v                          Human
      Autonomous                   escalation
       resolution
```

Examples:

### Low risk

```text
"What is the VPN setup procedure?"
```

→ Answer automatically.

### Medium risk

```text
"Why isn't my VPN working?"
```

→ Diagnose and guide.

### Controlled action

```text
"Reset my VPN credentials."
```

→ Verify identity → authorization → execute tool.

### High risk

```text
"Give me admin access to production."
```

→ Human approval.

This policy boundary is essential to safely increase autonomous resolution.

---

## 10. Measuring the 35% → 80% Improvement

Define the metric explicitly:

```text
Autonomous Resolution Rate
=
Requests successfully resolved
without human intervention
--------------------------------
Eligible requests
```

Break the improvement into contributing factors:

```text
Better intent recognition
+
Better retrieval
+
Better conversation state
+
Better tool execution
+
Better model routing
+
Better escalation
```

The architectural story should therefore be:

> The improvement came from optimizing the complete decision and execution path, not merely replacing one LLM with another.

---

## 11. Reference Architecture Summary

```text
                           USERS
                             |
            Portal / Slack / Teams / Email
                             |
                             v
                    CDN / Global LB / WAF
                             |
                             v
                       API Gateway
                             |
                    Auth / Rate Limit
                             |
                             v
                 ┌──────────────────────┐
                 │ Conversation Service │
                 └──────────┬───────────┘
                            |
                            v
                 ┌──────────────────────┐
                 │   AI ORCHESTRATOR    │
                 └──────────┬───────────┘
                            |
             ┌──────────────┼─────────────────┐
             |              |                 |
             v              v                 v
        Intent Model     Context          Policy Engine
        MiniLM/etc.       Store               |
             |              |                 |
             └──────────────┼─────────────────┘
                            |
                            v
                    Retrieval Pipeline
                            |
              ┌─────────────┴─────────────┐
              |                           |
          Embedding                    Search
           Model                      Index
              |                           |
              └─────────────┬─────────────┘
                            |
                         Top 50
                            |
                            v
                       Reranker
                            |
                          Top 5
                            |
                            v
                    Context Builder
                            |
                            v
                      Model Gateway
                            |
              ┌─────────────┴──────────────┐
              |                            |
         Small LLM                    Large LLM
              |                            |
              └─────────────┬──────────────┘
                            |
                            v
                       Tool Router
                            |
                  ┌─────────┼─────────┐
                  |         |         |
                  v         v         v
                Jira      IAM       CMDB
                 API       API       API
                  |         |         |
                  └─────────┼─────────┘
                            |
                            v
                     Response Validator
                            |
                   ┌────────┴─────────┐
                   |                  |
                Safe               Unsafe
                   |                  |
                   v                  v
               Customer            Human
                                    Agent
```
