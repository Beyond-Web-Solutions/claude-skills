---
tags: [nextjs-data-streaming]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Our Next.js 16 app (cacheComponents: true) has a `proxy.ts` that only runs the next-intl middleware, and a settings page at `app/[locale]/dashboard/settings/page.tsx` whose `ProfileCard` calls `getProfile()` from `features/settings/data/queries.ts`. `getProfile()` reads the database without checking who is signed in. Signed-out visitors should end up on `/sign-in` instead of seeing the page. The quickest fix seems to be checking the session cookie in `proxy.ts` for everything under `/dashboard`. Make it work under /tmp/settings-example, with placeholder auth and queries.
