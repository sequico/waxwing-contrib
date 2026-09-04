---
name: taurus-i18n
description: Add or change user-visible strings in Waxwing the way the repo's i18n machinery demands — source key in en, parity across all 14 locale bundles, plural categories, the mechanical gate (scripts/locale-rules.mjs, apps/web/src/i18n/locales.test.ts) and the fast check-locales loop. Use whenever a UI string, translation, locale, or i18n key changes in this repo.
---

# Waxwing i18n / locale discipline

The repo ships **14 locale bundles**: `apps/web/src/i18n/locales/{en,de,cs,es,fr,it,ja,nl,
pl,pt,ru,tr,uk,zh}/common.json` — `en` is the source language, `de` is original, the other
twelve were added in v0.21.0 via machine translation behind a mechanical gate (ADR-036;
`docs/translating.md` is the translator-facing doc). Everything user-visible goes through
i18next; hardcoded strings are a DoD violation (CLAUDE.md).

## The rule that matters: parity, both directions

Every key that exists in `en` must exist in all 13 other bundles, and every key that exists
only in a translation is a defect too. Placeholder rules are ASYMMETRIC on purpose: a
translation may DROP a `{{placeholder}}` (the singular form rarely needs one) but must never
INVENT one, and `{{product}}` must never be dropped (white-labelling rule, FR-DEP-04).
`scripts/locale-rules.mjs` is the single implementation of the rules; it is invoked from two
places:

- the gate — `apps/web/src/i18n/locales.test.ts` runs `checkLocale` over all fourteen
  bundles inside `pnpm test`, so `pnpm verify` fails on a short bundle;
- the fast inner loop — `node scripts/check-locales.mjs` (optionally with language args:
  `node scripts/check-locales.mjs ru pl`) names the offender in ~1 s. Run it FIRST when you
  touch i18n, fix, then run the full suite.

## What the mechanical gate checks (beyond key parity)

- **Plural forms come from `Intl.PluralRules`, not a table.** ru/uk/pl/cs have four
  categories where en has two; a missing `_few` falls back to English for exactly the numbers
  2–4 mid-sentence. Keep the plural keys the English entry declares unless the target
  language's categories need more.
- **Placeholders:** an invented `{{placeholder}}` is a defect; omitting `{{product}}` is a
  defect (a drop of any OTHER placeholder can be legitimate).
- **No spelled-out "Waxwing"** where a placeholder/token exists, and the repo's typography
  rules (see locale-rules) hold.
- Translations that are more than half word-identical to the English source are flagged — a
  mechanical smell the rule itself hedges (short labels are legitimately identical across
  languages), so fix the real translations and let the gate's own comment be the judge.

## Changing a string — the order of operations

1. Edit the `en` bundle first (source of truth; the check compares everything to it).
2. `node scripts/check-locales.mjs` — it will tell you exactly which bundles are short.
3. Add the key to every other bundle (`de` included — it is original, not derived). For the
   twelve machine-translated languages the bar is: pass the mechanical gate with a genuine
   translation (not a word-identical copy) — the mechanical check exists precisely so a bad
   translation cannot hide; word choice that only a native speaker can judge is the known
   residual (v0.21.0 changelog).
4. `pnpm check:locales` clean, then the normal gate ladder (taurus-gate).

## Reference

- Rules + why: `scripts/locale-rules.mjs` header, `apps/web/src/i18n/locale-rules.test.ts`,
  `apps/web/src/i18n/locales.test.ts` (each failure class has a named case).
- Policy: ADR-036 (machine translation with a mechanical gate), `docs/translating.md`,
  v0.21.0 entry in implementation-plan §15.
- The language picker shows **endonyms** (`Intl.DisplayNames`) — do not add a language-name
  string table. RTL: ar/he are deliberately NOT shipped (FR-I18N-02; RTL remains unverified —
  see B41 for the three RTL-unsafe surfaces if it ever changes).
- Real bugs the twelve languages once hid: `\b` word boundaries are ASCII-only (Cyrillic/Han/
  Kana mentions were unreachable) and `toLowerCase()` mis-folds Turkish `İ` — use
  `\p{L}\p{N}` with the `u` flag and `toLocaleLowerCase(language)` respectively.
