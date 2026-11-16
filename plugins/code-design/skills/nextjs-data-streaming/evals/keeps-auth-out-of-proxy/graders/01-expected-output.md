---
type: llm
focus: trace
weight: 1
---

The result matches this description: proxy.ts is left with only the next-intl middleware and has no session or cookie check; getProfile is in a server-only DAL file and starts with a cached requireSession() that redirects to sign-in without a session; the page does not check the session; the answer explains that auth belongs in the Data Access Layer rather than the proxy.
