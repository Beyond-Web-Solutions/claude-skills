---
tags: [readable-code]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

In /tmp/invoices-example there is already `utils/date.ts` exporting `formatDate(date: Date): string` (dd-MM-yyyy, nl-NL). Create that file, then add a new `components/invoices/invoice-row.tsx` that shows an invoice number, its due date and its amount in euros, and calls `onOpen(invoice.id)` on click. Also add the component that renders a list of these rows for a given customer id.
