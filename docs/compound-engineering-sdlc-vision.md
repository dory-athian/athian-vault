# Compound Engineering SDLC Evolution — Athian Vision & Adoption Plan

*Author: Dory Weiss (VP Engineering & Product) — June 2026*

---

## The Core Idea

Compound Engineering's premise is simple and powerful: 80% of the effort should live in planning and review; 20% in execution. Each cycle should leave behind artifacts that make the next cycle faster and smarter — compound notes, plans, ADRs, test patterns, pulse reports. The codebase doesn't just accumulate features; it accumulates *judgment*.

The question for Athian isn't whether this is the right model. It clearly is, especially for a team our size where institutional knowledge and context-switching costs are existential. The question is how we wire it into the rhythms we already have — our signal sources, our toolchain, our team structure — and how we phase adoption in a way that doesn't create a second job for engineers.

This document captures the long-term vision and a pragmatic path to get there.

---

## Long-Term Vision: The Compounding Intelligence Loop

The mature state looks like this:

```
Market signals ──┐
Customer calls ──┤                    ┌── Code
Bug reports ─────┤→ [Signal Layer] ──→│── Documentation
PostHog errors ──┤                    │── Compound notes
Team convos ─────┘                    └── Marketing collateral
        ↑                                         │
        └──────────── [Pulse + Review] ←──────────┘
```

The loop is:

1. **Signals flow in continuously** from all sources and get themed into durable insight.
2. **Strategy stays anchored** to those themes via `STRATEGY.md`, maintained quarterly.
3. **Features originate from signal** — not from intuition alone — moving through ideate → brainstorm → plan → work → review.
4. **Each completed feature compounds** — leaving behind notes, patterns, and test artifacts that the next engineer (or agent) can build on.
5. **Pulse reports close the loop** — pulling PostHog data and error signals back into strategy.

The flywheel: the longer this runs, the better the AI's plans become, because the compound knowledge base grows. On a team of eight engineers, this is the difference between always starting from scratch and actually accumulating engineering leverage.

---

## The Signal Layer: Continuous Aggregation

This is the most novel and highest-value piece — and it builds directly on the Athian Feedback Hub work already in progress.

### What gets captured

| Source | Signal type | Collection mechanism |
|--------|-------------|----------------------|
| Sales/CS team conversations | Pain points, competitive mentions, feature gaps | Granola → n8n |
| Prospect/customer/partner calls | Detailed product feedback, objections, requests | Granola → n8n |
| Engineering/product team convos | Technical constraints, design decisions, risks | Granola → n8n |
| Bug reports | Recurring failures, edge cases, data integrity issues | GitHub Issues / Linear → n8n |
| PostHog errors | Production failures, JS errors, funnel drops | PostHog webhook → n8n |
| Support cases | User confusion, onboarding gaps, documentation failures | CS system → n8n |

### How signals become intelligence

Every signal source feeds n8n, which applies a classification layer (using Claude) to extract structured fields: signal type, theme, affected module, severity, verbatim quote, source, date. These land in a structured store (Google Sheet or, better, a Notion database once we migrate).

Signals are then **themed weekly** — grouped by recurring topic — and surfaced as a Signal Digest. This digest becomes:
- Input to the weekly pulse report
- Input to `/ce-strategy` updates
- Input to brainstorm sessions ("three customers mentioned X this week")

The existing Feedback Hub covers the Granola → structured store path. The extension is: (a) add PostHog and bug sources, (b) add a theming/digest step, and (c) wire the digest into the compound engineering workflow explicitly.

### The `/ce-product-pulse` integration

CE's product pulse skill generates a time-windowed report on what users actually experienced — usage patterns, error rates, performance, anomalies. Combined with the Signal Digest, this becomes the primary observability surface for Dory and Mike. Saved to `docs/pulse-reports/`, it forms a browseable timeline of product health.

The target rhythm: pulse report generated automatically weekly, reviewed by Dory every Monday morning, used to inform the next `/ce-strategy` update.

---

## The SDLC Loop: How Work Gets Done

### Layer 1 — Strategy (Quarterly, or when signals demand it)

**Owned by:** Dory + Jess, with Royce on technical strategy  
**Artifact:** `STRATEGY.md` at repo root  
**Skill:** `/ce-strategy`

