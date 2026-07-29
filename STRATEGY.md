---
name: Athian Compliance
last_updated: 2026-07-27
---

# Athian Compliance Strategy

## Target problem

Food service buyers (CPGs, QSRs, retailers) and their suppliers (co-ops, farmers, producers) spend the majority of their time on manual evidence collection and reconstruction rather than on the compliance judgment itself. There is no centralized system of record: every update requires recompiling evidence from scratch across email, surveys, and documents, a burden that falls on both the buyer's compliance team and the supplier — who re-answers the same questions separately for every buyer relationship.

## Our approach

We model compliance standards as structured, machine-readable specifications in a domain-specific language—each requirement typed with its unit, threshold, time period, required data, and accepted sources—so that collection, review, and reporting become computable rather than manual.

## Who it's for

**Primary:** The buyer — an internal compliance team at a food service CPG, QSR, or retailer. They're hiring Athian Compliance to collect and pre-verify evidence across their entire supplier base without reconstructing it by hand each cycle. The buyer drives the sales motion today.

**Secondary (network effect):** The supplier — a co-op, farmer, or producer. Initially pulled onto the platform because a buyer requires it, but because it makes data collection and reporting so easy, they'll want to use it for all of their buyer relationships. That stickiness is intended to become its own bottom-up sales engine.

**Lower priority for now:** Third-party verification/validation bodies (VVB/CAB, e.g. Sensiba) who want to onboard new standards faster. Gated on DSL onboarding being de-risked.

## Key metrics

- **Time to first assessment** — standard config start to first live assessment run; measured in product
- **AI automation success rate** — elapsed time and pass rate for document ingestion, pre-verification, and related automations; measured in product logs
- **Standard retention + active ratio** — standards retained across renewal cycles; configured-to-active trendline; measured in DB
- **Time to onboard a standard** — config start to marking a standard as active; measured in product
- **Time saved per assessment** — AI automation savings vs. user-provided pre-product baseline; measured via qualitative survey

## Tracks

### DSL + Standard Onboarding

Defining and refining the DSL, the API layer, and the UI flow for getting new standards onto the platform.

_Why it serves the approach:_ The DSL is the approach — formalizing standards as machine-readable specs is what makes every downstream automation possible.

### AI Automation

Document ingestion and classification, pre-verification, and report generation.

_Why it serves the approach:_ This is where the DSL's computable structure translates into real time savings for the compliance user — the core value proposition.

### Assessment Workflow

The full evidence exchange cycle: compliance user requesting and reviewing evidence, pre-verification report review, back-and-forth with the supplier about gaps, and supplier evidence submission.

_Why it serves the approach:_ The primary persona's day-to-day experience — the UI that makes automated evidence collection and review usable in practice.

### Trial Experience

A self-serve trial where a prospect uploads a set of requirements (an internal policy, an industry scheme, a regulatory framework) plus a document or evidence, and sees whether it "complies." Aimed mostly at buyers, though nothing stops a supplier from trying it.

_Why it serves the approach:_ Makes the offering tangible — substance over hype — by running the DSL and AI automation live on the prospect's own material. As it matures, it shortens the sales cycle.

## Milestones

- **2026-07-01** — Nestlé POC delivered successfully (AI automation output)
- **2026-08 (mid-month)** — CDI audit begins (part of the Nestlé pilot arc)
- **2026-09-30** — Pilot delivered to Nestlé (web app)
