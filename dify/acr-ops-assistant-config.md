Dify Chatflow: ACR Ops Assistant

Flow

START → USER INPUT → ops_case_data (Custom Tool) → LLM → ANSWER

Custom Tool schema (Swagger/OpenAPI)

Added via Integrations → Tools → Swagger API as Tool in Dify.

yamlopenapi: 3.0.0
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

Authorization method: None.

LLM node configuration


Model: gpt-5.5 (Chat)
Context: {{#getOpsCaseData.text#}} — the raw JSON string returned by the tool call, injected directly into the system prompt
Memory: built-in, window size 10 (multi-turn conversation support)


System prompt

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

Why Dify over extending n8n's own chat capability

n8n has no native multi-turn conversational memory or natural-language reasoning over follow-up questions. Rather than building that from scratch in a workflow tool, Dify owns the conversational layer exclusively, while all computation and business logic stays in n8n (the Ops Query API workflow). This keeps the two systems decoupled — Dify's tool contract only needs the response shape below to stay stable; it never needs to know how the counts were computed.

Expected response shape from the tool

json{
  "total": 105,
  "p1_count": 20,
  "p2_count": 39,
  "p3_count": 46,
  "p1_cases": [ { "case_type": "...", "priority": "...", "entity": "...", "segment": "...", "summary": "...", "ai_remediation_summary": "..." } ],
  "p2_cases": [ "..." ],
  "p3_cases": [ "..." ]
}
