# UX Artifacts

Writing rules and quality check for the `## User Journeys` and `## Layout Design` sections of `prd.md`. Used by assessing Phase 6 (Initial mode) when UX artifacts were confirmed in Phase 2 Step 2.

## Writing Rules

Place both sections AFTER `## Requirements` and BEFORE `## Delivery Estimate`. Omit both entirely if no UX artifacts were confirmed.

**User Journeys:**

- One Mermaid flowchart (`graph TD` format) per primary user path (typically 3–5 journeys).
- Show decisions, branches, happy path, and key error states. Include auth flows if relevant.
- Keep each journey focused: if a flow would exceed 12–15 nodes, split it into focused journeys (e.g. "User Registration" separate from "User Login").
- For multi-actor flows (user + system concurrent actions), consider swim lane diagrams.
- Wrap in ` ```mermaid ... ``` `.

**Layout Design:**

- One ASCII wireframe per unique page/screen type.
- Use Unicode box-drawing (┌─┐│├┤└─┘) unless the project's markdown processor doesn't preserve them. If unsure, ask: "Does your markdown processor preserve Unicode box-drawing in code blocks?" On no, fall back to `+--+|`.
- Label major regions (Header, Nav, Content, Sidebar, Footer, Modal).
- Show proportions (e.g. "280px", "flex-1", "70%/30%").
- If desktop/mobile layouts differ significantly, show both. If responsive breakpoints exist within desktop range, annotate the wireframe with breakpoint behavior (e.g. "sidebar collapses at 1280px").
- Wrap in ` ``` ... ``` ` (no language specifier).

## Quality Check

Run before setting `status: final`. Verify:

- User journeys cover 3–5 primary paths, not every edge case.
- Journeys show decisions and branches, and identify the actor when multiple user types exist.
- No journey exceeds 12–15 nodes. Otherwise split it or use swim lanes for multi-actor flows.
- One wireframe per distinct screen type, with clear spatial arrangement (proportions, labeled regions).
- Responsive behavior noted where desktop/mobile differ or desktop-range breakpoints exist (e.g. 1280px vs 1440px) — annotation or a separate wireframe for the critical breakpoint.
- Box-drawing characters render in the project's markdown processor. If markdown is rendered to HTML for review, confirm `<pre>` blocks preserve them.
- Interaction model stated explicitly (Conversational | Form-driven | Dashboard | Workflow/wizard | Hybrid) with a one-line rationale.
- Platform targets stated (desktop-only / desktop-first / mobile-first / PWA / native) — never ambiguous.

If gaps remain, embed `{open: topic}` callouts inline in the relevant UX section.
