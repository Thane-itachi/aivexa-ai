# Aivexa AI — V1 Build Spec (Base44 Builder Prompt)

Scope: Chat + Files & Knowledge + Research + Memory/RAG + Free/Pro subscription with daily free-token limits. Dark, modern, premium UI.

## PAGES

- Landing page: hero ("Your AI workspace for research, knowledge, and building"), CTA to sign up, feature grid (Chat, Research, Files & Knowledge, Memory), clean footer.
- Dashboard: list of conversations + recent knowledge bases, quick-start prompt suggestions.
- Chat page (main screen): conversation sidebar, streaming message area, markdown rendering with code blocks + syntax highlighting, source citations shown under answers when research/files were used. Always show a token-budget meter (remaining tokens today) in the header when on the Free plan.
- Knowledge page: upload PDF/DOCX/TXT/CSV/code files, list uploaded documents, ask questions grounded in them.
- Memory page: view, search, edit, and delete saved facts Aivexa has learned about the user.
- Billing page: Free vs Pro plan cards, current plan, daily/monthly token usage, upgrade CTA.
- Settings: profile, theme toggle, danger zone.

## ENTITIES

- Conversation (title, last_message_at)
- Message (conversation_id, role, content, sources, tokens_used)
- KnowledgeBase (name)
- Document (knowledge_base_id, name, file_url, status: processing/ready/error, summary)
- DocumentChunk (document_id, content, embedding_ready)
- MemoryItem (content, category, source, confidence)
- Subscription (plan: free/pro, status: active/past_due/cancelled, started_at, provider, provider_subscription_id)
- DailyUsage (day_key: "YYYY-MM-DD", tokens_used, requests_count) — one record per user per day
- UsageRecord (metric: chat_tokens/research_tokens/document_tokens, quantity, occurred_at) — append-only ledger for metering and the usage dashboard

## SUBSCRIPTION PLANS + TOKEN LIMITS

Plans:

```text
FREE
  10,000 tokens per day (resets midnight, user's timezone)
  3 knowledge bases, 20 documents max
  5 research requests per day
  Standard model only
  Chat history capped at last 7 days

PRO — $12/month (or local-equivalent pricing)
  500,000 tokens per day, 10M per month
  Unlimited knowledge bases and documents
  100 research requests per day
  Faster/premium model access
  Full history + data export
  Priority support
```

Quota enforcement rules (server-side, never trusted from the client):

- Before every model call, check the user's plan and their DailyUsage record for today.
- If the user is out of daily tokens, DO NOT call the model. Return a friendly structured message: "You've used your 10,000 free tokens for today. They reset at midnight — or upgrade to Pro for 500k tokens/day." with an Upgrade button linking to the Billing page.
- If a message would exceed the remaining tokens mid-response, stop the stream cleanly, keep what was generated, and show the same upgrade message.
- Count tokens used per message (tokens_used on Message) and increment DailyUsage + append a UsageRecord for every call.
- Research requests additionally count against the 5/day free limit (requests_count on DailyUsage).
- Show remaining tokens today as a subtle meter in the chat header on Free, and full usage charts on the Billing page.
- Upgrades/downgrades take effect immediately; downgrades keep access until the end of the paid period.

## AI PERSONA / SYSTEM PROMPT (the "main brain")

Store the system prompt in ONE place (a `systemPrompt` constant or settings record) — never duplicated per page — and use it for every AI call.

The Aivexa system prompt must include, at minimum:

```text
You are Aivexa, an AI workspace assistant for research, knowledge, and building.
Your developer is Terdoo Jedidiah ("Jedidiah"). If asked who developed you or who
created Aivexa AI, answer: Jedidiah.
Be precise, cite your sources when using research or uploaded documents, and
never invent facts. Ground answers in the user's documents and memory when
relevant. Keep a warm, professional tone.
```

When the user asks "who developed you / who made you", the assistant answers Jedidiah. The name must never be hard-coded in page components; it lives in the system prompt only.

## FUNCTIONS / BACKEND

- chatCompletion: takes conversation_id + user message; first runs checkQuota (plan + DailyUsage) and returns the upgrade message instead of calling the model when the free limit is exhausted; otherwise retrieves relevant context (recent messages + matching document chunks + memory items) using vector search; calls the AI model with the Aivexa system prompt; streams the response; saves both messages; records tokens used.
- processDocument: on upload, extracts text, splits into chunks, stores chunks for retrieval. Enforce plan limits (Free: max 20 documents, 3 knowledge bases).
- researchAnswer: when the user's message is flagged as a research request, checks the 5/day free research limit, then performs web search, ranks sources, synthesizes an answer with citations.
- checkQuota: returns remaining tokens and research requests for today given the user's plan.
- getUsage: usage summary for the Billing page (tokens by day, research requests, documents stored).

## LOGIC

- Auth with email/password + Google.
- Row-level security: users only see their own conversations, files, memories, subscriptions, and usage records.
- Usage metering: every model call writes tokens_used to Message, increments DailyUsage, and appends a UsageRecord.
- New users default to the Free plan with a Subscription record created at signup.
- Responsive, polished design system: dark theme default, one accent color, consistent spacing and typography.

## How to use this

1. Go to https://app.base44.com and create a new app named "Aivexa AI".
2. Paste the PAGES / SUBSCRIPTION PLANS / AI PERSONA / ENTITIES / FUNCTIONS / LOGIC sections above as your first builder prompt.
3. Publish the app when the build looks right.
4. Later phases (agent runtime, tool registry, sandbox, automations) can be layered on in the same app. Payment processing (card capture) can be wired through the builder's payment integration after the app exists.

## Keep the backend modular

Because you've previously used builders such as Floot/Base44/Bolt, keep the architecture modular enough that critical services can eventually move out of the builder and into your own infrastructure.

## Roadmap note

This V1 scope is deliberately small: Chat, Files, RAG, Memory, Research, Model Gateway, plus Free/Pro plans with daily token quotas. Agents, tools, coding, sandbox, and the Builder are **not** in V1 — they land in V2-V5 per `11-training-and-model-roadmap.md`. The subsystem specs in docs 03-09 must be followed when those versions are built; do not let a builder improvise the Model Gateway, Agent Runtime, Tool Registry, permissions, or sandbox.
