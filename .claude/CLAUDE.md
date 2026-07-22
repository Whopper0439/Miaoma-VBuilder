# Project: miaoma-vbuilder

## Overview
🔥 @miaoma/vbuilder

## Architecture
This is a **monorepo** project.
Max directory depth: 6

## Tech Stack
- **Languages:** Vue (61%), TypeScript (32%), JavaScript (7%)
- **Package Manager:** pnpm
- **Bundler:** Turbopack

## Code Conventions
- **Formatter:** Prettier
- **Linter:** ESLint, Stylelint
- **TypeScript:** strict mode enabled
- **EditorConfig:** present (ensures consistent editor settings)

## File Structure
```
apps/
  builder/
  playwright/
packages/
  blocks/
  eslint/
```

## Key Patterns
No specific architectural patterns detected.

## Testing
- **Convention:** `*.spec.{ts,js}` files

## Build & Deploy
- **Build:** `turbo run build`
- **Dev:** `turbo run dev`

## Common Tasks
- `npm run dev` — turbo run dev
- `npm run build` — turbo run build
- `npm run dev:builder` — pnpm --filter builder dev
- `npm run dev:runner` — pnpm --filter runner dev
- `npm run lint:vue` — eslint --fix
- `npm run lint:style` — stylelint "{packages,apps}/**/*.{css,scss,vue}"
- `npm run lint` — pnpm lint:ts && pnpm lint:style
- `npm run spellcheck` — cspell lint --dot --gitignore --color --cache --show-suggestions "(packages|apps)/**/*.@(html|js|cjs|mjs|ts|tsx|css|scss|md|vue)"
- `npm run typecheck` — pnpm --filter builder typecheck 
- `npm run commit` — git-cz
- `npm run clean` — rimraf "{apps,packages}/**/{node_modules,docs,lib,dist,stats.html}" node_modules pnpm-lock.yaml .eslintcache .cspellcache docs dist

## Gotchas
- **Sparse documentation:** README is minimal or missing. Update it when making significant changes.
- **Sparse documentation:** README is minimal or missing. Update it when making significant changes.
- **Large files:** apps/builder/src/components/ChartRenderer/EchartsRenderer/MOCK_DATA.ts — these may exceed context limits. Consider breaking them up.
- **Low test coverage:** Be careful when refactoring — insufficient tests may hide regressions.
