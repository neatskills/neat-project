# Integration Architecture Design

**Include when:** significant cross-boundary connections exist — AI↔app, external APIs, SaaS, events.

**Inputs:** Agent, app, and data architecture outputs (as generated). Run after those documents are complete.

## Pattern Identification

**AI ↔ Traditional (include if agent is present):**

| Pattern | Description | Key design concerns |
| --- | --- | --- |
| **Chatbot/UI → Agent** | Conversational interface calls the agent orchestrator | Streaming (SSE/WebSocket), session state, latency |
| **Agent → Tool** | Agent calls traditional APIs or functions as tools | Idempotency, timeout, schema contract, auth |
| **Traditional event → Agent** | A workflow or system event triggers an agent | Event schema, async processing, retry on failure |
| **Agent → Traditional storage** | Agent writes results back to a database or system of record | Data shape, conflict resolution, audit trail |
| **Agent → Agent** | One agent run dispatches and communicates with another | Context/permission scoping, sync-vs-async routing, concurrent shared-resource access |

**External (include if connecting to external services):**

| Pattern | Description | Key design concerns |
| --- | --- | --- |
| **Outbound REST/GraphQL** | System calls an external API (payment gateway, government API, SaaS) | Auth (API key / OAuth), rate limiting, retry, schema versioning |
| **Inbound webhook** | External service pushes events to the system | Signature validation, idempotency key, async processing, replay |
| **SaaS data sync** | Bi-directional or one-way sync with a SaaS platform | Conflict resolution, field mapping, pagination, delta sync |
| **Event bus / message queue** | Async communication via shared bus (Kafka, SQS, EventBridge) | Topic/queue design, consumer group, dead letter queue, ordering |
| **File / batch transfer** | Scheduled bulk data exchange (SFTP, S3 drop, ETL trigger) | File format contract, schedule coordination, failure detection, idempotency |

## Integration Design

List each cross-boundary point (typically 3–8 for a focused MVP). For each, define:

- **Trigger:** user action, system event, scheduled job, or state change
- **Sync or async:** async for long-running AI ops; sync for fast on-demand responses
- **Data flow:** what goes in, comes out, gets stored and where
- **Streaming:** SSE, WebSocket, or chunked transfer for progressive AI responses
- **Error handling:** idempotent retries with exponential backoff; fallback when AI unavailable
- **Permission context:** how user identity and access scope passes to AI component

## Conventions

| Concern | Key decisions |
| --- | --- |
| **API Contract** | One endpoint per integration point; sync vs async; request/response shape |
| **API Versioning** | Strategy (URL path `/v1/`, header, content negotiation); how breaking changes are communicated |
| **Error Handling** | Idempotent retries with exponential backoff; dead letter queues for async; fallback so users can proceed when AI is down |
| **Deployment** | AI as separate service or co-located (decide based on scale and team); health checks and circuit breakers required |

## Performance & Scalability

- **Throughput design:** for high-volume integration points, decide queue depth, back-pressure strategy, and whether fan-out is bounded
- **Async vs sync:** async for long-running AI ops; sync only when latency requirement demands it and the upstream can handle the wait
- **Caching:** cache repeated AI calls where inputs are stable; state cache invalidation strategy
- **Rate limiting:** rate-limit outbound LLM calls to stay within provider limits; rate-limit inbound webhooks to protect the system

**Defer to planning:** queue technology · specific rate limit thresholds · cache TTL values

## Observability

- **Distributed tracing:** propagate a single correlation ID across all service boundaries; each integration point logs the ID on entry and exit
- **Integration health:** track latency and error rate per integration point; flag points where degradation cascades
- **Audit trail:** log what crossed each system boundary, when, and under which identity — required when data leaving the system has compliance implications

**Defer to planning:** tracing tooling · alerting thresholds (owned by infra)

## Security & Compliance

- **Inter-service auth:** API key, OAuth, or mTLS per integration point — state the mechanism and trust model
- **PII handling:** redact or mask PII before sending to external LLM providers; state what leaves the system boundary
- **Compliance:** audit trail for data crossing system boundaries; confirm data residency requirements are met for each external call; flag regulated data types (health, financial, personal)

**Defer to planning:** auth library · certificate management (owned by infra)
