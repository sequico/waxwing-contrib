---
name: taurus-implement
description: Carry a plan-driven implementation or defect fix on the Taurus/Waxwing repo end to end — read the right docs, start or continue a work package on the status board, run the guarded implementation + adversarial-review workflow, then close the session by updating the board, changelog and ADRs. Use when starting work that follows docs/implementation-plan.md, fixing a tracked Bxx/N-xx defect, or adding a feature to this repo.
---

# Taurus implementation protocol

This repo was built by plan-driven work packages with an adversarial review at every
milestone; a change that follows the pattern lands, one that skips it drifts. The full law
is `.codewhale/instructions.md`; this skill is the how-to.

## 0. Before writing anything

- Read the docs in order (`.codewhale/instructions.md` §2): functional-specification →
  tech-stack → implementation-plan status board → CONTRIBUTING → CLAUDE.md.
- Skim the ADRs; the constraints they record will otherwise look arbitrary (engine keyed by
  account ADR-018, SSE-first push ADR-005, one subscription ADR-035, creates-not-idempotent
  ADR-038, …).
- Check the board's current state: since v0.22.0 **every WP is `done`** and new work is a
  §11 backlog item, a §13 owner action, or a defect with a `Bxx`/`N-xx` id. If the user asks
  for work the board does not name, do not invent a WP row silently — add one only as the
  session's bookkeeping (protocol §2.2), and say so.

## 1. Scope the change

- Anchor it to the requirement: FR-/NFR- id, WP title, or defect id. Name exit criteria in
  the brief before code exists.
- One WP (or one coherent fix) at a time. If it is too large for the session, finish a
  coherent subset, check off finished tasks, leave the repo green.
- When a plan row exists: set Status to `in-progress` in the board **before** writing code.

## 2. Implement — guarded

- For anything beyond a one-file fix, work in a separate **git worktree or branch**; never
  run parallel writers in the main checkout. In Codewhale: `agent(action=start, type=builder,
  worktree=true)` with bounded `write_roots` covering only the areas you own.
- For genuinely divergent approaches, run candidates in parallel (2–4, one rubric) and pick
  the winner on evidence — dispatch is not completion: re-check the merged result yourself.
- Follow the DoD from the start: TS strict, Biome, i18n keys into all 14 bundles, axe on
  new interactive surfaces, account-scoped data, budget. See skill **taurus-gate**.

## 3. Prove it — tests that can fail

- Unit/component test for every new behaviour, next to the source; E2E for user-visible
  flows (skill **taurus-e2e**).
- **Mutation-check the fix**: remove it, watch the test go red. Comment states which defect
  the test guards.
- Geometry/colour claims go to the Playwright suites, not jsdom. Flaky-looking? It is a
  race — wait for the state (interact.ts helpers), never paper over it.

## 4. Adversarial review — by someone else

- The implementer does not review their own work. Get a separate reviewer
  (`agent(action=start, type=reviewer)`), or at minimum review the diff cold against the
  brief.
- Findings are named (`Bxx`/`N-xx`), each either fixed at root cause with a regression test
  or refuted with evidence — this repo's history records both, and "not fixed but not
  claimed fixed" is an honest, usable outcome. Record what a review found even when it found
  nothing: the repo's review reports live in `docs/reviews/`.
- A review that confirms zero findings in code that has none is normal; a review that
  rubber-stamps is not a review.

## 5. Gate it

- `pnpm verify` for the inner loop; `pnpm gate` (or `pnpm verify:e2e` when Docker is
  available but a full gate is not warranted) before proposing. Both must be green; if they
  cannot pass locally, say why — a named failure beats quiet hope (CONTRIBUTING).

## 6. Close the session — bookkeeping is part of done

- Update the status board and the WP's task checkboxes to reality; add follow-ups as new
  checkbox items (never only code comments or chat).
- Append a one-line changelog entry in `implementation-plan.md` §15 (date, WP(s)/defect,
  outcome) — the house style for a real change is a dense paragraph: what was wrong, what it
  cost, the evidence, what stays open and why.
- Deviation from spec/stack/plan? Write the ADR (`docs/adr/NNN-title.md`, MADR one-pager,
  next sequential number) and update the affected docs. Never diverge silently.
- Keep `.codewhale/instructions.md` and CLAUDE.md in sync if a rule changed.
- Repo green, no stray files. Do not commit, push, tag or release unless the user asked for
  it in the current turn — leave the work in the working tree and propose the exact commands.

## 7. Handoff

If the work spans sessions, close with a handoff that states: what is done and verified
(commands + outputs), what remains, which findings are open, and where the next session
should start on the board.
