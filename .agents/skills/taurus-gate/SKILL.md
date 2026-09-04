---
name: taurus-gate
description: Run the Taurus/Waxwing verification ladder and Definition of Done for a change — choosing between the inner loop (pnpm verify), the hermetic pre-push half (pnpm gate:fast), the full Docker+browser gate (pnpm gate) and pnpm verify:e2e, plus the i18n/budget checks. Use whenever a change needs to be made green on this repo or a failure needs interpreting.
---

# Taurus gate

Run this repository's own checks — never substitute an opinion for the scripts (ADR-003:
everything CI runs is a pnpm script you can run here).

## Preflight

- Node major **must be 24** (`node --version`; `.nvmrc`, `engines ">=24 <25"`). The gate
  refuses to run on a wrong major on purpose: Node 26 defines a global `localStorage` that
  shadows jsdom's and fails ~22 healthy tests. Switch with `nvm use` / `corepack` rather
  than debugging a phantom failure.
- pnpm: use the `packageManager` pin in package.json (`pnpm@11.25.0` as of 2026-09; the pin
  has self-updated before — accept the newer pin rather than reverting it).
- `pnpm install` once; `node_modules/` present.

## The ladder — pick by what changed

| Situation | Command | What it runs / costs |
|---|---|---|
| Any code change, before proposing; also the CI "required check" | `pnpm verify` | hermetic, ~2 min: check:node → build:libs → typecheck → lint → test (all Vitest projects incl. `scripts/*.test.ts`) → size (budget ≤300 KB gz, ≥15 % headroom) → check:dist → check:site → check:actions → check:nul |
| Pre-push | `pnpm gate:fast` | the hermetic half (`.githooks/pre-push` calls it) |
| i18n touched | `pnpm check:locales` first | names the short locale bundle in ~1 s; fix before running the full suite |
| One narrow area | `pnpm typecheck`, `pnpm lint` (full `biome check .`), or a filtered test: `pnpm vitest run apps/web/src/mail` / `pnpm vitest run <file>` | fastest feedback; still run `pnpm verify` before calling it green |
| Anything touching sync/jmap, browser-only behaviour (layout, colour, focus, touch, sandboxed frame), or user-visible flows | `pnpm verify:e2e` (needs Docker) or one suite — see skill **taurus-e2e** | placeholder → mount → integration → read → write → shared → webkit → deploy; guaranteed teardown |
| Proposing a real change | `pnpm gate` | full pipeline ~10–12 min: preflight → verify → integration vs live fixture → E2E. `pnpm gate --no-e2e` stops after integration |

If a step fails, read the failing test/spec output, not only the exit status. A red test is
either a real defect or a test defect — see CONTRIBUTING before deciding which.

## Definition of Done recap (full law: `.codewhale/instructions.md` §4, plan §2.4)

- TypeScript strict (`exactOptionalPropertyTypes`): no `any`, no non-null assertion to slip
  past a type error.
- Biome clean; an `ignore` comment needs a stated reason on the line above.
- Tests: new logic unit/component-tested; touched user-visible flows keep E2E coverage;
  each test comment names the defect it guards; **mutation-prove the fix** (remove it, test
  goes red).
- No hardcoded user-visible strings: new i18n key → `en` first, then all 14 bundles under
  `apps/web/src/i18n/locales/*/common.json`; `node scripts/check-locales.mjs` says which.
- Keyboard/a11y basics + axe on new interactive surfaces (jsdom cannot judge contrast or
  target size — those claims go to the Playwright suites).
- Budget: `pnpm size` ≤ 300 KB gz with ≥ 15 % headroom (~285 KB gz today). New dependency
  justified against tech-stack §2, AGPL-compatible; `packages/*` stay MIT-clean.
- Persisted data account-scoped; docs updated where behaviour/config changed.

## Common failure classes and where they belong

- **jsdom can't see geometry/colour.** A green unit suite says nothing about a contrast
  ratio, a 44 px target or a bounded scroll container. If the claim is visual, the test
  belongs in `e2e/` — say so instead of faking it in jsdom.
- **A flake is a race.** Disabled-until-synced controls are silent no-ops under
  `click()` — use `clickButton`/`clickWhenEnabled` (`apps/web/src/test/interact.ts`). If a
  value crosses a store, a database or the network, wait for it.
- **Never delete a failing test to green the gate.** Wrong test → fix/remove and say so;
  right test → the code is wrong.
- **`pnpm verify` output lies only when it can't run**: wrong Node major, missing
  `build:libs` (verify does it first — the error reads like an app bug if you skip it),
  missing Playwright browsers (`pnpm --filter @waxwing/e2e exec playwright install
  chromium`).

## Done

Exit status 0 on the chosen level, with the actual command recorded in your summary.
Never claim "all green" from a subset: name exactly which commands ran and what passed.
