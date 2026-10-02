# Aivexa AI — V1 Build Spec (Base44 Builder Prompt)

Scope: Chat + Files & Knowledge + Research + Memory/RAG. Dark, modern, premium UI.

## PAGES

- Landing page: hero ("Your AI workspace for research, knowledge, and building"), CTA to sign up, feature grid (Chat, Research, Files & Knowledge, Memory), clean footer.
- Dashboard: list of conversations + recent knowledge bases, quick-start prompt suggestions.
- Chat page (main screen): conversation sidebar, streaming message area, markdown rendering with code blocks + syntax highlighting, source citations shown under answers when research/files were used.
- Knowledge page: upload PDF/DOCX/TXT/CSV/code files, list uploaded documents, ask questions grounded in them.
- Memory page: view, search, edit, and delete saved facts Aivexa has learned about the user.
- Settings: profile, theme toggle, danger zone.

## ENTITIES

- Conversation (title, last_message_at)
- Message (conversation_id, role, content, sources, tokens_used)
- KnowledgeBase (name)
- Document (knowledge_base_id, name, file_url, status: processing/ready/error, summary)
- DocumentChunk (document_id, content, embedding_ready)
- MemoryItem (content, category, source, confidence)

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

- chatCompletion: takes conversation_id + user message; retrieves relevant context (recent messages + matching document chunks + memory items) using vector search; calls the AI model with the Aivexa system prompt (see AI PERSONA above); streams the response; saves both messages.
- processDocument: on upload, extracts text, splits into chunks, stores chunks for retrieval.
- researchAnswer: when the user's message is flagged as a research request, performs web search, ranks sources, synthesizes an answer with citations.

## LOGIC

- Auth with email/password + Google.
- Row-level security: users only see their own conversations, files, and memories.
- Usage metering: store token usage per message.
- Responsive, polished design system: dark theme default, one accent color, consistent spacing and typography.

## How to use this

1. Go to https://app.base44.com and create a new app named "Aivexa AI".
2. Paste the PAGES / AI PERSONA / ENTITIES / FUNCTIONS / LOGIC sections above as your first builder prompt.
3. Publish the app when the build looks right.
4. Later phases (agent runtime, tool registry, sandbox, automations) can be layered on in the same app.

## Keep the backend modular

Because you've previously used builders such as Floot/Base44/Bolt, keep the architecture modular enough that critical services can eventually move out of the builder and into your own infrastructure.

## Roadmap note

This V1 scope is deliberately small: Chat, Files, RAG, Memory, Research, Model Gateway. Agents, tools, coding, sandbox, and the Builder are **not** in V1 — they land in V2-V5 per `11-training-and-model-roadmap.md`. The subsystem specs in docs 03-09 must be followed when those versions are built; do not let a builder improvise the Model Gateway, Agent Runtime, Tool Registry, permissions, or sandbox.
