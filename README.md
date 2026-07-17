# ACR Framework — Ask, Check, Recommend

An AI-powered escalation engine for commercial banking client feedback. Built with **n8n** (classification, orchestration, SLA-aware alerting) and **Dify** (conversational query layer via live tool-calling into an n8n webhook).

> Personal demo project. All sample data is synthetic — no client or employer data involved.

📄 Full write-up: [`docs/ACR_Framework_Article_Final.md`](docs/ACR_Framework_Article_Final.md)
🤖 Dify setup (step-by-step): [`dify/DIFY_INTEGRATION_GUIDE.md`](dify/DIFY_INTEGRATION_GUIDE.md)
🔧 Dify technical reference (schema, prompt, API): [`dify/acr-ops-assistant-config.md`](dify/acr-ops-assistant-config.md)

---

## Architecture

```
Google Sheets (raw feedback, 100-row synthetic dataset)
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│  WORKFLOW 1: acr-classification-pipeline.json                │
│                                                                │
│  Manual Trigger ─┐                                            │
│  Schedule Trigger┴─► Get row(s) in sheet                      │
│                              │                                │
│                              ▼                                │
│                        AI Agent (classify)                    │
│                     [Model: OpenAI Chat + Structured           │
│                      Output Parser — category + segment,      │
│                      feedback_id passed through verbatim]     │
│                              │                                │
│                              ▼                                │
│                    Code: normalize segment/category,          │
│                    derive priority strictly from category     │
│                    (AML-Compliance→P1, Tech-Infra→P2, else P3)│
│                              │                                │
│                              ▼                                │
│                    If (category = AML-Compliance)             │
│                    ├─ True (19) ──► Code: tag P1               │
│                    └─ False (81) ─► If1 (category=Tech-Infra)   │
│                                 ├─ True (30) ──► Code1: P2      │
│                                 └─ False (51) ─► Code2: P3      │
│                              │                                │
│                              ▼                                │
│                           Merge (3 inputs → 1)                │
│                              │                                │
│                              ▼                                │
│                    Append or Update row in sheet               │
│                    (matched on feedback_id — prevents          │
│                     re-run duplication; single source          │
│                     of truth, all 100 rows)                    │
│                              │                                │
│                              ▼                                │
│                    If2 (priority != P3)                        │
│                    ├─ True (49) ──► Code: build SLA-tagged     │
│                    │                HTML digest                │
│                    │                    │                     │
│                    │                    ▼                     │
│                    │              Gmail: Send digest           │
│                    │              (grouped P1/P2, SLA labels)  │
│                    └─ False (51) ─► No Operation (logged only) │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  WORKFLOW 2: ops-query-api.json                               │
│                                                                │
│  Webhook (GET /webhook/ops-query, no auth)                    │
│         │                                                     │
│         ▼                                                     │
│  Get row(s) in sheet (reads current live data)                 │
│         │                                                     │
│         ▼                                                     │
│  Code: compute counts + filter by case_type                    │
│  (p1_count, p2_count, p3_count, p1_cases[], p2_cases[],        │
│   p3_cases[], total)                                            │
│         │                                                     │
│         ▼                                                     │
│  Respond to Webhook (returns JSON)                              │
└────────────────────────────────────────────────────────────┘
        │
        ▼  (Dify calls this endpoint as a Custom Tool, on demand)
┌────────────────────────────────────────────────────────────┐
│  Dify Chatflow: ACR Ops Assistant                              │
│                                                                │
│  START → USER INPUT                                            │
│              │                                                │
│              ▼                                                │
│  Custom Tool: ops_case_data                                     │
│  (Swagger/OpenAPI schema, GET, calls Workflow 2's                │
│   production webhook URL, no auth)                              │
│              │                                                │
│              ▼                                                │
│  LLM (gpt-5.5, Context = tool output, Memory ON, window=10)     │
│              │                                                │
│              ▼                                                │
│  ANSWER (natural-language response, grounded in live data)      │
└────────────────────────────────────────────────────────────┘
        │
        ▼
   Ops Manager (conversational, plain-English queries)
```

---

## Live Result

Running the classification pipeline against the 100-row synthetic dataset produces:

| Category | Count | Priority |
|---|---|---|
| AML-Compliance | 19 | P1 |
| Tech-Infrastructure | 30 | P2 |
| General Service | 51 | P3 |
| **Total** | **100** | — |

**49 cases (P1 + P2) trigger the automated escalation digest.** The math reconciles exactly at every branch of the workflow — no items lost, no items duplicated.

---

## Workflow 1: Classification & Alerting Pipeline

**File:** `n8n-workflows/acr-classification-pipeline.json`

