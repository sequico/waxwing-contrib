---
name: taurus-upstream
description: Contribute work from this checkout (the public fork sequico/waxwing-contrib) to the upstream repository Heiko-W/waxwing — remote layout, cutting feature branches from upstream/main so local-only commits never ride in a PR, syncing the fork, and opening the pull request against upstream. Use whenever work done in this repo is meant to land upstream, or the fork needs fetching/rebasing.
---

# Waxwing contribution workflow (fork → upstream)

This checkout is a **fork of Heiko-W/waxwing**, and its whole purpose is upstream
contribution. Two facts shape everything here:

1. **The fork's `main` carries local-only commits** that upstream does not have (e.g.
   e0f703f, which adds the `.agents/skills/` operating skills; nothing upstream-ward may
   ever include them).
2. **A PR is a diff against upstream** — anything extra in the branch (skills, agent
   state, fork-only docs) shows up in it. CONTRIBUTING's rule is "fork and branch from
   main"; on this fork that means **from `upstream/main`**, not from the fork's own main.

## Remote layout (as configured in this checkout)

- `origin` = https://github.com/sequico/waxwing-contrib.git (this fork — where branches
  and PRs are pushed from).
- `upstream` = https://github.com/Heiko-W/waxwing.git (the project — `main` here is what
  an upstream PR must be based on and diffed against).
- Authentication is the gh credential helper (https). There is no auto-push anywhere:
  committing, pushing, or opening a PR happens only when the user asks for it in the
  current turn.

## Starting work that must land upstream

```sh
git fetch upstream            # upstream/main is the base, always fresh
git switch -c feat/NAME upstream/main
# or, for parallel work: git worktree add ../waxwing-feat-NAME feat/NAME upstream/main
```

Base = upstream/main, so the branch contains **only** the feature commits — upstream/main has
NO `.agents/` (verified: the fork's main differs from upstream/main only by the skills commit
e0f703f). If work was started on the fork's own main by mistake, rebase it before proposing:
`git rebase --onto upstream/main <fork-main-point> feat/NAME`. This matters because the
fork main includes the local-only skills commit — a branch cut from it would offer
`.agents/` to upstream.

Sync the branch from `upstream/main` ONLY. Never `git merge origin/main` (the fork's own
main) into a feature branch — that is the other way the skills commit rides in.

## The rule, stated as a tripwire

The skills must stay on the fork's `main` and never appear in a feature branch's diff
against upstream. Before proposing, run:

```sh
git diff --name-only upstream/main...HEAD | grep -E '^\.agents/' && echo 'INQUINATO — .agents nel branch' || echo 'pulito'
```

Also check `git diff --stat upstream/main...HEAD` — nothing but the feature and its
docs/ADRs may be in it.

While working on a feature branch, never commit with a blanket `git add .` / `git add -A`:
stage only the feature paths. Uncommitted edits to `.agents/skills` in the working tree
follow a `git switch` onto the new branch — commit (or stash) them on the fork's `main`
first, or they silently become part of the feature branch's next commit.

## The local pre-push barrier (installed 2026-09-04)

This clone has a local-only pre-push hook at `.git/hooks/pre-push` (NOT versioned — it
cannot ride into any branch or PR; a copy lives at `~/.codewhale/hooks/pre-push-waxwing`
for reinstalling after a fresh clone). It refuses, with an actionable message:

- any push to upstream (Heiko-W/waxwing);
- deleting `refs/heads/main` (the skills' designated home);
- pushing any branch other than `main` whose `upstream/main...<branch>` diff touches
  `.agents/` or `.codewhale/` — i.e. the exact diff an upstream PR would show.

Pushing `main` is always allowed. Bypass is `git push --no-verify` (and means it).
Caveat: if `core.hooksPath` is ever set (e.g. to `.githooks` for the gate), git stops
reading `.git/hooks/pre-push` — merge the two hooks then.

Tested 2026-09-04: polluted branch → refused (exit 1); clean branch and `main` → allowed
(exit 0); upstream push and main deletion → refused.

## Before the PR (CONTRIBUTING, in short)

- Branch names are free-form; Conventional Commits carry the meaning (scopes: `web`,
  `jmap`, `mail-html`, `jscontact`, `sync`, `e2e`, `docs`, `ci`).
- The gate must be green locally: `pnpm verify` (inner loop) and `pnpm gate` before
  proposing — see skill **taurus-gate**. If it cannot pass locally, say so in the PR and
  why; a named failure is welcome, a quiet hope is not.
- Sanity-check the diff scope: `git diff upstream/main...HEAD --stat` — nothing but the
  feature and its docs/ADRs may be in it (see the tripwire above).
- Never open the PR from the fork's `main` branch, and never against the fork's own main as
  base: base must be upstream `main`, head a feature branch (`sequico:feat/NAME`).
- Upstream's CI runs `pnpm verify` (required check) and `pnpm verify:e2e` (not required —
  a deliberate asymmetry, read CONTRIBUTING). No approving review is required to merge;
  the project is one-maintainer.

## Opening the PR

```sh
git push -u origin feat/NAME
gh pr create --repo Heiko-W/waxwing   # branch checked out → gh derives the head
# explicit form when needed: --base main --head sequico:feat/NAME  (owner:branch)
```

Note the `--head` shape: `owner:branch` (`sequico:feat/NAME`), never `owner:repo:branch`.
If the repository is ever detached from the fork network (private fork), it CANNOT open
PRs to its former parent — that state is not recoverable from the API side; keep the fork
public.

## Syncing the fork's main

Local-only commits on main should stay stacked on top of upstream:

```sh
git fetch upstream
git rebase upstream/main      # replays e0f703f (skills) and anything else local on top
git push --force-with-lease origin main   # only with the user's go-ahead
```

Never merge upstream into the fork main unless you want merge commits in PRs; a PR base
that contains upstream history is fine, one that contains the local-only commits is not.

## Material that stays local, by design

- `.agents/skills/` (this skill set), `.codewhale/` (agent runtime state) — upstream has
  neither; neither belongs in a PR.
- `e2e/audit/out/`, `e2e/shots/out/`, `playwright-report/`, `test-results/` (gitignored
  capture output — evidence, not PR material).
- Anything the user explicitly keeps out of the contribution (feature work not yet ready,
  personal notes). When in doubt, list the files in the handoff and let the user decide.
