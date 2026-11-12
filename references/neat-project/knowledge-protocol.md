# Knowledge Protocol

Answers questions about the existing system from documented system context, so the main session does not have to load it all. Used by planning, building, and fixing. The query subagent never writes files.

**Core principle:** every answer is classified by evidence.

- **Proved** (concrete file reference) → main session uses it silently and lists it in the batch summary.
- **Inferred** (judgment or assumption) → main session shows it to the user immediately for confirmation.

## When to Use

Use for questions about:

- Module capabilities: "Does X module support Y?"
- Patterns and conventions: "What's the established pattern for Z?"
- Ownership: "Which module handles W?"
- Existence: "Do we have V?"
- Integration: "How does A connect to B?"
- Deployment and operations: "How do we deploy?", "What's the rollback procedure?"
- Architecture decisions: "Why did we choose X over Y?"
- External API contracts: "What Stripe API version do we use?", "How do we handle GitHub webhooks?"

Do not use when:

- The answer is already in loaded context (check first)
- The question needs a user decision or user knowledge
- The work is exploratory research (use a general-purpose subagent)
- The caller is a discovery skill (assessing, solutioning, and modernizing create this knowledge rather than query it)

Batch 2–5 related questions per spawn. Spawn at most 5 batches in parallel.

## Evidence Classification

**Proved:** a direct quote or field value from an existing file in system documentation. Examples:

- Module exports: `"exports": ["validateOAuth2Token()"]` in `modules/auth-service.md`
- An explicit sentence in `system.md § Patterns`
- Contract listed in `system.md § Contracts / Inbound Contracts`
- A file that exists at the cited path
- A dependency listed in `external_imports`
- Deployment steps in `runbooks/deployment.md`
- Architecture decision in `architecture/decisions/adr-003-database.md`
- External API details in `integrations/stripe-api.md`

**Inferred:** judgment, best practice, or indirect signals. Typical wording: "probably", "likely", "appears to", "recommend", "based on". Examples:

- Dependency → capability: "Has pdfkit → probably supports PDF generation"
- Directory → functionality: "`/webhooks` folder → likely handles webhooks"
- Naming → purpose: "`EmailService` class → probably sends email"
- Framework convention: "NestJS project → uses decorators for routing"

**Proved, but always shown to the user:**

- **Not found:** all modules searched and nothing found (proved absence)
- **Contradiction:** `system.md` says X, a module file says Y. Blocks until resolved.
- **Multiple options:** both patterns exist (e.g. JWT and session). The user picks.

## Invocation

Spawn one subagent per batch:

```text
Agent({
  description: "Query system context",
  prompt: `Follow {knowledge-protocol-path} § Subagent Execution to answer these questions:

1. {question}
2. {question}

Classify each answer as proved or inferred.`
})
```

`{knowledge-protocol-path}` is the resolved absolute path of this file. The main session continues with work that does not depend on the answers while the subagent runs.

## Subagent Execution

**Step 1 — Load base context:**

- `specs/system/system.md` (all sections)
- `specs/system/modules/index.md`
- `specs/discovery/solution.md`

**Step 2 — Discover additional knowledge artifacts:**

Scan `specs/system/` for all `.md` files (excluding `modules/` which has its own index):

```bash
find specs/system -name "*.md" -not -path "*/modules/*" -not -name "system.md"
```

Group files by directory and note structure:

```text
specs/system/
  architecture/
    README.md (if exists, describes what's in this directory)
    decisions/
      adr-001-api-versioning.md
      adr-002-database-choice.md
    diagrams.md
  runbooks/
    deployment.md
    troubleshooting.md
  integrations/
    stripe-api.md
```

Build a knowledge inventory. For each directory with docs:

- If `README.md` exists → read it for directory context (what's inside, when to use)
- List all .md files with their paths

**Step 3 — Load modules progressively:**

- Question names a module ("Does **auth-service** support OAuth2?") → load `specs/system/modules/{module}.md`.
- Question asks which module owns something → scan `index.md` first, then load the matching module file.
- Question asks about a pattern → `system.md § Patterns` is already loaded; load nothing more.

**Step 4 — Match additional docs by keywords:**

For each question, extract keywords and match against discovered docs:

- **File path matching**: "stripe" → `integrations/stripe-api.md`
- **Directory matching**: "deployment" → `runbooks/deployment.md`
- **README context**: If directory README mentions relevant keywords
- **Smart loading**: Load matched docs (max 3 per question to avoid context bloat)

If multiple docs match and relevance is unclear, note all matches in the answer for user selection.

**Step 5 — Answer each question:**

- Determine the question type: module capability, pattern, ownership, existence, integration, or general system knowledge.
- Search in this order per type:
  - **Module capability** → `exports` (proved) → module file if `type=api` (proved) → `external_imports` (presence proved, capability inferred)
  - **Pattern/convention** → `system.md § Patterns` → `system.md § Conventions` → matched docs from Step 4 (architecture/, ADRs) → `solution.md` (proved when explicit)
  - **Ownership** → `index.md` Key Exports → the module's `exports` (proved)
  - **Existence** → `index.md` → all module exports → `system.md § Contracts` → matched docs (proved when found)
  - **Integration** → `internal_imports` → both modules' exports → matched docs (integrations/) → `system.md § Contracts` (proved)
  - **Deployment/Operations** → matched docs (runbooks/, operations/) → `system.md § Foundation` (proved when found in docs)
  - **Architecture decisions** → matched docs (architecture/decisions/, ADRs) → `solution.md` → `system.md § Architecture` (proved when explicit)
  - **External API contracts** → `system.md § Contracts / Outbound Contracts` → matched docs (integrations/) (proved when found)
- Classify: exact match in a file → `proved`. No match but signals exist → `inferred`. No match and no signals → answer `Not found`, evidence `proved`.
- Validate every `proved` answer: the cited file exists and contains the quoted text. Otherwise downgrade to `inferred`.

**Step 6 — Return batch results:**

```jsonc
{
  "results": [
    {
      "question": null,      // original question text
      "answer": null,        // structured information, not just yes/no; "Not found" for proved absence
      "evidence": null,      // "proved" | "inferred"
      "source": null,        // "file:line" for proved; files analyzed for inferred; search scope for not found
      "reasoning": null,     // inferred only: why this conclusion
      "signals": [],         // inferred only: signals that led to the inference
      "recommendation": null, // not found only: { "suggestion": "...", "evidence": "inferred", "reasoning": "..." }
      "show_user": null      // "batch" for proved; true for inferred, not found, contradiction, multiple options
    }
  ],
  "summary": {
    "total_questions": null,
    "proved": null,
    "inferred": null,
    "show_user_count": null
  }
}
```

Compare your output against this schema — confirm all required keys are present.

## Main Session Handling

For each result:

- `show_user: true` → present immediately, wait for confirmation, continue based on the answer.
- `show_user: "batch"` → use silently and keep it for the batch summary.

Inferred answer format:

> Inferred: {answer}
> Reasoning: {reasoning}
> Signals: {signals}
> Proceed with this? [1] Yes  [2] No — I'll give the answer  [3] Other

Not-found format:

> Not found: {capability}
> Details: {what was searched and found}
> Recommendation (inferred): {suggestion} — {reasoning}
> Proceed? [1] Yes  [2] No  [3] Other

## Main Session Usage

When a question about modules, patterns, ownership, or existing capabilities comes up that loaded context does not answer:

- Batch 2–5 related questions.
- Spawn the query subagent per § Invocation.
- Present inferred, not-found, contradiction, and multiple-option answers immediately per § Main Session Handling.
- Use proved answers silently and keep them for § Batch Summary.

## Batch Summary

Run at the calling skill's checkpoint (building: wave complete; planning: wave planned; fixing: fix complete) when any query returned proved answers. The caller supplies `{checkpoint}`.

> {checkpoint}
> Research used (proved sources):
>
> - {source description} → {file:line}
>
> All sources correct? [1] Yes  [2] No

On [2]:

- Ask which source is wrong and note the correction.
- Ask: "Update system.md?" [1] Yes  [2] No
- On [1], update the relevant `system.md` section immediately.