### Nodes, in order

| Node | Type | Purpose |
|---|---|---|
| Manual Trigger | `When clicking 'Execute workflow'` | Demo/on-demand execution |
| Schedule Trigger *(deactivated)* | Schedule (30 min interval) | Documented production cadence — inactive in this repo's demo state |
| Get row(s) in sheet | Google Sheets → Get Row(s) | Reads raw feedback (`Feedback_ID`, `User_Segment`, `Source_Channel`, `Raw_Text`) |
| AI Agent | AI Agent + OpenAI Chat Model + Structured Output Parser | Classifies each row into `category`, passes `feedback_id` through verbatim, generates `issue_summary`, `recommended_action` |
| Code (normalize) | Code (JS) | Normalizes any near-match `segment`/`category` values the model returns (e.g. "Corporate" → "Enterprise / Large Corporate"), derives `priority` strictly from `category` by rule — never trusted from the model directly |
| If / If1 | If | Branches on normalized `category`: AML-Compliance → true; else check Tech-Infrastructure |
| Code in JavaScript / 1 / 2 | Code (JS) | Tags each branch into a consistent flat `output` object |
| Merge | Merge (3 inputs) | Recombines all three branches into one stream (100 items) |
| Append or Update row in sheet | Google Sheets → Append or Update Row | Writes every case to the single source-of-truth sheet, matched on `feedback_id` — re-running the workflow updates existing rows instead of duplicating them |
| If2 | If | Filters `priority != P3` → routes P1/P2 to alerting, P3 to no-op |
| Code in JavaScript3 | Code (JS) | Builds the SLA-tagged, category-grouped HTML email body |
| Send a message | Gmail | Sends the digest email |
| No Operation, do nothing | NoOp | Terminates the P3 (already-logged) branch |

### Key logic: SLA tagging (Code in JavaScript3, excerpt)

```javascript
rows.forEach(d => {
  // Simulated ingestion age — source data has no real timestamp
  d.simulated_age_hours = Math.floor(Math.random() * 60) + 1;
  const warning = d.warning_threshold_hours || 24;
  const critical = d.critical_threshold_hours || 48;
  d.sla_status = d.simulated_age_hours >= critical ? "🔴 CRITICAL - Breach imminent"
               : d.simulated_age_hours >= warning ? "🟠 WARNING - Approaching deadline"
               : "🟢 On track";
});

const p1 = rows.filter(d => (d.case_type || '').trim() === 'AML-Compliance');
const p2 = rows.filter(d => (d.case_type || '').trim() === 'Tech-Infrastructure');
```

### Key logic: deterministic priority (Code node, excerpt)

```javascript
function normalizeCategory(val) {
  const v = (val || '').toLowerCase();
  if (v.includes('aml') || v.includes('compliance')) return 'AML-Compliance';
  if (v.includes('tech') || v.includes('infra')) return 'Tech-Infrastructure';
  return 'General Service';
}

const category = normalizeCategory(item.category);
const priority = category === 'AML-Compliance' ? 'P1'
              : category === 'Tech-Infrastructure' ? 'P2'
              : 'P3';
```

> Priority is never asked of the model as an independent field. It's computed once, deterministically, from `category` — this removed a recurring class of bug where an AI-derived priority field silently disagreed with its own category. See [Known Issues & Fixes](#known-issues--fixes).

---

## Workflow 2: Ops Query API

**File:** `n8n-workflows/ops-query-api.json`

**Webhook endpoint:** `GET https://<your-instance>.app.n8n.cloud/webhook/ops-query`
No authentication. Must be **Published/Active** for the production URL to respond (the Test URL is single-use, for debugging only).

### Response shape

```json
{
  "total": 100,
  "p1_count": 19,
  "p2_count": 30,
  "p3_count": 51,
  "p1_cases": [ { "case_type": "...", "priority": "...", "entity": "...", "segment": "...", "summary": "...", "ai_remediation_summary": "..." }, ... ],
  "p2_cases": [ ... ],
  "p3_cases": [ ... ]
}
```

### Code node logic

```javascript
const rows = $input.all().map(i => i.json);

const p1 = rows.filter(r => (r.case_type || '').trim() === 'AML-Compliance');
const p2 = rows.filter(r => (r.case_type || '').trim() === 'Tech-Infrastructure');
const p3 = rows.filter(r => (r.case_type || '').trim().startsWith('General Service'));

return [{
  json: {
    total: rows.length,
    p1_count: p1.length,
    p2_count: p2.length,
    p3_count: p3.length,
    p1_cases: p1,
    p2_cases: p2,
    p3_cases: p3
  }
}];
```

