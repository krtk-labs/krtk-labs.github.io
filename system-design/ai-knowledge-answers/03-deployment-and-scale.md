# Enterprise Conversational AI Deployment, Scaling and Operations

## 1. Deployment Goals

The deployment architecture must provide:

- Horizontal scalability
- Independent scaling of services
- GPU isolation
- High availability
- Multi-region resilience
- Low interactive latency
- Failure isolation
- Secure multi-tenancy
- Cost control
- Model/provider abstraction
- Observability

The key design principle is:

> **Keep the application/orchestration tier stateless and independently scalable, while treating model inference as a separate capacity pool.**

---

# 2. Reference Deployment Architecture

```text
                         INTERNET
                            |
                            v
                  Global Load Balancer
                            |
                            v
                       CDN / WAF
                            |
                            v
                      API Gateway
                            |
             ┌──────────────┼───────────────┐
             |              |               |
             v              v               v
      Conversation     AI Gateway       Auth Service
         Pods              Pods
             |              |
             └───────┬──────┘
                     |
                     v
              AI Orchestrator
                     |
        ┌────────────┼─────────────┐
        |            |             |
        v            v             v
   Retrieval      Tool Engine   Model Gateway
        |            |             |
        |            |       ┌─────┴─────┐
        |            |       |           |
        |            |       v           v
        |            |   Small LLM   Large LLM
        |            |
        |            v
        |       Enterprise APIs
        |
        v
 Search / Vector Infrastructure
```

---

# 3. Kubernetes Deployment

Stateless services can run in Kubernetes.

Example services:

```text
conversation-service
ai-orchestrator
retrieval-service
tool-service
model-gateway
intent-service
reranker-service
```

Conceptually:

```text
Kubernetes Cluster
        |
        +-- Conversation Pods
        |
        +-- AI Orchestrator Pods
        |
        +-- Retrieval Pods
        |
        +-- Tool Gateway Pods
        |
        +-- Model Gateway Pods
```

The application tier should be independently deployable.

---

# 4. Horizontal Autoscaling

Do not scale AI services only on CPU.

Useful signals include:

```text
Request rate
Active requests
Queue depth
P95 latency
CPU
Memory
GPU utilization
Tokens/sec
Model inference queue
```

Example:

```text
Normal:

AI Orchestrator = 10 pods

Traffic spike:

10 → 50 pods
```

GPU inference should scale independently.

---

# 5. GPU Inference Architecture

Do not place an LLM inside every application pod.

Instead:

```text
Application Pods
       |
       v
Inference Gateway
       |
       v
GPU Inference Pool
       |
   ┌───┼────┐
   |   |    |
   v   v    v
 GPU GPU  GPU
 Pod Pod  Pod
   |   |    |
 Model Model Model
```

Possible inference runtimes:

```text
vLLM
NVIDIA Triton
TensorRT-LLM
Equivalent optimized model-serving runtime
```

The exact runtime depends on model architecture and hardware.

---

# 6. Why Separate GPU Inference?

GPU capacity is expensive and behaves differently from normal application compute.

If the application and model server are coupled:

```text
Traffic spike
   |
   v
Application pods scale
   |
   v
GPU capacity scales with them
```

This can create:

- Poor utilization
- Expensive overprovisioning
- Slow scaling
- Operational complexity

Instead:

```text
Application scaling
       ≠
GPU scaling
```

This allows each tier to optimize independently.

---

# 7. GPU Scaling

Use:

```text
Request queue depth
+
GPU utilization
+
Tokens/sec
+
P95 inference latency
```

to determine scaling.

Example:

```text
Normal traffic
GPU Pods = 20

Traffic spike
GPU Pods = 20 → 100
```

For interactive workloads, maintain some warm capacity because GPU cold starts can be too slow.

A practical design is:

```text
Warm baseline
+
Autoscaled burst capacity
```

---

# 8. Model Gateway

The Model Gateway is the control point between the application and inference providers.

```text
                  AI Orchestrator
                         |
                         v
                   Model Gateway
                         |
       ┌─────────────────┼──────────────────┐
       |                 |                  |
       v                 v                  v
 Internal GPU       Provider A          Provider B
 Model Server
```

Responsibilities:

- Model routing
- Provider abstraction
- Authentication
- Rate limiting
- Token budgets
- Timeouts
- Retries
- Circuit breakers
- Fallback
- Cost accounting
- Observability
- Request tracing

This prevents application code from being coupled to one model provider.

---

# 9. Model Fallback

Suppose the primary model fails.

```text
Primary Model
     |
     X
     |
     v
Fallback Model
     |
     X
     |
     v
Simplified Workflow
     |
     X
     |
     v
Human Agent
```

Use timeouts and circuit breakers.

