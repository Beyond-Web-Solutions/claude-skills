---
tags: [nextjs-data-streaming]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Our Next.js 16 app (cacheComponents: true) has `app/admin/page.tsx`: `export default async function Page()` that first does `await requireSession()`, then `const [stats, orders] = await Promise.all([getStats(), getRecentOrders()])`, and renders a stats card and an orders table. There is also `app/admin/loading.tsx` rendering `<AdminPageSkeleton />` from `components/loading/page-skeletons.tsx`. The page takes 2 seconds before anything shows. Rewrite it the right way under /tmp/admin-example, with placeholder queries.
