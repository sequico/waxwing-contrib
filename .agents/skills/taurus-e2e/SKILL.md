---
name: taurus-e2e
description: Operate the Waxwing Playwright suites and the Stalwart JMAP Docker fixture — bring the fixture up/down/provision/smoke, choose which suite or config answers which question, run single specs, and handle teardown discipline. Use whenever e2e, Playwright, the fixture, seeding, or "run it against a real server" comes up in this repo.
---

# Waxwing e2e and the Stalwart fixture

Playwright drives the REAL production bundle against a live Stalwart server in Docker
(`e2e/stalwart/`, design: ADR-002, `e2e/stalwart/README.md`). Facts below were verified
2026-09-04 against v0.22.0 (1d38e7a); the README and the configs are the source of truth when
they drift.

## Fixture essentials

- Requires Docker (`docker` + `docker compose`). Plain HTTP on **localhost:18080**
  (container port 8080). Compose profiles `dev` (baseline pin
  `stalwartlabs/stalwart:v0.16.18-alpine`, container `waxwing-stalwart-dev`) and `main`
  (compat) — a bare `docker compose up` starts nothing; both variants share the port, so
  only one may run at a time.
- Domain **`waxwing.test`** (RFC 6761 reserved). Accounts, one dev-only password
  **`waxwing-e2e-Pw1!`**: `admin` (recovery admin via `STALWART_RECOVERY_ADMIN`) and
  `alice@`, `bob@`, `carol@` `waxwing.test`.
- **Invariant:** `provision()` leaves the three accounts standalone (single-account UI).
  Delegations are opt-in: the shared suite grants them in its setup and revokes in teardown.
  `smoke()` asserts the single-account default, so a leaked share fails at its source.
- **Volume discipline:** `up` never wipes the data volume (would destroy seeded state).
  `down` is the explicit data-losing step. After changing the pinned image tag you MUST
  `pnpm e2e:server:down` first — Stalwart seeds server defaults (e.g. the Web-Push VAPID
  keypair) only into a virgin registry, and a stale volume silently loses them.

## Commands (from repo root)

```sh
pnpm e2e:server                  # fixture up -> wait ready -> provision -> smoke (idempotent)
pnpm e2e:server:down             # compose down -v (containers + ephemeral volumes)
pnpm e2e:server:main             # the `main` compat profile instead of `dev`
node e2e/stalwart/fixture.mjs status      # container state + connection details
pnpm --filter @waxwing/e2e run server:provision   # idempotent domain+accounts (server already up)
pnpm --filter @waxwing/e2e run server:smoke       # assert it is a working, auth-enforcing JMAP server
pnpm demo [--lan]                # fixture + seeded mail + Vite dev on :5173 (demo mode, teardown guaranteed)
```

If Docker is missing/stopped, `up` fails fast with an actionable message — that is its
preflight. A leaked container (Ctrl-C during a suite) trips the next `up`'s port-conflict
preflight: run `pnpm e2e:server:down` before retrying.

## Suite matrix — which config answers which question

Gated suites (the seven `pnpm verify:e2e` Playwright runs, cheapest first — enumerated in
`scripts/verify-e2e.mjs`; the `@waxwing/jmap` integration stage runs between mount and read;
a suite not listed there is NOT gated). Each config pins `testMatch` deliberately — read the
config's header for the current list. The one allowed overlap is `read.spec.ts`, which the
webkit config runs a second time (B11); `scripts/e2e-suites.test.ts` guards R-44: no other
spec may run under two gate configs.

- `pnpm e2e` (playwright.config.ts) — `shell.spec.ts` only, fixture-free placeholder: a
  bundle that cannot boot fails in seconds.
- `pnpm e2e:mount` — `mount.spec.ts`, fixture-free static mount: boot under a path prefix
  (the Stalwart `/mail/` deployment shape).
- `pnpm e2e:read` — the M1.9 read harness, serial with per-test reseed: `read` plus `keyboard`,
  `offline`, `push`, `pwa`, `notify`, `target-size`, `focus-visible`, `security`, `a11y`,
  `perf`, `public-computer`, `narrow`, `viewports` at a 1440×900 desktop viewport, with
  dedicated projects: `chromium-touch` (swipe), `chromium-phone` (narrow 390×844),
  `chromium-notify` (full chromium build + notification permission). Fixture self-managed,
  alice seeded.
- `pnpm e2e:write` — the M2.9 write harness, serial with per-test reset: `write`,
  `settings`, `account-security`, `contacts`, `calendar`, `files`.
- `pnpm e2e:shared` — delegation + sharing: `shared`, `sharing`, `sharing-pim`,
  `delegation`. Grants/revokes shares of bob's and carol's Inboxes (and PIM shares) to
  alice in setup/teardown.
- `pnpm e2e:webkit` — `webkit.spec.ts` + a second full pass of `read.spec.ts` (B11; Safari
  disagreed with Chromium in real defects — ADR-029). Needs the WebKit browser installed:
  `pnpm --filter @waxwing/e2e exec playwright install webkit`.
- `pnpm e2e:deploy` — `deploy.spec.ts`; the only suite that builds the app TWICE (staged
  second deploy, `e2e/pwa-stage.vite.config.mjs`), so it runs last.

Deliberately NOT in the gate (run by hand when you need them):

- `pnpm e2e:demo` — demo spec (dev demo mode).
- `pnpm e2e:large` — `perf-large.spec.ts` (100k-message perf; seed via `pnpm seed:large`).
- Screenshot/audit passes: `pnpm shots` (docs/site shots, converts to WebP), and the
  Playwright audit/sicht configs (`playwright.audit.config.ts`, `playwright.sicht.config.ts`)
  with their own `e2e/audit/*.spec.ts` captures — evidence for a review, outputs gitignored.

Seeding is normally done by each suite's setup (`e2e/*.setup.mjs`, `e2e/stalwart/seed-*.mjs`);
manual seeding after a fixture `up`: `node e2e/stalwart/seed-read.mjs` (or the `pnpm seed:*`
aliases where present). Integration tests for `@waxwing/jmap` live outside Playwright:
`pnpm verify:integration` / `scripts/integration.mjs` — they FAIL OPEN, so the gate asserts
they were not skipped (defect B22).

## Running one spec

```sh
pnpm --filter @waxwing/e2e exec playwright test -c playwright.read.config.ts --grep "thread"
# or with a file filter; the config's projects/testMatch decide what collects what
```

Read the spec's config header first: every config pins `testMatch` deliberately (a missing
match silently collects everything — a defect class this repo has paid for twice). No spec
may run under two gate configs, except `read.spec.ts` on WebKit (R-44, guarded by
`scripts/e2e-suites.test.ts`).

## Teardown discipline (non-negotiable)

The gate backstops teardown in a `finally`; when you run suites by hand, mirror that:
bring the fixture down when done (`pnpm e2e:server:down`), especially before image bumps.
Suites respect `WAXWING_KEEP_FIXTURE=1` to leave the fixture up for iteration (their
teardowns still revoke shares/sweep state first). Report evidence: which suite/config ran,
engine(s), counts and exit status — not just "e2e passed".

## Gotchas seen in the repo's own history

- A share notice arriving pushes folder rows down under the pointer → click coordinates go
  stale; prefer focus-based activation (`press('Enter')`) over hovers that assert position.
- Don't click a control the moment a query returns it (disabled until synced) — see
  taurus-gate.
- The fixture is deliberately insecure (well-known passwords, no TLS). Never point it at
  anything real, never expose it.
