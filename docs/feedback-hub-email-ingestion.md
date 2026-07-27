# Feedback Hub — Email Ingestion Workflow

**Status:** Implementation spec — ready to build in n8n  
**Date:** 2026-07-02  
**Relates to:** Granola ingestion workflow (Slice 2b)

---

## Overview

This workflow listens for inbound emails to a Gmail inbox, extracts a Raw Feedback row and zero or more Problem rows, and writes them to the same Google Sheets tabs used by the Granola flow.

**Design constraint:** The Claude system prompts are identical to the Granola workflow. Only the user-turn message changes — it adapts email headers + body into the same structural format the prompts already expect. This guarantees output consistency across sources without prompt divergence.

---

## Prerequisite: Rename in n8n and Sheets

Before building the email flow, apply these renames to the existing Granola workflow:

| Where | Old name | New name |
|---|---|---|
| Google Sheet tab | `Meetings` (if named that) | `Raw Feedback` |
| n8n node writing to that tab | `Write Meeting Row` (or similar) | `Write Raw Feedback Row` |

The sheet column `feedback_raw` is already correctly named in the schema doc.

---

## Workflow Architecture

```
[1] Gmail Trigger
      ↓
[2] Filter: Is this a relevant external email?
      ↓ (yes)
[3] Format Email for Claude
      ↓
[4] Build Raw Feedback Prompt      ← same system prompt as Granola
      ↓
[5] HTTP Request → Claude (Haiku)
      ↓
[6] Parse Raw Feedback Response
      ↓
[7] Assemble Raw Feedback Row
      ↓
[8] Write to Raw Feedback sheet
      ↓
[9] Build Problems Prompt          ← same system prompt as Granola
      ↓
[10] HTTP Request → Claude (Sonnet)
      ↓
[11] Parse Problems Response
      ↓
[12] Split Problems into Items
      ↓
[13] Assemble Problem Row (per item)
      ↓
[14] Write to Problems sheet
```

---

## Node Specifications

---

### [1] Gmail Trigger

**Type:** Gmail Trigger  
**Operation:** Watch for new emails  
**Configuration:**
- Credentials: Gmail OAuth2 — authenticate as `feedback-hub@athian.ag` (Google Group shared inbox; ensure the authenticating account has full mailbox access)
- Poll interval: Every 1 minute (or use Push via Gmail webhook if available in your n8n instance)
- Filters: Leave blank — filtering is handled in node [2] so logic is visible and editable

**Output fields used downstream:**
- `from.value[0].address` — sender email
- `from.value[0].name` — sender name
- `to.value[0].address` — recipient
- `subject` — email subject line
- `date` — ISO date string
- `text` — plain text body (prefer over `html`)
- `messageId` — used as dedup key / source_url
- `html` — fallback if `text` is empty

---

### [2] Filter: Is this a processable email?

**Type:** IF node  
**Purpose:** Drop automated noise (bounces, autoreplies, delivery reports). Allow everything else — including emails from Athian employees, who are legitimate senders forwarding external emails or injecting ad hoc notes.

**Conditions (ALL must be true to proceed):**

```
1. from.value[0].address  does NOT contain  noreply
2. from.value[0].address  does NOT contain  no-reply
3. from.value[0].address  does NOT contain  mailer-daemon
4. subject                does NOT match regex  ^(Auto:|Automatic reply|Out of Office|Undelivered|Delivery Status Notification)
```

> Athian `@athian.ag` / `@athian.ai` senders are explicitly **allowed** — they are contributors forwarding external email or writing ad hoc notes.

**True branch** → proceeds to node [3]  
**False branch** → NoOp / end execution (no error needed — automated noise is expected)

---

### [3] Format Email for Claude

**Type:** Code node  
**Purpose:** Normalise the Gmail trigger output into a clean structured object. Handles three email modes:

- **External inbound** — an external contact emailed `feedback-hub@` directly
- **Forwarded** — an Athian employee forwarded an external email; the real sender is in the body
- **Ad hoc note** — an Athian employee wrote their own observations directly

