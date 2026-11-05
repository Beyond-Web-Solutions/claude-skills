---
tags: [component-architecture]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

I have an Expo Router app. My `app/orders.tsx` route fetches the current user, the open orders, the delivery slots and the cart, builds `onSelectOrder`, `onCancelOrder` and `onPickSlot` handlers, and passes all of it as 9 props into `screens/orders.tsx`, which passes most of them on to `OrderList`, `SlotPicker` and `CartSummary`. Every time I add something I have to touch four files. Can you restructure this properly? Create the files under /tmp/orders-example with realistic placeholder hooks so I can see the target shape.
