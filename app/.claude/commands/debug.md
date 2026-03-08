---
description: Debug and fix a specific issue
---

Debug and fix: $ARGUMENTS

1. Reproduce the issue: run the relevant command or navigate to the page
2. Read the error message carefully - identify the exact file and line
3. Trace the root cause (not just the symptom):
   - Is it a type error? Check the interface definition
   - Is it a missing import? Check the export
   - Is it a runtime error? Check the data flow
   - Is it an API error? Check the endpoint and request shape
4. Fix the root cause, not a workaround
5. Check if the same pattern exists elsewhere (grep for similar code)
6. Fix all instances, not just the one reported
7. Run npm run build to verify zero errors
8. Run npm run dev and test the specific flow that was broken
