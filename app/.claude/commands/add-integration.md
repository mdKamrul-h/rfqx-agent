---
description: Add a new external service integration
---

Add integration for: $ARGUMENTS

1. Create src/lib/integrations/[service-name]/
2. Create types.ts - TypeScript types for requests and responses
3. Create client.ts - real implementation using the service SDK/API
4. Create stub.ts - mock implementation that returns realistic sample data
5. Create index.ts - exports real or stub based on USE_STUBS env var:
   ```
   const useStub = process.env.USE_STUBS === 'true'
   export const client = useStub ? stubClient : realClient
   ```
6. Add all required env vars to .env.example with descriptions
7. Add USE_STUB_[SERVICE]=true to .env.example
8. Install any needed npm packages
9. Verify npm run dev has no errors
10. Test: call the integration from an API route or agent tool