Do not blindly retry tool execution because some operations are not safe to repeat.

---

# 10. Event-Driven Architecture

Use an event bus for asynchronous work.

```text
                 Kafka / Pub/Sub
                       |
       ┌───────────────┼─────────────────┐
       |               |                 |
       v               v                 v
 Knowledge          Analytics          Audit
 Indexing           Pipeline           Pipeline
```

Example events:

```text
ConversationStarted
ConversationCompleted
TicketCreated
TicketUpdated
KnowledgeArticleUpdated
ToolExecuted
HumanEscalation
ResolutionAccepted
```

This keeps analytics and indexing out of the critical synchronous path.

---

# 11. Knowledge Indexing Deployment

```text
                Knowledge Sources
                       |
                       v
                 Event Bus
                       |
                       v
              Document Processors
                       |
                       v
                  Chunkers
                       |
                       v
               Embedding Workers
                       |
                       v
                 Search Index
```

These workers can scale independently.

For example:

```text
Normal:
Embedding workers = 10

Large documentation update:
10 → 100 workers
```

After processing completes, scale back down.

---

# 12. Data Architecture

## Operational datastore

Use a transactional datastore for:

```text
Tickets
Conversation metadata
Workflow state
Tenant configuration
Application configuration
```

Possible technologies:

```text
PostgreSQL
Distributed SQL
Equivalent managed relational database
```

## Cache

Use Redis or equivalent for:

```text
Conversation state
Session data
Rate limits
Frequently accessed configuration
Short-lived context
```

## Search

Use:

```text
OpenSearch
Elasticsearch
Equivalent managed search platform
```

for:

- Keyword search
- Hybrid retrieval
- Filtering
- Exact matching

## Vector index

Use:

```text
Vector-capable search
Dedicated vector database
```

for semantic retrieval.

## Object storage

Use:

```text
GCS / S3 / equivalent object storage
```

for:

- Documents
- Model artifacts
- Evaluation datasets
- Training datasets
- Large logs

---

# 13. Multi-Tenancy

Every request should carry:

```text
tenant_id
user_id
permissions
```

Retrieval must enforce tenant isolation.

Bad:

```text
Query
 |
 v
Vector Search
 |
 v
Top 10 Documents
```

Correct:

```text
Query
 |
 v
Vector Search
 +
 tenant_id filter
 +
 ACL filter
 +
 document status
 +
 user permissions
 |
 v
Authorized Documents
```

The model must never determine whether a user is authorized to access a document.

---

# 14. Security Architecture

```text
                 WAF
                  |
             API Gateway
                  |
               OAuth
                  |
          Identity / RBAC
                  |
          AI Orchestrator
                  |
        ┌─────────┴─────────┐
        |                   |
    Retrieval             Tools
        |                   |
       ACL                 AuthZ
        |                   |
        v                   v
 Knowledge Base       Enterprise APIs
```

Security controls:

```text
Encryption in transit
Encryption at rest
Tenant isolation
RBAC
ABAC where required
Secrets management
Audit logging
PII controls
Prompt-injection defenses
Tool authorization
Rate limiting
Network segmentation
```

---

# 15. Reliability

Assume every dependency can fail.

Potential failures:

```text
LLM provider unavailable
Vector database unavailable
Knowledge source unavailable
JSM API unavailable
Redis unavailable
Event bus unavailable
GPU capacity exhausted
```

Use:

```text
Timeouts
Retries with exponential backoff
Circuit breakers
Bulkheads
Dead-letter queues
Idempotency
Fallback models
Graceful degradation
```

Example:

```text
RAG unavailable
     |
     v
Use deterministic workflow / known intent
     |
     v
If insufficient confidence
     |
     v
Human escalation
```

The platform should fail safely rather than invent an answer.

---

# 16. Latency Architecture

A conversational system should optimize the critical path.

Illustrative budget:

```text
API Gateway             ~20 ms
Conversation lookup     ~10 ms
Intent model            ~30 ms
Retrieval               ~50 ms
Reranking               ~50 ms
LLM generation          ~500-1500 ms
-------------------------------------
Total                   ~700-1700 ms
```

These are architectural targets, not guaranteed values.

Use streaming:

```text
LLM
 |
 | token
 | token
 | token
 v
Customer
```

instead of waiting for the entire response.

---

# 17. Observability

## Infrastructure metrics

```text
CPU
Memory
GPU utilization
Network
Pod health
Node health
```

## Application metrics

```text
Request rate
Error rate
P50/P95/P99 latency
Queue depth
Tool failures
Timeouts
```

## AI-specific metrics

```text
Intent accuracy
Retrieval precision
Recall@K
Reranker quality
Groundedness
Hallucination rate
Tool selection accuracy
Tool success rate
Escalation rate
Autonomous resolution rate
Token consumption
Cost/request
```

