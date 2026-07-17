# Dify Chatflow: ACR Ops Assistant — Technical Reference

This is the schema, prompt, and API contract for the Dify side of the ACR Framework. For click-by-click setup steps in the Dify UI, see [`DIFY_INTEGRATION_GUIDE.md`](DIFY_INTEGRATION_GUIDE.md) instead — this doc assumes the app already exists and documents exactly what's configured inside it.

## Flow

```
START → USER INPUT → GETOPSCASEDATA (Custom Tool) → LLM → ANSWER
```

Four nodes. No RAG, no vector store, no knowledge base. Every answer is grounded in a live API call made at query time.

---

## Custom Tool: `ops_case_data`

Added via **Integrations → Tools → Swagger API as Tool** in Dify. The tool's `operationId` (`getOpsCaseData`) is what shows up as the node name `GETOPSCASEDATA` on the chatflow canvas.

### OpenAPI/Swagger schema

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

**Authorization method:** None. (See Known Limitations in the main README — this is a demo-only setting, not a production one.)

### Expected response shape from the tool

This is the actual, verified shape returned by the n8n webhook on the current 100-row dataset:

```json
{
  "total": 100,
  "p1_count": 19,
  "p2_count": 30,
  "p3_count": 51,
  "p1_cases": [
    {
      "case_type": "AML-Compliance",
      "priority": "P1",
      "entity": "Nexus Trading LLC",
      "segment": "SME / Small Business",
      "summary": "LEI verification is failing even though it is active on the GLEIF website.",
      "ai_remediation_summary": "AML/KYC Ops"
    }
  ],
  "p2_cases": [ "..." ],
  "p3_cases": [ "..." ]
}
```

---

## How the webhook call is actually made

This is the mechanical detail worth being explicit about, since it's the part most likely to break silently if replicated incorrectly:

1. A user types a question into the Dify chat UI (e.g. *"Could you give me a summary of the escalated P1 cases?"*)
2. Dify's LLM node, seeing the tool is available and relevant, issues a **GET request** to `https://<your-instance>.app.n8n.cloud/webhook/ops-query` — no request body, no query parameters, no auth header (matching the "None" auth setting above)
3. The n8n **Webhook node** receives this and triggers Workflow 2 (`ops-query-api.json`): it reads the live Google Sheet, computes `p1_count`/`p2_count`/`p3_count` and the three case arrays fresh, and returns them as JSON via **Respond to Webhook**
4. Dify receives that JSON as the tool's output and injects it into the LLM's context via `{{#getOpsCaseData.text#}}` — the raw JSON string, not a pre-parsed object
5. The LLM reasons over that raw JSON directly to answer the user's question in natural language

**Nothing is cached at any point in this chain.** Every question triggers a fresh webhook call, which triggers a fresh Sheets read. This is the specific design choice that makes the assistant trustworthy for exact counts — the alternative (loading a snapshot into a Dify knowledge base) was tried first and failed, because semantic search over static text can't reliably answer "how many."

**One method-matching detail that broke this in practice:** the n8n Webhook node and the Dify tool schema both need to agree on **GET**, not just have a URL that happens to respond. A mismatch here (e.g. Webhook set to POST while the schema says GET) fails silently from Dify's side — the URL "works" in isolation but the integration doesn't, and the failure looks unrelated to its actual cause.

---

## LLM node configuration

- **Model:** gpt-5.5 (Chat)
- **Context:** `{{#getOpsCaseData.text#}}` — the raw JSON string returned by the tool call, injected directly into the system prompt
- **Memory:** built-in, window size 10 (multi-turn conversation support — e.g. a follow-up like *"what about the P2s?"* resolves correctly against the prior turn)

## System prompt

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

---

## Why Dify is still needed here — not just n8n's own chatbot

n8n has an AI Agent node and can technically hold a conversation. So it's a fair question: why introduce a second platform at all, instead of exposing n8n's own agent as the chat interface?

Three concrete, technical reasons this build keeps them separate:

1. **Conversation state is not n8n's job.** n8n's AI Agent node is designed for single-shot or workflow-triggered reasoning within one execution. It has no persistent, per-user multi-turn memory across separate webhook calls — every incoming request is its own execution with no built-in concept of "the previous message in this conversation." Dify's chatflow has session-scoped memory (window size 10 here) as a first-class feature, not something to bolt on.
2. **Decoupling the contract from the logic.** Dify's tool integration only needs to know the response *shape* (`total`, `p1_count`, `p1_cases`, etc.) — never *how* those numbers are computed. If the classification rule changes tomorrow (a new category, a different SLA threshold), only the n8n Code node changes. The Dify side needs zero updates, because the contract between them never touched the business logic.
3. **Each tool does the job it's actually good at.** n8n is excellent at deterministic branching and orchestration; it's not built for natural-language reasoning over ambiguous follow-up questions. Dify is built for exactly that, and does nothing else here — it has no independent business logic of its own, it's a thin, replaceable conversational layer over a single API call.

In short: this isn't "Dify instead of n8n" — it's Dify doing the one job n8n isn't designed for, while n8n keeps every source-of-truth decision it's already good at.

