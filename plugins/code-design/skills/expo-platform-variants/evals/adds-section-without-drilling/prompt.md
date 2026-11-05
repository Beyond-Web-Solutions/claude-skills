---
tags: [expo-platform-variants, component-architecture]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

In our React Native app (Expo, @expo/ui with .ios.tsx and .android.tsx files) the home screen shows a "Recent activity" list. I want to add a "Recommended for you" section above it that loads recommendations from `services/recommendations.ts` and opens the product page when you tap one. The existing home screen component already takes `user`, `recentActivity`, `onOpenProduct` and `onRefresh` as props from the route. Show me how to add the new section the right way, with both platform files, under /tmp/home-example.
