# ACR Framework — Ask, Check, Recommend

An AI-powered escalation engine for commercial banking client feedback. Built with **n8n** (classification, orchestration, SLA-aware alerting) and **Dify** (conversational query layer via live tool-calling into a n8n webhook).

> Personal demo project. All sample data is synthetic — no client or employer data involved.

📄 Full write-up: [`docs/ACR_Framework_Article_Final.md`](docs/ACR_Framework_Article_Final.md)

---

## Architecture

```
Google Sheets (raw feedback, 105-row synthetic dataset)
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
│                      Output Parser — enforces category enum]  │
│                              │                                │
│                              ▼                                │
│                    If (category = AML-Compliance)             │
│                    ├─ True ──► Code: tag priority=P1           │
│                    └─ False ─► If1 (category = Tech-Infra)     │
│                                 ├─ True ──► Code1: priority=P2  │
│                                 └─ False ─► Code2: priority=P3  │
│                              │                                │
│                              ▼                                │
│                           Merge (3 inputs → 1)                │
│                              │                                │
│                              ▼                                │
│                    Append row in sheet                        │
│                    (single source of truth — all 105 rows,    │
│                     mapped fields: case_type, priority,       │
│                     entity, segment, summary,                 │
│                     ai_remediation_summary, status)            │
│                              │                                │
│                              ▼                                │
│                    If2 (priority != P3)                        │
│                    ├─ True (59) ──► Code: build SLA-tagged     │
│                    │                HTML digest                │
│                    │                    │                     │
│                    │                    ▼                     │
│                    │              Gmail: Send digest           │
│                    │              (grouped P1/P2, SLA labels)  │
│                    └─ False (46) ─► No Operation (logged only) │
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
        ▼
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

## Workflow 1: Classification & Alerting Pipeline

**File:** `n8n-workflows/acr-classification-pipeline.json`

### Nodes, in order

| Node | Type | Purpose |
|---|---|---|
| Manual Trigger | `When clicking 'Execute workflow'` | Demo/on-demand execution |
| Schedule Trigger *(deactivated)* | Schedule (30 min interval) | Documented production cadence — inactive in this repo's demo state |
| Get row(s) in sheet | Google Sheets → Get Row(s) | Reads raw feedback (`Feedback_ID`, `User_Segment`, `Source_Channel`, `Raw_Text`) |
| AI Agent | AI Agent + OpenAI Chat Model + Structured Output Parser | Classifies each row into `category`, generates `priority`, `issue_summary`, `recommended_action` |
| If / If1 | If | Branches on `category`: AML-Compliance → true; else check Tech-Infrastructure |
| Code / Code in JavaScript1 / Code in JavaScript2 | Code (JS) | Tags `priority` (P1/P2/P3) and normalizes each branch into a consistent flat `output` object |
| Merge | Merge (3 inputs) | Recombines all three branches into one stream (105 items) |
| Append row in sheet | Google Sheets → Append Row | Writes every case to the single source-of-truth sheet |
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

> **Important:** priority grouping for the digest is done on `case_type`, not the AI Agent's own `priority` field. See [Known Limitations](#known-limitations) — the two fields can drift.

---

## Workflow 2: Ops Query API

**File:** `n8n-workflows/ops-query-api.json`

**Webhook endpoint:** `GET https://<your-instance>.app.n8n.cloud/webhook/ops-query`
No authentication. Must be **Published/Active** for the production URL to respond (the Test URL is single-use, for debugging only).

### Response shape

```json
{
  "total": 105,
  "p1_count": 20,
  "p2_count": 39,
  "p3_count": 46,
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

**Why a webhook and not a direct Sheets-to-Dify integration:** this endpoint is the single place all business logic (category-to-priority mapping, filtering) lives. Dify never re-implements that logic — it just calls the endpoint and gets a pre-computed, correct answer. If business rules change, only this Code node needs updating.

---

## Dify Chatflow: ACR Ops Assistant

**File:** `dify/acr-ops-assistant-config.md`

### Flow
`START → USER INPUT → ops_case_data (Custom Tool) → LLM → ANSWER`

### Custom Tool schema (Swagger/OpenAPI, added via Integrations → Tools → Swagger API as Tool)

```yaml
openapi: 3.0.0
info:
  title: Ops Case Data API
  version: 1.0.0
