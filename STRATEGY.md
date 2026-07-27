---
name: Athian Compliance
last_updated: 2026-06-22
---

# Athian Compliance Strategy

## Target problem

Compliance professionals—internal specialists at CPG companies and third-party auditors—spend the majority of their time on manual evidence collection and reconstruction rather than on the compliance judgment itself. There is no centralized system of record: every update requires recompiling evidence from scratch across email, surveys, and documents, a burden that falls on both the compliance pro and the supplier.

## Our approach

We model compliance standards as structured, machine-readable specifications in a domain-specific language—each requirement typed with its unit, threshold, time period, required data, and accepted sources—so that collection, review, and reporting become computable rather than manual.

## Who it's for

**Primary:** Internal compliance specialist at a large CPG company — they're hiring Athian Compliance to automate evidence collection and pre-review across thousands of suppliers so they can focus on the compliance decision itself rather than data assembly.

**Secondary:** Third-party auditing firms (e.g., Sensiba) who want to expand business by onboarding new standards faster. This persona is gated on DSL onboarding being de-risked.

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

## Milestones

- **2026-07-01** — POC delivered to Nestle (AI automation output)
- **2026-09-30** — Pilot delivered to Nestle (web app)
