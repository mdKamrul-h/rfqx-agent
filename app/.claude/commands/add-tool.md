---
description: Add a new tool to the AI agent
---

Add a new agent tool: $ARGUMENTS

1. Create the tool file at src/agent/tools/[tool-name].ts
2. Define the tool with: name, description, input schema (zod), execute function
3. Register it in src/agent/tools/index.ts
4. Add TypeScript types for input and output
5. Create a stub version if it calls external services
6. Update the agent system prompt to know about this tool
7. Test: call the agent with a message that should trigger this tool
8. Verify npm run dev has no errors
