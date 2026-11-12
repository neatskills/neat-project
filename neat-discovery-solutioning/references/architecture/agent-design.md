# Agent Architecture Design

**Include when:** non-deterministic or AI requirements exist.

**Inputs:** Requirements and high-risk stories (XL, XXL) from prd.md's Delivery Estimate; Current State (if available).

## Requirements Analysis

Scan requirements for: distinct domains/capabilities, shared concepts, knowledge sources, orchestration needs. Assess knowledge fragmentation (source count, synthesis required, real-time needs) and integration complexity (how AI connects to traditional systems). Summarize before proceeding.

## Runtime Selection

**Include when:** the project builds its own agent runtime/harness.

**Decision: what does the agent orchestration run on?**

| Category | Choose when | Trade-off |
| --- | --- | --- |
| **Minimal/generic tool-calling harness** | Flat LLM↔tool loop; must preserve fidelity to an existing execution model | Less built-in orchestration; more must be built |
| **Graph/state-machine framework** | Complex branching is central; benefits from a large ecosystem | Execution model may not map to a flat loop or existing model |
| **Managed cloud agent platform** | Team wants hosted orchestration and accepts vendor constraints | Vendor lock-in; less control over execution model |
| **Multi-agent framework** | Multi-agent collaboration (debate, negotiation, role-based teams) is core | Adds framework mental model on top of LLM provider primitives |
| **Fully custom** | Requirements are narrow enough that a custom loop is simpler | Full build cost; no ecosystem to lean on |

Evaluate on: execution model fit, out-of-the-box vs build coverage, and fidelity to any existing execution model the project must remain compatible with. Distinct from LLM Provider & Cost Design — it's about what orchestrates the agent, not which model it calls.

## Knowledge Design

| | YES when | NO when |
| --- | --- | --- |
| **Ontology** | Multiple agents share concept definitions; relationship reasoning required; cross-domain semantic alignment needed | Simple stable domain; sources already aligned |
| **Knowledge system** | 10+ fragmented sources; real-time synthesis required; sources lack APIs | Current sources sufficient; direct agent-to-source access viable |

For each YES: cite specific requirements as rationale, state the trade-off, and specify the design (ontology: concepts/relationships; knowledge system: storage/access/update).

Self-check: Ontology YES — can you name which agents share definitions? Knowledge system YES — can you cite specific sources from Current State that lack APIs or require synthesis? If not, revert to NO.

## Agent Design

Map requirements to agents, prioritizing XL/XXL stories.

**For each agent:** name and purpose (mapped to requirement group), knowledge sources, reasoning capabilities, input/output interfaces. Reference estimation assumptions — if estimate says "user validation workflow," agent design must enable it.

**Orchestration patterns:**

- Sequential: Agent B needs Agent A's output
- Parallel: independent agents work simultaneously
- Hierarchical: coordinator manages sub-agents
- Background (fire-and-forget): parent dispatches a sub-run it doesn't wait on; reconciles result on its next relevant turn

**Knowledge access:** direct source vs knowledge system, caching needs, real-time vs batch. If estimate says "async processing required," architecture must support it.

**Integration points with traditional systems:** enumerate systems and data format at each boundary. Auth, error handling, retry, and fallback are owned by integration — cross-reference the AI↔Traditional patterns table there.

## Knowledge Flow

**Flow pattern: sources → [ontology] → [knowledge system] → agents → users**

For each step: what transformations occur, trigger, latency requirements, error handling.

**AI unavailability:** define the fallback when the LLM provider is unreachable (graceful degradation, cached last response, or hard failure). Flag for integration to design the fallback handler.

**Knowledge updates:** how source changes propagate (real-time vs periodic sync), conflict resolution strategy.

## LLM Provider

**Provider selection:** evaluate context window against document volume requirements, structured output support, p95 latency, cost per token, and data residency. State provider category (hosted API / cloud-managed / self-hosted) with rationale. Defer specific model version to planning.

**Model routing (if applicable):** fast/cheap model for classification and routing; capable model for reasoning and synthesis. State the routing trigger (complexity signal, task type, cost threshold).

**Prompt caching:** identify stable prompt segments; state whether provider-level caching applies; estimate cache hit rate.

**Cost model:** estimate tokens per request, daily call volume, and produce an order-of-magnitude monthly cost estimate (feasibility signal, not a budget). Flag if estimate is a risk.

**Defer to planning:** specific model version · fine-tuning · prompt engineering · token optimisation

## Performance & Scalability

**Include when:** concurrent users, high request volume, or real-time latency requirements exist.

- **Processing model:** sync (direct LLM call per request) vs async (queue-backed); async required when load exceeds LLM rate limits or calls exceed user wait tolerance
- **Statelessness:** agents must be stateless to scale horizontally — state the context/session management strategy; stateful agents are a scaling ceiling
- **Rate limit strategy:** decide how to handle LLM provider limits — request queuing, token budgeting, or priority routing
- **Circuit breaking:** define fallback when LLM calls are slow or failing — required when agent is on a critical path
- **Cost ceiling:** high traffic scales token cost linearly — decide throttling or tiered access at architecture level

**Defer to planning:** queue technology · worker counts · timeout values · rate limit thresholds

## Observability

- **Agent run tracing:** define what each agent run must emit — inputs, tool calls, outputs, and decision points; attach to the user request correlation ID
- **Evaluation design:** define success and failure signals per agent (task completion rate, fallback rate, user correction rate); state whether LLM-as-judge applies
- **Cost attribution:** assign token cost per feature or per user to enable cost accountability

Tooling, log aggregation, and alerting are owned by infra.

## Security & Compliance

**Prompt injection:** identify inputs reaching the LLM prompt (user text, retrieved docs, tool outputs); state trust level per source (user-controlled = untrusted; internal DB = semi-trusted); state mitigation (input sanitisation, prompt structure separating instructions from data, output validation).

**Compliance classification:** do AI outputs constitute advice (legal, medical, financial)? Do they include user data? State obligations (disclaimer, human review, right-of-access, deletion). Confirm data residency with LLM provider.

**Defer to planning:** sanitisation libraries · output filtering · audit log schema

API key management and secrets rotation are owned by infra.
