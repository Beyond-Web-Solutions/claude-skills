---
tags: [readable-code]
max_turns: 30
timeout_seconds: 900
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Create /tmp/appointments-example/components/appointment-list.tsx as it is now, then clean it up. Current file: a `type AppointmentListProps` alias; a `formatDutchDate(date: Date)` helper and a `const STATUS_COLORS = {...}` above the component; inside the JSX `appointments.filter(a => !a.cancelled && a.date >= today).sort((a,b) => a.date - b.date).map(...)`; a nested ternary for the status label (`a.paid ? "Betaald" : a.overdue ? "Te laat" : "Open"`); an `if (loading) return <AppointmentListSkeleton />` where `AppointmentListSkeleton` is declared at the bottom of the same file; and one-line `if`s without braces. Write the cleaned-up version and every file it needs.
