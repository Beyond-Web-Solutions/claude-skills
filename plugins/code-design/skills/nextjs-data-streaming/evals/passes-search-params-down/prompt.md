---
tags: [nextjs-data-streaming]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Add a page `app/invoices/page.tsx` to our Next.js 16 app (cacheComponents: true) that shows a heading, a status filter (open / paid / all, stored in `?status=`) and a table of invoices filtered by that status, plus a small "total outstanding" figure that does not depend on the filter. Create it under /tmp/invoices-page-example with placeholder queries.