---

# 18. Distributed Tracing

A single conversation should have a trace ID.

```text
Trace ID
   |
   +-- API Gateway
   |
   +-- Conversation Service
   |
   +-- Intent Model
   |
   +-- Retrieval
   |
   +-- Reranker
   |
   +-- Model Gateway
   |
   +-- LLM
   |
   +-- Tool Gateway
   |
   +-- Jira API
```

This lets engineers answer:

> Why did this request take 4 seconds?

or:

> Why was this request escalated?

---

# 19. Cost Control

At large scale, model cost becomes an architectural concern.

Controls include:

### Model routing

Use small models where possible.

### Context reduction

Don't send unnecessary documents or history.

### Response limits

Set token budgets.

### Caching

Cache safe, reusable results.

### Semantic caching

Potentially reuse responses for sufficiently similar low-risk questions.

### Retrieval optimization

Don't send 50 documents to the LLM.

### Batch processing

Use asynchronous batching for offline embedding/indexing.

---

# 20. Multi-Region Deployment

For large SaaS scale:

```text
                   Global Load Balancer
                           |
             ┌─────────────┴─────────────┐
             |                           |
             v                           v
         Region A                    Region B
             |                           |
        Kubernetes                  Kubernetes
             |                           |
       AI Application              AI Application
             |                           |
       GPU Inference               GPU Inference
```

Stateless services can be active-active.

Data requires more careful design.

Use:

```text
Replication
Backups
Point-in-time recovery
Regional failover
Disaster recovery procedures
```

Avoid unnecessary cross-region synchronous calls on the interactive path.

---

# 21. Deployment Strategy

Use progressive delivery.

## Step 1 — Offline evaluation

```text
Candidate Model
      |
      v
Evaluation Dataset
      |
      v
Quality Gates
```

## Step 2 — Shadow

```text
Production Request
       |
       +----> Current Model → Real response
       |
       +----> New Model → Evaluation only
```

## Step 3 — Canary

```text
95% → Current
5%  → New
```

## Step 4 — Progressive rollout

```text
5%
 ↓
25%
 ↓
50%
 ↓
100%
```

Rollback if:

```text
Resolution rate decreases
Latency increases
Hallucinations increase
Tool failures increase
Cost increases unexpectedly
```

---

# 22. Google-Scale Interpretation

"Google scale" should not be interpreted as one giant server.

It means designing around:

```text
Horizontal scaling
Stateless services
Partitioning
Asynchronous processing
Caching
Independent capacity pools
Multi-region deployment
Backpressure
Failure isolation
Observability
```

A conceptual architecture:

```text
                     Global Traffic
                           |
                           v
                  Global Load Balancer
                           |
             ┌─────────────┼─────────────┐
             |             |             |
             v             v             v
          Region A      Region B      Region C
             |             |             |
          K8s/GKE       K8s/GKE       K8s/GKE
             |             |             |
          Services       Services       Services
             |             |             |
        Model Gateway Model Gateway Model Gateway
             |             |             |
         GPU Pool      GPU Pool      GPU Pool
```

Not every component needs to be globally replicated.

For example:

```text
Stateless API tier → globally distributed
Inference tier     → regionally distributed
Search             → regionally distributed/replicated
Operational data   → carefully replicated
Analytics          → asynchronous/global
```

---

# 23. Scaling Dimensions

Different components scale differently.

| Component | Scaling signal | Scaling strategy |
|---|---|---|
| API Gateway | Requests/sec | Horizontal |
| Conversation Service | Requests/sec | Horizontal |
| AI Orchestrator | Active requests | Horizontal |
| Intent Model | Inference QPS | Horizontal |
| Retrieval | Query QPS | Horizontal |
| Reranker | Query QPS / latency | Horizontal |
| LLM | Tokens/sec / queue depth | GPU autoscaling |
| Tool Gateway | Tool calls/sec | Horizontal |
| Knowledge Indexer | Queue depth | Worker autoscaling |
| Analytics | Event backlog | Consumer scaling |

This independent scaling is one of the strongest architectural properties of the design.

---

# 24. Failure Scenarios

## LLM unavailable

```text
Primary LLM
    |
    X
    |
Fallback model
    |
    v
Response
```

## Retrieval unavailable

```text
RAG
 |
 X
 |
Known deterministic workflow
 |
 v
If confidence insufficient
 |
 v
Human
```

## Tool API unavailable

```text
LLM
 |
 v
Tool Gateway
 |
 X
 |
 v
Do not claim success
 |
 v
Explain / retry / escalate
```

## Event bus unavailable

Synchronous conversation should not necessarily fail.

Instead:

```text
Conversation
   |
   +--> Critical synchronous path
   |
   +--> Async event
             |
             X
             |
          Retry / DLQ
```

---

# 25. Architecture Decision Records

## ADR-001: Separate Model Gateway

**Decision:** All LLM calls go through a Model Gateway.

**Reason:**

- Provider abstraction
- Routing
- Cost control
- Fallback
- Observability
- Rate limiting

---

## ADR-002: Separate Tool Gateway

**Decision:** LLMs cannot directly call enterprise APIs.

**Reason:**

- Authorization
- Security
- Audit
- Schema validation
- Idempotency
- Policy enforcement

---

## ADR-003: Separate GPU Inference

**Decision:** GPU model serving is independently scalable from application services.

**Reason:**

- Expensive hardware
- Different scaling characteristics
- Better utilization
- Independent model deployment

---

## ADR-004: Hybrid Retrieval

**Decision:** Combine semantic and keyword retrieval.

**Reason:**

- Better exact-match behavior
- Better semantic matching
- Better technical terminology handling

---

## ADR-005: Event-Driven Indexing

**Decision:** Knowledge indexing is asynchronous.

**Reason:**

- Keeps indexing out of request path
- Independent scaling
- Retryability
- Failure isolation

---

# 26. Principal Engineer Discussion Points

If asked why the architecture is structured this way, the strongest answers are:

### Why not one LLM?

Because different tasks have different latency, cost, and reasoning requirements.

### Why have an intent model?

To cheaply identify known workflows before invoking expensive reasoning.

### Why RAG?

Because enterprise knowledge changes more frequently than foundation models and needs access control and source grounding.

### Why reranking?

Vector retrieval is good at candidate generation but ranking quality can be improved with a cross-encoder/reranker.

### Why a Model Gateway?

To abstract providers/models and centralize routing, fallback, rate limiting, observability, and cost controls.

### Why a Tool Gateway?

The LLM should never own authorization.

### Why separate GPU infrastructure?

LLM inference has fundamentally different compute and scaling characteristics from stateless application services.

### Why event-driven indexing?

Document processing is asynchronous and should not affect interactive request latency.

### Why human escalation?

Autonomy should be confidence- and risk-aware, not binary.

### How did you improve autonomous resolution?

Improve the entire decision path:

```text
Intent
+
Context
+
Retrieval
+
Reasoning
+
Tool execution
+
Validation
+
Escalation
```

---

# 27. Suggested Interview Narrative

If asked:

> "What did you mean by ML inference pipelines?"

A concise answer:

> "The conversational flow wasn't a single LLM call. The inference path combined intent classification, entity extraction, context construction, retrieval and ranking of knowledge, model inference, tool selection, and response validation. We separated those stages so inexpensive models handled classification and retrieval tasks while larger models were reserved for complex reasoning. The orchestration layer also provided model routing, fallbacks, observability, and policy enforcement."

If asked about scale:

> "The key scaling decision was to keep the application and orchestration tiers stateless and independently autoscalable, while treating model inference as a separate capacity pool. We used an inference gateway between the orchestration layer and model servers so GPU-heavy workloads could scale independently. Retrieval and indexing were also separated, with asynchronous document ingestion and independently scalable online retrieval."

If asked how autonomous resolution improved:

> "The improvement wasn't simply from using a better LLM. We improved the complete decision path: intent recognition, conversation state, retrieval, controlled tool execution, model routing, and escalation. That allowed us to safely automate more classes of requests while routing low-confidence or high-risk requests to human agents."

---

# 28. Resume Wording Guidance

Use **"ML inference pipelines"** only if you genuinely owned model-serving or ML inference infrastructure.

If your actual work was primarily backend orchestration around foundation models, a safer and stronger phrase is:

> **"AI orchestration, retrieval and inference pipelines"**

For example:

> Designed and developed key conversational AI flows within Jira Service Management, driving autonomous ticket resolution from 35% to 80% by redesigning backend services, AI orchestration, retrieval and inference pipelines, and frontend integrations.

This communicates deep technical ownership without implying that you personally trained or operated proprietary ML models.

---

# 29. Final Architecture

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

---

# 30. The Architectural Mental Model

The simplest way to remember the design is:

```text
UNDERSTAND
    ↓
Intent + Entities
    ↓
REMEMBER
    ↓
Conversation State
    ↓
KNOW
    ↓
RAG + Retrieval + Reranking
    ↓
REASON
    ↓
Model Gateway + LLM
    ↓
ACT
    ↓
Tool Gateway + Enterprise APIs
    ↓
VERIFY
    ↓
Validation + Policy + Confidence
    ↓
RESOLVE OR ESCALATE
```

That is the core architecture you should be able to explain on a whiteboard.
