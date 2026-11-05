# Claude skills

[Claude Code](https://claude.com/claude-code) skills used at Beyond Web Solutions,
published as a plugin marketplace.

## Install

In Claude Code:

```
/plugin marketplace add Beyond-Web-Solutions/claude-skills
/plugin install code-design@beyond-web-solutions
```

Update later with `/plugin marketplace update beyond-web-solutions`.

## Plugins

### `code-design`

How we write and structure TypeScript, React and Next.js code.

| Skill | What it covers |
|---|---|
| [`readable-code`](plugins/code-design/skills/readable-code/SKILL.md) | Single Responsibility, no prop drilling, one component per file, `interface Props`, reusable helpers in `utils/` and `hooks/`, named derived values (`useMemo`) instead of logic in JSX, explicit control flow. |
| [`component-architecture`](plugins/code-design/skills/component-architecture/SKILL.md) | Pages and screens as composition only, self-contained sections that own their data, pure leaves. Works for React, Next.js and React Native. |
| [`nextjs-data-streaming`](plugins/code-design/skills/nextjs-data-streaming/SKILL.md) | Next.js App Router with Cache Components: synchronous pages, data fetched in the component that renders it, tight `<Suspense>` boundaries with matching skeletons, `params` / `searchParams` passed down as promises, auth checks in the Data Access Layer. |
| [`expo-platform-variants`](plugins/code-design/skills/expo-platform-variants/SKILL.md) | `.ios.tsx` / `.android.tsx` variants in Expo that share data, behaviour and props and differ only in rendering. |

Each skill has an `evals/` folder with test cases.

## Layout

```
.claude-plugin/marketplace.json     marketplace manifest
plugins/<plugin>/
  .claude-plugin/plugin.json        plugin manifest
  skills/<skill>/SKILL.md           the skill
  skills/<skill>/references/        longer examples, loaded on demand
  skills/<skill>/evals/             eval cases
```

## Contributing

Change a skill in `plugins/<plugin>/skills/<skill>/`, bump `version` in that
plugin's `plugin.json`, and check the manifests with:

```bash
claude plugin validate .
```
