---
description: Add a new page or screen to the frontend
---

Add a new page: $ARGUMENTS

1. Create the page file at the correct route (src/app/[route]/page.tsx)
2. Add server-side data fetching if the page needs data
3. Create any new components this page needs in src/components/
4. Add the page to sidebar navigation
5. Include: loading state, error state, empty state
6. Connect to existing API endpoints (or create new ones if needed)
7. Make it responsive (test narrow viewport)
8. Add to the breadcrumb trail if applicable
9. Verify npm run dev - page renders without errors
