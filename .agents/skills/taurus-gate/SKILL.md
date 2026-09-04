---
name: taurus-gate
description: Run the Waxwing verification ladder and Definition of Done for a change — choosing between the inner loop (pnpm verify), the hermetic pre-push half (pnpm gate:fast), the full Docker+browser gate (pnpm gate) and pnpm verify:e2e, plus the i18n/budget checks. Use whenever a change needs to be made green on this repo or a failure needs interpreting.
---

# Waxwing gate

Run this repository's own checks — never substitute an opinion for the scripts (ADR-003:
everything CI runs is a pnpm script you can run here; `.github/workflows/ci.yml` is a thin
caller of the same scripts, and CONTRIBUTING says what the local gate is for).

## Preflight

- Node major **must be exactly 24** (`.nvmrc` = `24`, `engines ">=24 <25"`). `pnpm check:node`
  (first step of `pnpm verify`) refuses to run on any other major, and `engineStrict: true` in
  `pnpm-workspace.yaml` already stops a wrong major at `pnpm install`. This is not pedantry: on
  node >= 25 a global `localStorage` exists but is `undefined` without `--localstorage-file`,
  and it shadows jsdom's — measured on node 26 as 22 healthy tests failing (later 54), with
  nothing wrong in the code under test. Switch with `nvm use` rather than debugging a phantom.
- pnpm: the `packageManager` pin in package.json is **`pnpm@11.1.1`** — use what corepack
  installs from the pin (engines allow `>=10`; the pin is the law). Do not fight or "upgrade"
  the pin on a whim; if a NEWER pin is already in package.json, accept it rather than reverting.
- `pnpm install` once; `node_modules/` present.

## The ladder — pick by what changed

| Situation | Command | What it runs / costs |
|---|---|---|
| Any code change, before proposing; also the CI "required check" | `pnpm verify` | hermetic, ~2 min: check:node → build:libs → typecheck → lint → test (all Vitest projects incl. `scripts/*.test.ts`) → size (budget ≤ 300 KB gz, NFR-PERF-01; ~285 KB gz measured at v0.22.0) → check:dist → check:site → check:actions → check:nul |
| Pre-push | `pnpm gate:fast` | the hermetic half — `.githooks/pre-push` calls it (enable once per clone: `git config core.hooksPath .githooks`) |
| i18n touched | `pnpm check:locales` first | names the short locale bundle(s) in ~1 s; the same rules also run inside `pnpm test` (`apps/web/src/i18n/locales.test.ts` checks all 14 bundles) — see skill **taurus-i18n** |
| One narrow area | `pnpm typecheck`, `pnpm lint` (full `biome check .`), or a filtered test: `pnpm vitest run apps/web/src/mail` / `pnpm vitest run <file>` | fastest feedback; still run `pnpm verify` before calling it green |
| Anything touching sync/jmap, browser-only behaviour (layout, colour, focus, touch, sandboxed frame), or user-visible flows | `pnpm verify:e2e` (needs Docker) or one suite — see skill **taurus-e2e** | placeholder → mount → jmap integration → read → write → shared → webkit → deploy; guaranteed teardown |
| Proposing a real change | `pnpm gate` | full pipeline ~10 min: preflight → verify → integration vs live fixture → E2E. `pnpm gate --no-e2e` stops after integration |

If a step fails, read the failing test/spec output, not only the exit status. A red test is
either a real defect or a test defect — see CONTRIBUTING before deciding which.

## Definition of Done recap

Full law: `CLAUDE.md` (Rules) + `docs/implementation-plan.md` §2.4 (DoD) and §2.5
(conventions). **This repo has no `.codewhale/instructions.md` — upstream's CLAUDE.md
governs.** Key points:

- TypeScript strict: `tsconfig.base.json` sets `strict`, `noUncheckedIndexedAccess`,
  `exactOptionalPropertyTypes` — no `any`, no non-null assertion to slip past a type error.
- Biome clean; an `ignore` comment needs a stated reason (repo practice: the line above it).
- Tests: new logic unit/component-tested; touched user-visible flows keep E2E coverage. The
  repo's convention is that a test guarding a defect says so in a comment (`Bxx`/`R-##`/`N-##`
  ids appear all over the suites). **Mutation-prove the fix**: remove it, the test goes red —
  CONTRIBUTING's central rule.
- No hardcoded user-visible strings: new i18n key → `en` first, then all 14 bundles under
  `apps/web/src/i18n/locales/*/common.json`; `pnpm check:locales` says which one is short.
- Keyboard/a11y basics + axe on new interactive surfaces (jsdom cannot judge contrast or
  target size — those claims go to the Playwright suites: `a11y.spec.ts`, `target-size.spec.ts`).
- Budget: `pnpm size` ≤ 300 KB gz (`.size-limit.js`, NFR-PERF-01; the build measures ALL eager
  chunks — lazy ones must be named in the exclude list). New dependency justified against
  tech-stack §2, AGPL-compatible; `packages/jmap` and `packages/jscontact` stay MIT.
- Persisted data account-scoped (FR-AUTH-07 readiness); docs updated where behaviour/config
  changed (ADR per plan §2.3, never a silent deviation).

## Common failure classes and where they belong

- **jsdom can't see geometry/colour.** A green unit suite says nothing about a contrast
  ratio, a 44 px target or a bounded scroll container. If the claim is visual, the test
  belongs in `e2e/` — say so instead of faking it in jsdom.
- **A flake is a race.** Disabled-until-synced controls are silent no-ops under
  `click()` — use `clickButton`/`clickWhenEnabled` (`apps/web/src/test/interact.ts`, and the
  rule CONTRIBUTING states: if a value crosses a store, a database or the network, wait for it).
- **Never delete a failing test to green the gate.** Wrong test → fix/remove and say so;
  right test → the code is wrong.
- **`pnpm verify` output lies only when it can't run**: wrong Node major (must be 24),
  missing `build:libs` (verify does it first — the error reads like an app bug if you skip it),
  missing Playwright browsers (`pnpm --filter @waxwing/e2e exec playwright install chromium`
  / `webkit`).

## Done

Exit status 0 on the chosen level, with the actual command recorded in your summary.
Never claim "all green" from a subset: name exactly which commands ran and what passed.
