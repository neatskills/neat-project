# Solution Flows

Document all flows that apply from these categories:

## 1. Core Business Flows

**Always required** — system implementation flows for Active-scope requirements

Document how the system implements each requirement (technical perspective):

- Component interactions (API → service → database → worker)
- Data flow between system parts
- Happy path + error branches
- State transitions (if entities have workflow states)

**Not:** User journey/experience flows (that's UX design, not system implementation).

**Anti-pattern:** "Similar to Flow X" or "omitted for brevity" — if it's Active scope, create the diagram.

## 2. State Machine Flows

**Required when:** Domain entities have status/workflow fields

Document:

- All states and valid transitions (use mermaid stateDiagram)
- Transition guards/conditions
- Invalid transition handling

**Skip when:** Only CRUD or binary states (active/inactive).

## 3. Integration Flows

**Required when:** System calls external APIs or services

Document:

- Request/response sequence with external system
- Error handling (timeouts, 4xx, 5xx, network failures)
- Retry logic, circuit breakers, fallbacks

**Anti-pattern:** Describing retry logic in prose instead of showing it in a sequence diagram.

## 4. Asynchronous & Background Flows

**Required when:** Background workers, queues, or scheduled jobs exist

Document:

- Worker trigger mechanism (queue, cron, event)
- Processing logic and completion/failure handling
- Queue processing (polling, batching, DLQ routing)

## 5. Authentication & Authorization

**Required when:** Users or services need to authenticate

Document:

- User authentication (login, token exchange, session creation)
- Token refresh and expiration handling
- Service-to-service authentication
- Authorization checks (roles, permissions)

**Skip when:** Public read-only systems, internal scripts, libraries, CLI tools with no auth
