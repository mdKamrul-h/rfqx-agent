---
description: Add a new multi-step skill to the AI agent
---

Add a new agent skill: $ARGUMENTS

A skill is a complex capability that composes multiple tools in sequence.

1. Create src/agent/skills/[skill-name].ts
2. Define the skill with: name, description, steps (which tools to call in what order)
3. Implement the orchestration logic: call tool A -> process result -> call tool B -> combine
4. Add proper error handling at each step (if tool A fails, what happens?)
5. Register in src/agent/skills/index.ts
6. Update agent system prompt to describe when to use this skill
7. Add a test case: a user message that should trigger this skill end-to-end
8. Verify npm run dev has no errors