The mode is detected automatically. `claudeUserMessage` is always formatted the same way so both Claude prompts need no changes. The `contributor` field captures the Athian employee when they are the sender.

```javascript
const item = $input.item.json;

const ATHIAN_DOMAINS = ['@athian.ag', '@athian.ai'];

const fromAddress = item.from?.value?.[0]?.address || '';
const fromName    = item.from?.value?.[0]?.name || '';
const rawSubject  = item.subject || '(no subject)';
const messageId   = item.messageId || '';
const emailDate   = item.date ? new Date(item.date).toISOString().split('T')[0] : new Date().toISOString().split('T')[0];

// Prefer plain text; fall back to stripped HTML
const rawBody = item.text || (item.html || '').replace(/<[^>]+>/g, ' ').replace(/\s+/g, ' ').trim();
const body = rawBody.length > 6000 ? rawBody.slice(0, 6000) + '\n\n[truncated]' : rawBody;

const isAthianSender = ATHIAN_DOMAINS.some(d => fromAddress.includes(d));

// ── Detect forwarded email ────────────────────────────────────────────────────
// Gmail forwards typically insert a "---------- Forwarded message ---------"
// block with From:/Date:/Subject:/To: headers inside the body.
const fwdHeaderMatch = body.match(
  /[-]{5,}\s*Forwarded message\s*[-]{5,}\s*From:\s*(.+?)\s*<([^>]+)>/i
);
const isForwarded = /^fwd?:/i.test(rawSubject) || !!fwdHeaderMatch;

// ── Resolve effective sender and contributor ──────────────────────────────────
let effectiveSenderName, effectiveSenderAddress, contributor, emailMode, subject, date;

if (isAthianSender && isForwarded && fwdHeaderMatch) {
  // Forwarded: Athian employee is the contributor; original external sender is the contact
  contributor           = fromAddress;
  effectiveSenderName   = fwdHeaderMatch[1].trim();
  effectiveSenderAddress = fwdHeaderMatch[2].trim();
  // Try to pull the original date from the forwarded block
  const fwdDateMatch = body.match(/From:.*\nDate:\s*(.+)/i);
  date    = fwdDateMatch ? new Date(fwdDateMatch[1]).toISOString().split('T')[0] : emailDate;
  // Strip "Fwd:" / "FW:" prefix from subject for cleaner meeting title
  subject = rawSubject.replace(/^(fwd?:\s*)+/i, '').trim();
  emailMode = 'forwarded';

} else if (isAthianSender && !isForwarded) {
  // Ad hoc note: Athian employee is both contributor and author; no external contact
  contributor           = fromAddress;
  effectiveSenderName   = fromName;
  effectiveSenderAddress = fromAddress;
  subject   = rawSubject;
  date      = emailDate;
  emailMode = 'adhoc_note';

} else {
  // External inbound: external contact emailed the inbox directly
  contributor           = 'feedback-hub@athian.ag';
  effectiveSenderName   = fromName;
  effectiveSenderAddress = fromAddress;
  subject   = rawSubject;
  date      = emailDate;
  emailMode = 'external_inbound';
}

// ── Build claudeUserMessage ───────────────────────────────────────────────────
// Always mirrors the Granola format so both Claude prompts work unchanged.
// For ad hoc notes, the "attendees" line signals no external participant,
// which triggers the prompt's internal-call handling rules automatically.
const attendeesLine = emailMode === 'adhoc_note'
  ? `Athian team member: ${effectiveSenderName} <${effectiveSenderAddress}> (internal note — no external attendees)`
  : `${effectiveSenderName} <${effectiveSenderAddress}> (external); Athian contributor <${contributor}>`;

const claudeUserMessage = [
  `Meeting title: ${subject}`,
  `Date: ${date}`,
  `Attendees: ${attendeesLine}`,
  '',
  'Notes:',
  body
].join('\n');

return [{
  json: {
    messageId,
    subject,
    date,
    fromAddress,
    fromName,
    contributor,
    effectiveSenderName,
    effectiveSenderAddress,
    emailMode,     // 'forwarded' | 'adhoc_note' | 'external_inbound' — useful for debugging
    body,
    claudeUserMessage,
  }
}];
```

