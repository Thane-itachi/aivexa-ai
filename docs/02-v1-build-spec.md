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

## FUNCTIONS / BACKEND

- chatCompletion: takes conversation_id + user message; retrieves relevant context (recent messages + matching document chunks + memory items) using vector search; calls the AI model with a strong system prompt for the Aivexa assistant persona; streams the response; saves both messages.
- processDocument: on upload, extracts text, splits into chunks, stores chunks for retrieval.
- researchAnswer: when the user's message is flagged as a research request, performs web search, ranks sources, synthesizes an answer with citations.

## LOGIC

- Auth with email/password + Google.
- Row-level security: users only see their own conversations, files, and memories.
- Usage metering: store token usage per message.
- Responsive, polished design system: dark theme default, one accent color, consistent spacing and typography.

## How to use this

1. Go to https://app.base44.com and create a new app named "Aivexa AI".
2. Paste the PAGES / ENTITIES / FUNCTIONS / LOGIC sections above as your first builder prompt.
3. Publish the app when the build looks right.
4. Later phases (agent runtime, tool registry, sandbox, automations) can be layered on in the same app.

## Keep the backend modular

Because you've previously used builders such as Floot/Base44/Bolt, keep the architecture modular enough that critical services can eventually move out of the builder and into your own infrastructure.
