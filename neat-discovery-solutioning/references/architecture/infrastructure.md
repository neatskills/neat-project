# Infrastructure & Platform Architecture Design

**Include when:** any deployment concern exists, or the project is built within a managed platform ecosystem.

## Path Selection

| Path | When to use |
| --- | --- |
| **A — Managed Platform** | Project is built within a platform that imposes its own development model (Salesforce, ServiceNow, Dynamics 365, Databricks, Snowflake, SAP) |
| **B — Cloud / PaaS / Local** | Cloud provider (AWS, GCP, Azure) or PaaS chosen as infrastructure without imposing a development model |

---

## Path A: Managed Platform Ecosystem

**Generate this section first.** Platform constraints govern what is possible in app, data, and integration architecture.

**Inputs:** Platform identified in prd.md; build/buy decisions and constraints from Delivery Estimate.

### Platform Identification & Constraints

| Tier | Description | Examples |
| --- | --- | --- |
| **CRM/ERP platform** | Object model, workflows, declarative configuration, governor limits | Salesforce, ServiceNow, Dynamics 365 |
| **Data platform** | Compute model, storage format, workspace/catalog architecture | Databricks, Snowflake, dbt Cloud |
| **ERP/finance platform** | Process model, module dependencies, customisation boundaries | SAP, Oracle EBS |

Document the platform constraints: governor limits, extension model (declarative vs programmatic), data model constraints (locked objects/schemas), authentication model, and API surface available for external integration.

### Architecture Within the Platform

**Data model:** which platform objects are used as-is vs extended vs replaced; naming conventions; relationship design within platform constraints; identify data that must live outside the platform.

**Extension approach:** declarative first (flows, formula fields, validation rules); state the programmatic boundary (what requires code and why); integration approach within platform limits.

**Compute and processing model (data platforms):** workspace/catalog/schema structure; job vs interactive compute; cluster/warehouse sizing strategy — defer sizes to planning; data sharing approach.

**AI and agent integration (if agent is present):** platform-native AI vs external LLM — state the trade-off (data residency vs model flexibility). If native: document model, invocation API, and governor limits. If external: document how calls leave the platform and any outbound restrictions.

**Defer to planning:** specific field definitions · object-level security · declarative rule implementation

### Release Pipeline

**Environments:** state the environment structure (e.g., Salesforce sandboxes: Developer → Partial Copy → Full; Databricks: dev → prod workspace).

**Deployment mechanism:** Salesforce (Change Sets vs SFDX vs unlocked packages); ServiceNow (update sets vs application scoping); Databricks/Snowflake (dbt, Terraform, or platform-native pipelines).