`STRATEGY.md` is the single-page anchor: target problem, approach, persona, key metrics, current tracks. Every downstream artifact — brainstorms, plans, ideation — is grounded to it. It gets updated when signal themes or market conditions shift materially, not on a rigid schedule.

Royce should maintain a parallel technical version scoped to architecture: current technical bets, known constraints, non-negotiable invariants. This prevents `/ce-plan` from proposing things that violate architectural coherence.

### Layer 2 — Ideation (As needed, for big open problems)

**Owned by:** Dory, Jess, or Royce  
**Artifact:** Ranked ideation doc in `docs/ideations/`  
**Skill:** `/ce-ideate`

This is the upstream step before brainstorming — useful when the problem space is genuinely open and you want the agent to generate and critically evaluate multiple approaches before committing to one. Not every feature needs this; reserve it for strategic features or novel technical challenges.

Jess is a natural user of `/ce-ideate`. She can bring a design-space question ("we need to rethink the producer onboarding flow") and get a structured set of approaches before she ever opens Figma.

### Layer 3 — Requirements and Mockups (Per Feature)

**Owned by:** Jess (product/design), relevant engineer  
**Artifacts:** Requirements doc in `docs/brainstorms/`, Figma mockup  
**Skill:** `/ce-brainstorm`

This is where the compound loop starts in earnest. `/ce-brainstorm` runs an interactive Q&A to produce a right-sized requirements doc. Jess's Figma mockups are explicitly an input here — not a parallel artifact, but a cited reference in the requirements doc.

The requirements doc gets reviewed through `/ce-doc-review`, which spins up multiple persona agents:
- `ce-product-lens-reviewer` — challenges problem framing and scope
- `ce-feasibility-reviewer` — evaluates whether the approach will survive contact with the codebase
- `ce-design-lens-reviewer` — ensures interaction states and design decisions are addressed
- `ce-scope-guardian-reviewer` — challenges premature abstraction

Royce should be tagged on any requirements doc that touches core data models or cross-service boundaries.

### Layer 4 — Architecture Decisions (As needed)

**Owned by:** Royce, with Jeff/Jason as domain owners  
**Artifact:** ADR in `docs/adrs/`  
**Skill:** `/ce-brainstorm` or `/ce-plan` with Royce as reviewer

Not every feature needs an ADR. The bar: anything that changes a data model, introduces a new abstraction, affects cross-service contracts, or has long-term reversibility implications. Royce is the sign-off authority. The ADR format should align with the existing `docs/solutions/` pattern (YAML frontmatter with `module`, `tags`, `problem_type`).

### Layer 5 — Implementation Planning (Per Feature)

**Owned by:** The engineer who will execute (Jeff, Jason, or Carolyn)  
**Artifact:** Implementation plan  
**Skill:** `/ce-plan`

`/ce-plan` turns the requirements doc into a detailed, step-by-step implementation plan. For backend features (Jeff/Jason), this includes API contracts, migration steps, and service boundaries. For frontend features (Carolyn), this includes component structure, state management approach, and Figma → code alignment.

Before execution begins, Jenn reviews the test plan section. This is Jenn's primary insertion point in the AI-assisted loop — she's the judgment layer on AI-generated test coverage, not the generator. Her review should explicitly answer: "Are the test cases sufficient? Are there coverage gaps? Does this match how the feature will actually be used?"

### Layer 6 — Execution

**Owned by:** Jeff, Jason, or Carolyn  
**Tool:** Claude Code + Warp/VSC + CE plugin  
**Skills:** `/ce-work`, `/ce-debug`

`/ce-work` executes plans in worktrees with task tracking. Engineers work in their preferred environment. The key discipline: don't skip the plan. `/ce-work` is most powerful when it has a well-formed plan to execute against — it becomes dramatically less effective as a "just build this" invocation.

`/ce-debug` handles failures systematically: reproduce → trace causal chain → form testable hypotheses → implement fix. The output feeds directly into `/ce-compound` (the fix pattern is worth preserving).

### Layer 7 — Review (Per PR)

**Owned by:** Engineer + domain owners + Royce (for architecture-touching changes)  
**Skills:** `/ce-code-review`, `/ce-doc-review`

