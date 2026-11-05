---
tags: [component-architecture]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Our Next.js 15 App Router dashboard page `app/dashboard/page.tsx` is marked "use client". It fetches the current user, the latest invoices and the team members with useEffect, builds `onPayInvoice` and `onInviteMember` handlers, and passes all of it as 7 props into `<Dashboard />`, which hands them on to `InvoiceTable` and `TeamList`. Can you restructure this the right way? Create the files under /tmp/dashboard-example with realistic placeholder data functions.