servers:
  - url: https://<your-instance>.app.n8n.cloud
paths:
  /webhook/ops-query:
    get:
      operationId: getOpsCaseData
      summary: Get live case counts and details for P1, P2, and P3 client feedback cases
      description: Returns total case count and breakdown by priority (P1=AML-Compliance, P2=Tech-Infrastructure, P3=General Service), including entity, segment, summary, and recommended action for each case.
      responses:
        '200':
          description: Successful response with case data
          content:
            application/json:
              schema:
                type: object
```
Authorization method: **None**.

### LLM node configuration
- **Model:** gpt-5.5 (Chat)
- **Context:** `{{#getOpsCaseData.text#}}` — the raw JSON string returned by the tool call, injected directly into the system prompt
- **Memory:** built-in, window size 10 (multi-turn conversation support)

### System prompt

```
You are an operations assistant for a commercial banking client feedback system.

You have access to live, real-time case data provided below:
{{#context#}}

The data includes fields: case_type, priority, entity, segment, summary, ai_remediation_summary.

Answer questions about case counts, priorities, SLA status, and specific cases
based only on this data. If asked something not answerable from the data,
say so clearly rather than guessing.

When asked about case counts, priorities, or specific case details, always
call the "ops_case_data" tool to fetch current live data rather than guessing.
Use the exact numbers and details returned by the tool in your answer.
```

### Why Dify over extending n8n's own chat capability
n8n has no native multi-turn conversational memory or natural-language reasoning over follow-up questions. Rather than building that from scratch in a workflow tool, Dify owns the conversational layer exclusively, while all computation and business logic stays in n8n (Workflow 2). This keeps the two systems decoupled — Dify's tool contract only needs the response shape above to stay stable; it never needs to know *how* the counts were computed.

---

## Repo structure

```
/
├── README.md                              ← this file
├── docs/
│   ├── ACR_Framework_Article_Final.md     ← full write-up (business-first framing)
│   ├── ACR_Framework_ShortPost.md         ← condensed LinkedIn post
│   └── screenshots/                       ← build + demo screenshots
├── n8n-workflows/
│   ├── acr-classification-pipeline.json   ← exported Workflow 1
│   └── ops-query-api.json                 ← exported Workflow 2 (webhook)
├── dify/
│   └── acr-ops-assistant-config.md        ← system prompt + tool schema (above)
└── sample-data/
    └── sample-feedback.csv                ← synthetic input dataset
```

**To export the n8n workflows:** open each workflow → workflow menu (`...`) → **Download** → save the `.json` file into `n8n-workflows/`.

---

## Known Limitations

- **Priority drift:** the AI Agent's own `priority` field can diverge from `category` even under structured output constraints (e.g., an AML-Compliance case occasionally tagged P2 instead of P1). All downstream logic (digest grouping, Ops Query API) filters on **`case_type`**, not `priority`, to route around this. The underlying classifier prompt would need further constraint-tuning to fully resolve at the source.
- **Simulated SLA timestamps:** the source dataset has no real ingestion timestamp; `simulated_age_hours` is randomly generated for demo purposes. A production version would use a real `created_at` field.
- **Append-only writes:** Workflow 1 uses pure append, which caused significant row duplication during iterative testing (recovered via manual dedupe on `case_type`+`summary`). Production use should implement an upsert/update pattern or a dedup key (e.g., `Feedback_ID`) per run.
- **No webhook authentication:** the Ops Query API endpoint (Workflow 2) has no auth — acceptable for a local demo, not for any real deployment. Add header-based auth in the n8n Webhook node and matching auth config in the Dify Custom Tool before exposing publicly.

---

## License

Personal / educational use. No proprietary or client data included.