`/ce-code-review` runs multi-agent review with specialized personas:
- `ce-correctness-reviewer` — logic errors, edge cases
- `ce-security-reviewer` — vulnerabilities
- `ce-performance-reviewer` — runtime performance
- `ce-testing-reviewer` — test coverage gaps
- `ce-maintainability-reviewer` — coupling and complexity
- `ce-data-integrity-guardian` — migration safety
- `ce-architecture-strategist` — ADR compliance (for Royce)

Domain-specific reviewers map to team ownership:
- Backend (Jeff/Jason): `ce-data-migrations-reviewer`, `ce-reliability-reviewer`
- Frontend (Carolyn): `ce-design-implementation-reviewer` (Figma → code alignment), `ce-julik-frontend-races-reviewer`

Jenn reviews AI-generated test plans before `/ce-work` begins, not after. The code review stage is a secondary QA gate, not the primary one.

### Layer 8 — Knowledge Capture (Post-Merge)

**Owned by:** The engineer who did the work  
**Artifact:** Compound note in `docs/solutions/` or `.compound-engineering/`  
**Skills:** `/ce-compound`, `/ce-compound-refresh`

This is the compounding mechanism. Every feature leaves behind at least one compound note documenting:
- What the problem was
- What was tried and why
- What worked and why
- What would have been faster in hindsight
- Gotchas for the next person (or agent) touching this area

The `docs/solutions/` directory already has a YAML frontmatter structure (`module`, `tags`, `problem_type`). This becomes the canonical compound knowledge store. Over time, `/ce-plan` and `/ce-brainstorm` pick this up as grounding, so the AI's plans get progressively better-calibrated to Athian's actual codebase.

`/ce-compound-refresh` runs periodically (or before major refactors) to update stale notes.

---

## Role-Specific Integration

### Royce — Lead Architect

Royce's primary touchpoints are upstream (strategy/ADRs) and at review. He's not writing code for most features, so forcing him into the execution loop doesn't make sense. What does:
- Co-owns `STRATEGY.md` technical sections
- Reviews requirements docs for anything touching core architecture
- Primary reviewer for ADRs; his compound notes on architectural decisions are high-value institutional memory
- `ce-architecture-strategist` flags things for his attention during code review, so he doesn't have to read every diff

### Jeff — Backend

Full loop: brainstorm → plan → work → review → compound. Jeff's compound notes on backend patterns, data migration approaches, and service boundaries will become some of the most-referenced artifacts. He should lean into `/ce-debug` as a first-class debugging workflow rather than ad-hoc investigation.

### Jason — Integrations

Same as Jeff, but Jason's compound notes skew toward integration patterns — retry logic, idempotency, external API quirks. These are exactly the kind of tribal knowledge that disappears when people leave. `/ce-issue-intelligence-analyst` is useful for Jason specifically: it can analyze patterns in GitHub/Linear integration issues to surface recurring failure modes before they become fires.

### Carolyn — Frontend

The design-implementation review agents (`ce-design-implementation-reviewer`, `ce-figma-design-sync`) are Carolyn's leverage multipliers. She can run `/ce-brainstorm` with a Figma link as input and get a requirements doc that includes component breakdown, state management considerations, and interaction states — before she writes a line of code.

Compound notes from Carolyn on frontend patterns (state management, component conventions, Athian Design System gotchas) are high-value because they bridge design-to-code in ways that generic AI context doesn't capture.

### Jenn — QA Lead

Jenn's role in the AI-assisted SDLC shifts from *test writer* to *test judgment layer*. She reviews AI-generated test plans before execution begins — this is a meaningful gate, not a rubber stamp. Her compound notes on QA patterns (what kinds of test plans the AI gets wrong, what coverage areas it consistently misses) feed directly back into future plan generation.

Over time, Jenn should maintain a `docs/solutions/testing/` section that captures test patterns by module — the AI gets better at generating test plans as this library grows.

### Mike — CTO

Mike's observability = pulse reports + `STRATEGY.md` + compound note trends. He shouldn't need to read individual PRs unless he chooses to. The target: a weekly digest (auto-generated, reviewed Monday morning) that synthesizes the product pulse, signal themes, and notable compound learnings from the prior week. This is the "jump in when you choose" interface — everything is grounded and legible without requiring context he doesn't have.

### Dory — VP Engineering & Product

