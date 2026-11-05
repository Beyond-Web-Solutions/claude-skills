---
type: llm
focus: trace
weight: 1
---

The result matches this description: the page is synchronous and passes the searchParams promise to the table, which awaits and zod-parses it inside its own Suspense boundary; the filter is a client leaf updating the URL; heading and filter are outside Suspense; the outstanding total has its own boundary and skeleton; queries check the session in the DAL; skeletons are separate files; no loading.tsx.
