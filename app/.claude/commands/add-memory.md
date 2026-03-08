---
description: Add new knowledge or memory source to the AI agent
---

Add memory/knowledge: $ARGUMENTS

1. Determine the type:
   a) Static knowledge: create a markdown file in src/agent/knowledge/ and embed it in the system prompt
   b) Dynamic knowledge (RAG): create embeddings and store in Supabase pgvector
   c) Conversation memory: extend the existing conversation buffer

2. For static knowledge:
   - Create src/agent/knowledge/[topic].md with the domain knowledge
   - Update the system prompt to reference this knowledge
   - Keep under 2000 tokens to fit in context

3. For RAG knowledge:
   - Create a seed script: scripts/seed-knowledge.ts
   - Chunk the content into ~500 token segments
   - Generate embeddings (use Anthropic or OpenAI embeddings API)
   - Store in Supabase pgvector table (agent_knowledge)
   - Update the agent loop to: query relevant knowledge before responding
   - Add similarity search function in src/agent/memory/retriever.ts

4. For conversation memory:
   - Extend src/agent/memory/ to store/retrieve conversation summaries
   - Add periodic summarization of long conversations
   - Store summaries in Supabase for cross-session persistence

5. Test: ask the agent something that requires this new knowledge
6. Verify npm run dev has no errors