**Why a webhook and not a direct Sheets-to-Dify integration:** this endpoint is the single place all business logic (category-to-priority mapping, filtering) lives. Dify never re-implements that logic — it just calls the endpoint and gets a pre-computed, correct answer. If business rules change, only this Code node needs updating. See [`dify/acr-ops-assistant-config.md`](dify/acr-ops-assistant-config.md) for exactly how Dify calls this endpoint and why Dify sits on top of it rather than n8n serving the chat layer itself.

---

## Dify Chatflow: ACR Ops Assistant

Full setup steps and technical reference have been split into two dedicated docs, since one audience wants to *replicate the integration* and the other wants the *exact schema/prompt/code*:

- **[`dify/DIFY_INTEGRATION_GUIDE.md`](dify/DIFY_INTEGRATION_GUIDE.md)** — functional, click-by-click walkthrough of building this in the Dify UI (Swagger tool import, chatflow wiring, publishing, testing)
- **[`dify/acr-ops-assistant-config.md`](dify/acr-ops-assistant-config.md)** — the OpenAPI/Swagger schema, LLM configuration, system prompt, and the exact call/response contract between Dify and the n8n webhook

**Quick summary:** `START → USER INPUT → ops_case_data (Custom Tool, GET) → LLM (gpt-5.5) → ANSWER`. The Custom Tool calls the n8n webhook above on every user question; nothing is cached or pre-loaded, so answers are always grounded in current sheet data.

---

## Repo structure

```
/
├── README.md                              ← this file
├── docs/
│   ├── ACR_Framework_Article_Final.md     ← full write-up (business-first framing)
│   ├── ACR_Framework_ShortPost.md         ← condensed LinkedIn post
│   ├── ACR_Framework_LinkedIn_Writeup_v2.md ← alternate long-form framing
│   └── screenshots/                       ← build + demo screenshots
├── n8n-workflows/
│   ├── acr-classification-pipeline.json   ← exported Workflow 1
│   └── ops-query-api.json                 ← exported Workflow 2 (webhook)
├── dify/
│   ├── DIFY_INTEGRATION_GUIDE.md          ← functional setup steps
│   └── acr-ops-assistant-config.md        ← technical reference (schema, prompt, API)
└── sample-data/
    └── sample-feedback.csv                ← synthetic input dataset (100 rows)
```

**To export the n8n workflows:** open each workflow → workflow menu (`...`) → **Download** → save the `.json` file into `n8n-workflows/`.

---

## Known Issues & Fixes

Documented here deliberately — these were real bugs hit during the build, and how each was actually resolved, not just described:

| Issue | Root cause | Fix |
|---|---|---|
| **Item count inflated (100 → 105+)** | Retry-on-failure and repeated manual test runs appended duplicate rows to an append-only sheet | Switched "Append row in sheet" to **Append or Update**, matched on `feedback_id` |
| **Priority drift** (category said Tech-Infrastructure, priority said P1) | AI Agent was asked to independently generate `priority` alongside `category` — the two could disagree even under schema constraints | Removed `priority` from the model's output entirely; it's now computed deterministically from `category` in a Code node |
| **Enum mismatch failures** (`"segment": "Enterprise"` instead of the full string) | Model returned a shortened, close-but-not-exact value against a strict JSON Schema enum | Loosened `segment`/`category` to plain strings with descriptions; added a normalization step in the Code node instead of relying on exact model wording |
| **`feedback_id` missing downstream** | The AI Agent's prompt only sent `Raw_Text` to the model — `Feedback_ID` was never in scope to be returned | Prompt now sends `Feedback_ID` explicitly; schema requires `feedback_id` in the output; system message instructs the model to copy it unchanged |
| **Broken `If` branch (100% landing in False)** | An `If` node still referenced `$json.output.category`, a path that no longer existed after an earlier flattening step | Corrected the field reference to `$json.category` |
| **Digest email lost its P1/P2 table** | "Append or Update row in sheet" had its field mappings (`case_type`, `entity`, `segment`, etc.) left blank, so the sheet stored empty values | Mapped each field to its actual upstream expression (e.g. `case_type` → `{{ $json.category }}`) |

**Simulated SLA timestamps:** the source dataset has no real ingestion timestamp; `simulated_age_hours` is randomly generated for demo purposes. A production version would use a real `created_at` field.

**No webhook authentication:** the Ops Query API endpoint (Workflow 2) has no auth — acceptable for a local demo, not for any real deployment. Add header-based auth in the n8n Webhook node and matching auth config in the Dify Custom Tool before exposing publicly.

---

## License

Personal / educational use. No proprietary or client data included.