Same observability model as Mike, but Dory's deeper engagement is at the brainstorm and strategy layers — running `/ce-brainstorm` for high-stakes features, reviewing `/ce-strategy` updates, and making prioritization calls informed by the signal digest. The goal is that Dory can stay in the loop without being in the weeds of execution.

### Jess — Director of Product & Design

Jess's unlock is the biggest non-obvious opportunity here. With CE + Figma + Claude Code, she can:
1. Run `/ce-ideate` on a design problem to evaluate approaches before committing in Figma
2. Produce a Figma prototype with Figma's AI
3. Run `/ce-brainstorm` with the Figma link to auto-generate a requirements doc that includes component structure, interaction states, and design decisions
4. Optionally: `/ce-work` to generate a static prototype from the Figma — without writing code herself

The compound engineering flow for Jess doesn't require her to be a developer. It requires her to be a rigorous thinker about requirements — which she already is. The AI does the translation.

---

## Tooling Decisions

### Issue Tracker: Move to Linear (Full)

The Compliance work is already in Linear. The migration from Jira should complete, not stall. Linear's issue model aligns better with the compound engineering workflow — lightweight, fast, git-integrated. The `ce-issue-intelligence-analyst` agent works against GitHub Issues; the workaround for Linear is to use Linear's GitHub sync or the Linear MCP (which we already have).

Recommendation: complete Linear migration within 90 days. Don't maintain two systems past the current Compliance pilot window.

### Knowledge Base: Lean Into Git-First

The compound engineering artifacts (brainstorms, plans, ADRs, compound notes) live best in the repo itself — versioned, linkable, and surfaced automatically to the AI. `docs/solutions/` is already structured well. Extend with:
- `docs/brainstorms/` — requirements docs from `/ce-brainstorm`
- `docs/plans/` — implementation plans from `/ce-plan`
- `docs/adrs/` — architecture decision records
- `docs/pulse-reports/` — product pulse reports from `/ce-product-pulse`
- `docs/strategies/` — STRATEGY.md history

For human browsing, a Notion or Obsidian layer on top is fine — but the source of truth is the repo. Don't build the knowledge management system primarily in Notion and then try to wire it to the AI; build it in the repo and mirror it to Notion.

### Confluence: Replace with Notion (or Obsidian for eng-internal)

If/when Confluence migration happens, Notion is the right choice for cross-functional content (product specs, runbooks, customer-facing docs). Engineers can use Obsidian as a personal layer that syncs to the repo. The compound knowledge base itself stays in Git.

---

## Phased Adoption Plan

### Phase 0 — Foundation (Weeks 1–2)

*Goal: Get the tool in place without changing how people work yet.*

- Install CE plugin in `platform` and `sustainability-platform`
- Run `/ce-setup` in both repos
- Create initial `STRATEGY.md` with `/ce-strategy` (Dory + Jess + Royce)
- Identify one in-flight feature to run through the full loop as a pilot

Engineering team introduction: one 45-minute session explaining the philosophy (not the commands). The goal is understanding *why* — each cycle should make the next one easier — before people start cargo-culting slash commands.

### Phase 1 — Execution Loop (Weeks 2–6)

*Goal: Jeff, Jason, and Carolyn running brainstorm → plan → work → review → compound on all new features.*

Start with small features. The first time through the loop will feel slow — resist the urge to skip steps. The value isn't visible in the first cycle; it compounds over the second and third.

Compound notes: establish the convention that every merged feature produces at least one compound note. Dory reviews compound notes in weekly 1:1s initially to ensure quality.

Jenn inserts at the plan stage: she reviews the test plan section before `/ce-work` begins.

### Phase 2 — Signal Integration (Weeks 4–8, overlapping Phase 1)

*Goal: Automated signal collection and weekly digest in place.*

Complete the Athian Feedback Hub: Granola → n8n → structured signals. Add PostHog error events and GitHub/Linear bug signals to the ingestion pipeline. Implement a weekly theming run that produces a Signal Digest.

Wire the Signal Digest into the Monday review: Dory reads it alongside the pulse report. This is the moment when product decisions start being visibly grounded in real signal rather than intuition.

First use of `/ce-product-pulse`: generate a 7-day pulse report and review with Mike. Calibrate what "observability" means for him — is it product health metrics? Engineering health? Both?

### Phase 3 — Product Loop (Weeks 6–12)