**Release governance:** platform-specific approval gates, change freeze windows, or deployment windows. (Serves as this path's Rollout content for Operational Concerns.)

**Defer to planning:** deployment scripts · package version selection · environment provisioning

### Cross-Cutting Constraints for Other Documents

- **app:** decisions already made by the platform (e.g., frontend mandated, all logic in Apex)
- **data:** storage decisions already made (e.g., all transactional data in platform objects)
- **integration:** platform API limits and available integration patterns

---

## Path B: Cloud / PaaS / Local

**Generate this section after** data architecture and app architecture are complete.

**Inputs:** Data architecture output (storage type); app architecture output (compute needs).

Skip to Local-first if the project is a CLI, local utility, or embeddable library.

### Hosting Model

| Criterion | PaaS | Cloud-native |
| --- | --- | --- |
| Operational overhead | Near-zero | Significant |
| Cost | Lower at small scale | Higher baseline, better at scale |
| Managed services | Limited | Rich |
| DevOps required | None | Cloud expertise |
| Vendor lock-in | Low | High |

**Decision inputs:** concurrent users, DevOps capability, budget, storage type, existing cloud contracts.

### Design

Pick one path and design only that:

- **PaaS:** platform, compute (container), storage (volume), CI/CD, env vars, health check, scalability ceiling.
- **Cloud-native:** provider, compute (Lambda / ECS / GKE), storage (S3 / EFS / GCS), networking (VPC, CDN, DNS), managed services, CI/CD.
- **Local-first:** packaging/distribution (npm / binary / git install), CI/CD for tests and releases, local secrets handling, scalability ceiling.

State decision, rationale, and resulting design.

### Deployment Diagram

Generate a `graph LR` Mermaid diagram immediately after the Container Diagram in `solution.md ## Architecture`, under the heading `### Deployment Diagram`.

**What to include:**

- Every named managed service chosen in the Design section above (e.g. `AWS Lambda`, `Amazon RDS`, `Azure Service Bus`, `Cloud Run`, `Cloud SQL`)
- Directional arrows labeled with the protocol or mechanism (`HTTPS`, `SQS poll`, `JDBC`, `S3 SDK`, `Pub/Sub push`, etc.)
- External systems that the Container Diagram already shows as out-of-scope actors — represent as a single node on the edge of the diagram; do not decompose their internals

**How to group:**

Use Mermaid subgraphs to group services by layer:

| Subgraph | Contains |
| --- | --- |
| `Networking / Edge` | CDN, API gateway, load balancer, DNS |
| `Compute` | Functions, containers, app servers, workers |
| `Data` | Databases, object storage, caches, search |
| `Messaging` | Queues, topics, event buses, streams |
| `Security / Auth` | Identity provider, secrets manager, WAF |

Omit a subgraph entirely if no services fall into it.

**Node naming:** use the provider's real service name, not a generic label. Write `AWS Lambda` not `Function`; `Azure Blob Storage` not `Object Store`.

**Risk annotation:** apply `danger` / `warning` node styling (per artifacts.md color rules) to any service flagged as high or medium likelihood in `## Technical Risks`.

**When to skip:** PaaS, local-first, and managed platform (Path A) projects — note `<!-- Deployment Diagram: not applicable — {reason} -->` in the Architecture section instead.

### Rollout

Deployment strategy (blue-green, canary, rolling); feature-flag usage for risky changes; rollback trigger and mechanism.

**Defer to planning:** feature-flag tooling · canary thresholds.

## Performance & Scalability

This section owns the authoritative answer on load ceiling — other architecture files flag the trigger; infra defines the ceiling.

State the load the current design supports, the first bottleneck, and the migration trigger.

## Observability

**Infrastructure:** error tracking, request tracing, log aggregation (sink + retention), alerting (on-call path).

**AI/LLM (if agent is present):** per call, capture prompt hash, token counts, model, latency, tool calls, response status. Do not log user message content or PII. Align retention with data lifecycle policy.

- **Tracing:** one correlation ID per user request, propagated across all agent calls; span structure for multi-step runs.
- **Cost attribution:** per-feature or per-user; tie to a business metric.
- **Tooling:** select an AI/LLM observability tool; state rationale.

**Defer to planning:** dashboard design · alert threshold values.

## Security & Compliance

**Secrets:** PaaS → platform env vars or external secrets manager. Cloud-native → native secrets manager with rotation frequency. Local-first → env vars or gitignored config with recovery plan. Enumerate LLM API keys and tool credentials. Apply least privilege to agent runtime. State who can access production secrets and how access is audited.

**Encryption:** confirm storage encrypts at rest; if not, state mechanism. HTTPS on all external endpoints; flag internal channels needing mTLS.

**Compliance:** state data residency requirements and which infrastructure layers satisfy them; identify regulatory mandates that constrain hosting or logging choices (GDPR, HIPAA, SOC 2).

**Defer to planning:** rotation automation · audit log schema · mTLS certificate management.

## Operational Decision Nodes

Seeded by solutioning Phase 3 (Gap Resolution). Each requires explicit evaluation. Record machine-readable markers in `solution.md § Operational Concerns`.

**Path A (managed platform):**

1. **Deployment mechanism** — evaluate platform deployment options per Path A § Release Pipeline. Record: `**Infra:** platform:{mechanism}`.
2. **Release governance** — platform-specific approval gates and deployment windows per Path A § Release Pipeline. Record: `**Delivery:** platform:{governance-model}`.

**Path B (cloud/PaaS/local):**

1. **Infrastructure provisioning** — evaluate IaC tools vs. manual provider CLI. If IaC tooling detected → extend it. If none detected → evaluate and choose. Record: `**Infra:** iac:{tool} | cli`.
2. **Application delivery** — evaluate CI/CD vs. manual deploy script. If CI/CD config detected → extend it. If none detected → evaluate and choose. Record: `**Delivery:** cicd:{tool} | manual`.

**Both paths:**

1. **Environment management** — single vs. multi-environment. Record: `**Environments:** single | multi:{dev,staging,prod}`.

**Reminders** (record in `solution.md § Operational Concerns`): confirm secret management mechanism, observability/logging strategy, and ongoing infra cost before planning. If a database is in scope, confirm a migration strategy before planning begins.