---

### [4] Build Raw Feedback Prompt

**Type:** Code node  
**Purpose:** Attach the Claude request body for the Raw Feedback extraction. The system prompt is **identical** to the Granola flow's Raw Feedback node — copy it verbatim. Only the model and max_tokens are set here; `claudeUserMessage` comes from node [3].

```javascript
const item = $input.item.json;

// ─── SYSTEM PROMPT ───────────────────────────────────────────────────────────
// THIS IS THE CANONICAL RAW FEEDBACK SYSTEM PROMPT.
// Keep in sync with the Granola workflow's equivalent node.
// If you change it here, change it there too (or extract to an n8n Variable).
const systemPrompt = "You are an extraction assistant for Athian's customer feedback intelligence system.\n\nYou will receive a structured block containing metadata and the full AI-generated notes from a Granola meeting. These notes are richer than a compressed summary — they include section headers, bullet points, and more specific detail. Direct verbatim quotes may be present.\n\n**Input format:**\n```\nMeeting title: {title}\nDate: {date}\nAttendees: {attendees with name and email}\n\nNotes:\n{full structured markdown notes}\n```\n\n**Fields to extract:**\n\n- `company` (string or null): The name of the customer or prospect company. Null if not mentioned or if this is an internal call. The meeting title often reveals the company name (e.g., \"Biofiltro / Athian working sesh\") — use it as a hint when the notes don't explicitly state the company.\n- `contact_name` (string or null): The primary external contact's name. Null if not mentioned.\n- `contact_roles` (string or null): The external contact's title or role, if mentioned. Null if not mentioned.\n- `call_type` (string): One of: `discovery`, `expansion`, `qbr`, `support`, `other`. Choose based on the primary purpose of the call.\n- `line_of_business` (string): Which Athian product line this feedback is about. One of: `sustainability-insets` (supply chain Scope 3 insetting programs where producers adopt emission-reduction protocols that generate carbon credits or verified reductions for CPG companies or retailers; specific protocols: Bovaer, AMMP, Rumensin, Carbon Intensity; goal is carbon credit generation or Scope 3 reduction credit), `sustainability-offsets` (carbon offset credit generation, trading, or offset programs — distinct from insetting), `compliance` (Athian's compliance product — for companies that need to comply with standards, regulations, policies, or responsible sourcing requirements that do NOT generate carbon credits; signals include: responsible sourcing programs, supplier auditing/reporting, labor/environmental certifications, regulatory or policy compliance frameworks), `unknown` (internal-only meeting or genuinely insufficient context).\n- `feedback_category` (string): The primary category of feedback. One of: `protocol`, `ux`, `pricing`, `verification`, `scope3_accounting`, `competitive`, `onboarding`, `other`. Choose the best single fit.\n- `theme_1` (string): The most prominent feedback or discussion topic. Required. Short noun phrase, 3–7 words.\n- `theme_2` (string or null): Second-most prominent theme, if present. Same format. Null if fewer than two distinct themes.\n- `theme_3` (string or null): Third distinct theme, if present. Null if fewer than three distinct themes.\n- `key_feedback` (string or null): The single most important thing the customer said or expressed. One to two sentences maximum. Prefer a direct verbatim quote when one appears in the notes; otherwise use a close paraphrase. Null if no meaningful customer statement or position is present.\n- `ai_summary` (string): A 1–2 sentence plain English summary of the overall feedback from the customer's perspective. Required — write one even if notes are sparse.\n- `sentiment` (string): The customer's overall expressed or implied attitude. One of: `positive`, `neutral`, `negative`, `mixed`, `blocking`. Use `blocking` only when the customer expressed something serious enough to threaten the relationship or deal (e.g., will churn, escalating to executive, deal on hold, contract pause threatened).\n- `priority` (string): How urgently this feedback warrants a response from Athian. One of: `high`, `medium`, `low`. Base on sentiment, blocking nature, and strategic importance of the company.\n- `next_steps` (string or null): A suggested follow-up action for the Athian team. Look for explicit next steps or action items in the notes — they often appear in a dedicated section or at the end of notes. Null if no clear action is warranted.\n\n**Rules:**\n- YOUR ENTIRE RESPONSE MUST BE VALID JSON. Do not wrap it in code fences. Do not add explanation before or after. If your response begins with anything other than `{`, it is wrong.\n- Themes must be distinct — do not restate the same idea across theme_1, theme_2, theme_3.\n- `call_type` reflects the primary purpose, not secondary topics.\n- `line_of_business`: determined by the customer's GOAL, not the product demonstrated. A demo of the insetting platform shown to a customer exploring a compliance use case is `compliance`. When the meeting is internal with no external attendees, use `unknown`. When the meeting clearly spans two LOBs, choose the dominant one. \"Insetting\" in the title is a hint, not a rule — if the customer's goal is compliance with standards or responsible sourcing requirements (not carbon credit generation), use `compliance`.\n- `feedback_category` classifies the primary theme. When in doubt, use `other`.\n- If the meeting has no external participants, return null for company, contact_name, contact_roles, and key_feedback; set call_type to `other`; extract themes and sentiment from what was discussed.\n- Do not infer company or contact details if not explicitly stated or clearly implied by the meeting title. Return null rather than guess.\n- `sentiment` and `priority` reflect the external party's position, not the outcome for Athian.\n- `blocking` sentiment always implies `high` priority.\n- The meeting title may be the only place the company name appears — it is a reliable source.\n- `key_feedback` should be a direct verbatim quote when one appears in the notes.\n\n**Fields set by the system — do not include in your output:**\n`row_id`, `source`, `source_url`, `granola_link`, `contributor`, `call_date`, `status`, `jira_ticket_ref`, `ingested_at`\n\n**Output format:**\n```\n{\n  \"company\": \"...\",\n  \"contact_name\": \"...\",\n  \"contact_roles\": \"...\",\n  \"call_type\": \"...\",\n  \"line_of_business\": \"...\",\n  \"feedback_category\": \"...\",\n  \"theme_1\": \"...\",\n  \"theme_2\": \"...\",\n  \"theme_3\": \"...\",\n  \"key_feedback\": \"...\",\n  \"ai_summary\": \"...\",\n  \"sentiment\": \"...\",\n  \"priority\": \"...\",\n  \"next_steps\": \"...\"\n}\n```";
// ─────────────────────────────────────────────────────────────────────────────

return [{
  json: {
    ...item,
    claude_request_body: {
      model: 'claude-haiku-4-5-20251001',
      max_tokens: 1024,
      system: systemPrompt,
      messages: [{ role: 'user', content: item.claudeUserMessage }]
    }
  }
}];
```

> **Refactor note:** Once the sub-workflow refactor is done (see below), this entire Code node is replaced by an Execute Workflow node. The prompt lives in the sub-workflow and is never duplicated.

---

### [5] HTTP Request → Claude (Raw Feedback)

**Type:** HTTP Request  
**Configuration:** Identical to the Granola workflow's equivalent node.

- Method: POST
- URL: `https://api.anthropic.com/v1/messages`
- Authentication: Header Auth
  - Name: `x-api-key`, Value: `{{ $vars.ANTHROPIC_API_KEY }}`
- Headers:
  - `anthropic-version`: `2023-06-01`
  - `content-type`: `application/json`
- Body: JSON — `{{ $json.claude_request_body }}`

**Pass-through context:** Use `$('Format Email for Claude').item.json` in downstream nodes (same pattern as Granola's HTTP Request → upstream context pattern noted in HANDOFF).

---

### [6] Parse Raw Feedback Response

**Type:** Code node  
**Purpose:** Extract and validate the JSON from Claude's response. Hard-error on non-JSON (same policy as Granola — no silent corrupt writes).

```javascript
const claudeResponse = $input.item.json;
const emailContext   = $('Format Email for Claude').item.json;

const raw = claudeResponse.content?.[0]?.text || '';

let extracted;
try {
  extracted = JSON.parse(raw);
} catch (e) {
  throw new Error(`Raw Feedback Claude response was not valid JSON.\nResponse: ${raw}`);
}

return [{
  json: {
    ...emailContext,
    extracted_raw_feedback: extracted,
  }
}];
```

---

### [7] Assemble Raw Feedback Row

**Type:** Code node  
**Purpose:** Merge extracted fields with system-set fields to produce the final sheet row. Mirrors the equivalent Granola node.

```javascript
const item = $input.item.json;
const ext  = item.extracted_raw_feedback;

const rowId      = $uuid();  // n8n built-in
const ingestedAt = new Date().toISOString();

// source_url for email = Message-ID (dedup key, same role as Slack permalink for Granola)
const sourceUrl = item.messageId || '';

return [{
  json: {
    row_id:              rowId,
    source:              'email',
    source_url:          sourceUrl,
    granola_link:        null,                // not applicable for email
    contributor:         item.contributor,  // Athian employee for forwarded/adhoc; inbox address for external inbound
    call_date:           item.date,
    company:             ext.company         ?? null,
    contact_name:        ext.contact_name    ?? null,
    contact_roles:       ext.contact_roles   ?? null,
    call_type:           ext.call_type       ?? null,
    line_of_business:    ext.line_of_business ?? null,
    feedback_category:   ext.feedback_category ?? null,
    theme_1:             ext.theme_1         ?? null,
    theme_2:             ext.theme_2         ?? null,
    theme_3:             ext.theme_3         ?? null,
    key_feedback:        ext.key_feedback    ?? null,
    ai_summary:          ext.ai_summary      ?? null,
    sentiment:           ext.sentiment       ?? null,
    priority:            ext.priority        ?? null,
    status:              'new',
    jira_ticket_ref:     null,
    next_steps:          ext.next_steps      ?? null,
    ingested_at:         ingestedAt,
  }
}];
```

---

### [8] Write to Raw Feedback sheet

**Type:** Google Sheets  
**Operation:** Append row  
**Configuration:** Identical to the Granola workflow's sheet write node — same spreadsheet ID, same tab name (`Raw Feedback`).

Map columns in the same order as the Granola node.

---

### [9] Build Problems Prompt

**Type:** Code node  
**Purpose:** Attach the Claude request body for Problems extraction. System prompt is **identical** to the Granola workflow's Problems node — copy verbatim. `claudeUserMessage` (the email body, already formatted in node [3]) is reused directly.

```javascript
const item = $input.item.json;

// ─── SYSTEM PROMPT ───────────────────────────────────────────────────────────
// THIS IS THE CANONICAL PROBLEMS SYSTEM PROMPT.
// Keep in sync with the Granola workflow's equivalent node.
// Refactor suggestion: move to n8n Variable PROMPT_PROBLEMS_SYSTEM.
const systemPromptLines = [
  'You are an Athian product intelligence assistant. Your job is to process a meeting transcript and extract structured discovery signal into a precise JSON format.',
  '',
  '## Context',
  '',
  'Athian is a protocol-native compliance execution platform. Every workflow maps to a five-step domain chain:',
  '',
  '  Protocol -> MonitoringPeriod -> VerificationEvent -> Claim -> AuditHandoff',
  '',
  '- Protocol: rules, thresholds, and evidence requirements for a standard',
  '- MonitoringPeriod: structured evidence collection window',
  '- VerificationEvent: third-party review -- site visit, desktop audit, remote assessment',
  '- Claim: serialized output -- certificate, opinion, verification statement',
  '- AuditHandoff: regulatory submission, registry filing, client delivery',
  '',
  'Athian serves three ICP segments:',
  '- CAB: Conformity Assessment Bodies -- ISO, organic, food safety, sustainability certifiers',
  '- VVB: GHG Validation/Verification Bodies -- firms accredited under Verra, CAR, ACR, Gold Standard',
  '- supplier_compliance: food and agriculture manufacturers running internal supplier compliance programs (the Supplier Compliance OS motion)',
  '- CPG_buyer: enterprise CPG brands (Nestle, Hershey) mandating compliance across their supplier networks',
  '- advisory_firm: accounting or advisory firms (Crowe, Sensiba) delivering compliance services to enterprise clients',
  '',
  'The config vs. code distinction is critical at Athian. A problem is config_only if it can be solved by encoding a new compliance standard into the existing domain model without changing platform code. It needs_new_code if it requires new schema abstractions or platform capabilities that do not currently exist. When in doubt, mark unclear and add notes -- this field is reviewed by the Principal Architect before any engineering decision is made.',
  '',
  '## Instructions',
  '',
  'Read the transcript carefully. Then produce a single JSON object conforming exactly to the schema below.',
  '',
  'Rules:',
  '- Use only what is in the transcript. Do not infer or fabricate.',
  '- For every field where the transcript provides no signal, use null for strings, [] for arrays, and false for booleans.',
  '- For enum fields, choose the closest match. If no match fits, use other where available or null.',
  '- problems must be an array -- extract each distinct pain, gap, or request as a separate object, even if raised by the same person.',
  '- problems[].id must follow the format: [YYYYMMDD]-[ACCOUNT_INITIALS_UPPERCASE]-[2-digit sequence]. Example: 20260629-NES-01, 20260629-NES-02.',
  '- evidence_quote must be a verbatim quote from the transcript, or null if no direct quote exists for that problem.',
  '- opportunities are inferred -- only include them if the transcript contains enough signal to support the inference. Mark confidence honestly.',
  '- belief_check items must each be a single plain-language sentence. Do not leave all three arrays empty unless the call contained genuinely no signal bearing on Athian\'s existing beliefs.',
  '- competitor_mentions includes any tool, vendor, internal system, or manual process named as a current or considered alternative.',
  '',
  '## Output schema',
  '',
  'Output only the JSON object. No preamble, no explanation, no markdown fences. The output must be valid JSON.',
  '',
  JSON.stringify({
    meeting_id: "",
    meeting_name: "",
    meeting_date: "",
    meeting_type: "cold_outreach | warm_intro | poc_working_session | design_partner | investor | internal | inbound_email",
    account_name: "",
    segment: "CAB | VVB | CPG_buyer | supplier_compliance | advisory_firm | other",
    participants: [
      { name: "", role: "", organization: "" }
    ],
    account_urgency: { stage: "active_budget | acknowledged_no_timeline | exploring | skeptical", evidence: "" },
    problems: [
      {
        id: "",
        type: "pain_point | feature_request | workflow_gap",
        domain_chain_location: "protocol | monitoring_period | verification_event | claim | audit_handoff | cross_cutting",
        description: "",
        current_workaround: "",
        evidence_quote: "",
        pain_severity: "high | medium | low",
        config_assessment: "config_only | needs_new_code | unclear",
        config_assessment_notes: ""
      }
    ],
    opportunities: [
      { description: "", source_problem_ids: [], confidence: "inferred | stated_explicitly" }
    ],
    risks: [
      { description: "", type: "competitive | technical | commercial | regulatory" }
    ],
    pull_signal: { present: "true | false", description: "", specificity: "named_capability | vague_interest | next_step_requested" },
    next_steps: [
      { description: "", owner: "", due_date: "" }
    ],
    belief_check: { confirms: [], complicates: [], contradicts: [] },
    competitor_mentions: [
      { name: "", context: "" }
    ],
    tags: { segment: [], domain_chain: [], theme: [] }
  }, null, 2)
];
const systemPrompt = systemPromptLines.join('\n');
// ─────────────────────────────────────────────────────────────────────────────

return [{
  json: {
    ...item,
    claude_request_body: {
      model: 'claude-sonnet-4-6',
      max_tokens: 8192,
      system: systemPrompt,
      messages: [{ role: 'user', content: item.claudeUserMessage }]
    }
  }
}];
```

> **Note on `meeting_type`:** Added `inbound_email` to the enum in the schema passed to Claude so it has an accurate value to choose for email-sourced records. The Granola flow's prompt does not include this value — add it there too for consistency, or leave it out (Claude will fall back to `other` for email records in the Granola flow, which is never triggered for email anyway).

---

### [10] HTTP Request → Claude (Problems)

**Type:** HTTP Request  
**Configuration:** Identical to node [5], same auth headers. This call uses Sonnet (higher token limit) — same as the Granola Problems node.

---

### [11] Parse Problems Response

**Type:** Code node  

```javascript
const claudeResponse = $input.item.json;
const emailContext   = $('Format Email for Claude').item.json;

const raw = claudeResponse.content?.[0]?.text || '';

let parsed;
try {
  parsed = JSON.parse(raw);
} catch (e) {
  throw new Error(`Problems Claude response was not valid JSON.\nResponse: ${raw}`);
}

// Attach email context for use in assembly
return [{
  json: {
    ...emailContext,
    problems_response: parsed,
  }
}];
```

---

### [12] Split Problems into Items

**Type:** Code node  
**Purpose:** Emit one item per problem so node [13] runs per-row (same pattern as Granola's SplitInBatches or equivalent).

```javascript
const item = $input.item.json;
const resp = item.problems_response;

const problems = resp.problems || [];

if (problems.length === 0) {
  // No problems extracted — end cleanly
  return [];
}

return problems.map(problem => ({
  json: {
    // top-level meeting context (repeated per row, same as Granola)
    meeting_date:              resp.meeting_date   || item.date,
    meeting_name:              resp.meeting_name   || item.subject,
    meeting_type:              resp.meeting_type   || 'inbound_email',
    account_name:              resp.account_name   || null,
    segment:                   resp.segment        || null,
    participants:              JSON.stringify(resp.participants || []),
    account_urgency_stage:     resp.account_urgency?.stage    || null,
    account_urgency_evidence:  resp.account_urgency?.evidence || null,
    // pull_signal (top-level)
    pull_signal_present:       resp.pull_signal?.present      || false,
    pull_signal_specificity:   resp.pull_signal?.specificity  || null,
    pull_signal_description:   resp.pull_signal?.description  || null,
    // competitor_mentions (top-level, stringified for sheet cell)
    competitor_mentions:       JSON.stringify(resp.competitor_mentions || []),
    // tags (top-level)
    tags_segment:              (resp.tags?.segment     || []).join(', '),
    tags_domain_chain:         (resp.tags?.domain_chain || []).join(', '),
    tags_theme:                (resp.tags?.theme        || []).join(', '),
    // belief check (top-level)
    belief_confirms:           (resp.belief_check?.confirms    || []).join(' | '),
    belief_complicates:        (resp.belief_check?.complicates || []).join(' | '),
    belief_contradicts:        (resp.belief_check?.contradicts || []).join(' | '),
    // this problem
    problem: problem,
    // email source context
    source_url:                item.messageId,
    ingested_at:               new Date().toISOString(),
  }
}));
```

---

### [13] Assemble Problem Row

**Type:** Code node  
**Purpose:** Flatten each problem object into the Problems sheet columns.

```javascript
const item    = $input.item.json;
const problem = item.problem;

// next_steps: join array if present
const nextSteps = Array.isArray(item.problems_response?.next_steps)
  ? (item.problems_response.next_steps || []).map(ns => ns.description).filter(Boolean).join(' | ')
  : null;

return [{
  json: {
    meeting_date:              item.meeting_date,
    meeting_name:              item.meeting_name,
    meeting_type:              item.meeting_type,
    account_name:              item.account_name,
    segment:                   item.segment,
    participants:              item.participants,
    account_urgency_stage:     item.account_urgency_stage,
    account_urgency_evidence:  item.account_urgency_evidence,
    problem_id:                problem.id,
    problem_type:              problem.type,
    domain_chain_location:     problem.domain_chain_location,
    description:               problem.description,
    current_workaround:        problem.current_workaround,
    evidence_quote:            problem.evidence_quote,
    pain_severity:             problem.pain_severity,
    config_assessment:         problem.config_assessment,
    config_assessment_notes:   problem.config_assessment_notes,
    pull_signal_present:       item.pull_signal_present,
    pull_signal_specificity:   item.pull_signal_specificity,
    pull_signal_description:   item.pull_signal_description,
    competitor_mentions:       item.competitor_mentions,
    tags_segment:              item.tags_segment,
    tags_domain_chain:         item.tags_domain_chain,
    tags_theme:                item.tags_theme,
    belief_confirms:           item.belief_confirms,
    belief_complicates:        item.belief_complicates,
    belief_contradicts:        item.belief_contradicts,
    next_steps:                nextSteps,
    register_candidate:        null,   // reviewed and set manually
    register_status:           null,   // reviewed and set manually
  }
}];
```

---

### [14] Write to Problems sheet

**Type:** Google Sheets  
**Operation:** Append row  
**Configuration:** Same spreadsheet ID as the Granola workflow, Problems tab.

Map columns in the same order as the Problems sheet header row.

---

## Shared Prompt Refactor (Recommended)

n8n Variable values are capped at 1000 characters — too short for these prompts. The correct n8n-native approach is **shared sub-workflows**.

### How it works

Create two new standalone n8n workflows:

| Sub-workflow | Trigger | Input | Output |
|---|---|---|---|
| `[Sub] Build Raw Feedback Prompt` | Execute Workflow trigger | `claudeUserMessage` (string) | `claude_request_body` (object) |
| `[Sub] Build Problems Prompt` | Execute Workflow trigger | `claudeUserMessage` (string) | `claude_request_body` (object) |

Each sub-workflow contains exactly one Code node — the prompt-builder — and a Respond to Webhook / Return node that emits `claude_request_body`.

In both the Granola workflow and the email workflow, replace nodes [4] and [9] (the prompt-builder Code nodes) with **Execute Workflow** nodes that call the relevant sub-workflow, passing `claudeUserMessage` and receiving `claude_request_body`.

### Sub-workflow structure

```
[Execute Workflow Trigger]  ← receives { claudeUserMessage }
        ↓
[Code: Build Prompt]        ← current Code node content, unchanged
        ↓
[Return]                    ← emits { claude_request_body }
```

### Execute Workflow node config (in parent)

- **Source:** Database
- **Workflow:** select the sub-workflow by name
- **Fields to send:** `claudeUserMessage` → `{{ $json.claudeUserMessage }}`
- **Wait for sub-workflow:** Yes

### Result

One edit to a sub-workflow prompt propagates to both the Granola and email flows immediately. No duplication, no character limit issue.

---

## Email Filtering Strategy Notes

**Inbox: `feedback-hub@athian.ag` (Google Group)**  
The Gmail Trigger monitors this dedicated Group inbox. All inbound email is presumed intentional — someone on the team forwarded or CC'd it deliberately. The filter in node [2] only needs to drop automated noise (bounces, autoreplies, noreply senders). No label rules or keyword filters needed.

---

## Open Questions / Next Steps

- [ ] **Problems sheet `source_url`:** Currently only the Raw Feedback sheet has `source_url`. Adding it (Message-ID) to Problem rows too would let you trace any problem back to the originating email. Low effort — worth adding now before data accumulates.
- [ ] **`inbound_email` meeting_type:** Confirm this as the canonical value for email-sourced records and add it to the Granola Problems prompt schema string for consistency (harmless — Granola flow never produces it).
- [ ] **n8n Variable refactor:** Do this before the email flow goes live so both flows share prompts from day one and drift is impossible.
- [ ] **Forwarded-email date extraction:** The regex in node [3] covers Gmail's standard forward format. Test with a real forwarded email — Outlook and other clients format the forwarded block differently (e.g., `From: X\nSent: ...` instead of `From: X\nDate: ...`). May need a second regex branch.
- [ ] **Outlook-style forwards:** Subject line pattern is `FW:` (not `Fwd:`). The current regex handles both (`/^fwd?:/i`), but validate with a real FW: email.
- [ ] **Test with all three modes** (forwarded, ad hoc note, external inbound) before enabling the trigger continuously.