*Goal: Jess running brainstorm from Figma; ADRs as standard practice; signal themes formally feeding into strategy.*

Jess pilot: take one design initiative through the full CE workflow (ideate → Figma → brainstorm → requirements doc → plan → Carolyn executes). Document what works and what doesn't for non-coders running the front half of the loop.

ADRs: establish the ADR trigger criteria (data model changes, new abstractions, cross-service contracts) and Royce's sign-off process. First three ADRs should be retrofitted for recent architectural decisions as practice.

`/ce-strategy` first full update: incorporate the first 6 weeks of signal themes and pulse data. This update should be visibly different from the initial strategy because it's grounded in real data.

### Phase 4 — Full Flywheel (Months 3–6)

*Goal: The compound knowledge base is self-evidently useful; signal → strategy → plan → code → review → compound is the default, not the exception.*

At this point, the AI's plans should be noticeably better than they were in Phase 1 — because the compound notes give it real institutional context about Athian's codebase, patterns, and failure modes.

Jess prototyping autonomously: she can go from a customer conversation to a Figma prototype to a requirements doc without needing an engineer in the loop until plan review.

Mike's observability: weekly digest auto-generated every Friday afternoon, covers product pulse + signal themes + notable compound learnings. He can read it in 10 minutes and have a real picture of what the team built and what customers experienced.

The signal → code → signal loop is closed: compound learnings feed back into future brainstorms, pulse reports feed back into strategy, and the team stops rediscovering the same lessons.

---

## Key Risks and Mitigations

**Risk: The loop feels like ceremony, engineers skip steps.**  
Mitigation: Start with the steps that create immediate individual value (brainstorm → better plan, plan → less rework, code-review → caught before merge). Don't enforce compound notes through process — make them feel valuable by demonstrating that the AI actually uses them in subsequent sessions.

**Risk: Jenn's QA review gate becomes a bottleneck.**  
Mitigation: Jenn reviews test plan sections asynchronously at the plan stage, not at PR review. Her goal is quality judgment, not approval blocking. If test plans are consistently good, she moves to spot-check mode.

**Risk: Signal aggregation adds noise rather than signal.**  
Mitigation: The theming step is critical. Raw signals are useless; themed patterns are actionable. Invest in the Claude prompt that themes signals — this is where most of the intelligence lives.

**Risk: Jess's non-coder workflow breaks on edge cases.**  
Mitigation: Carolyn is the fallback. Jess's CE workflow gets her to a requirements doc; Carolyn owns the plan and execution. The goal isn't full autonomy for Jess, it's removing the "I need a sprint before we can even prototype this" friction.

**Risk: Two issue trackers (Jira + Linear) create confusion about where the canonical state lives.**  
Mitigation: Complete the Linear migration on a hard deadline. The current Compliance pilot in Linear is the forcing function. Set a date.

---

## Open Questions

1. **Pulse report cadence and audience**: Weekly for Dory; what frequency and format does Mike actually want? This needs a direct conversation, not an assumption.

2. **Compound note ownership**: Should compound notes be owned by the engineer who wrote the feature, or reviewed/edited collaboratively? Starting with individual ownership and lightweight 1:1 review is probably right.

3. **Signal Digest format**: How structured should the weekly digest be? A ranked list of themes with verbatim quotes is more useful than a prose summary. Worth piloting two formats and seeing which actually gets used.

4. **ADR retroactive coverage**: Which existing architectural decisions are most important to document retroactively? Royce and Jeff should identify the top five — areas where tribal knowledge is highest-risk.

5. **CE + `/ce-slack-research`**: CE has a Slack research agent. Worth evaluating whether this can supplement or replace part of the Granola → n8n signal pipeline for team conversations.

---

## What Success Looks Like in 6 Months

- Every new feature has a requirements doc, implementation plan, and compound note as a matter of course — not because it's required, but because it's faster than not doing it.
- The weekly pulse report + signal digest is the primary mechanism through which Dory and Mike stay oriented to product health.
- Jess has shipped at least two features that started entirely from a Figma prototype she drove through `/ce-brainstorm` to a requirements doc, with no engineer involvement until plan review.
- The compound note library has at least 30 entries and is visibly influencing the quality of AI-generated plans.
- The team can onboard a new engineer in significantly less time because the institutional knowledge is written down in a form the AI can use.
