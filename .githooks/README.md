# Local git hooks (`.githooks/`)

Version-controlled git hooks for Resume-Matcher. We **do not** run a
PR-triggered GitHub Actions test workflow (the repo gets a high volume of
external contributor PRs, and CI would run on every one of them). Instead, the
maintainer's local clone gates pushes with a `pre-push` hook — a "local CI" that
keeps `main`/`dev` green without touching contributor PRs.

## What runs

`pre-push` runs before every `git push` and **blocks the push if anything is red**:

1. **Backend test suite** — `uv run pytest` in `apps/backend` (~8s). Deterministic;
   the LLM-as-judge evals are excluded by default (`addopts -m "not eval"`), so
   it makes **no network/LLM calls**.
2. **Frontend locale parity** — `scripts/check_locale_parity.py` verifies every
   `apps/frontend/messages/*.json` has the same key structure as `en.json`.
   Pure Python (no Node/npm/nvm). This guards the exact i18n mismatch that once
   broke `next build` and only surfaced post-merge in the Docker job.
3. **Frontend test suite** — `vitest run` in `apps/frontend`, but only when Node
   and the local vitest binary are present (git hooks may run without nvm's
   `node` on `PATH`); otherwise skipped with a warning.
4. **Frontend type check** — `tsc --noEmit` in `apps/frontend` (~3s), same
   availability rule as the test suite. `lib/i18n/messages.ts` types the locale
   files off `typeof en`, so a missing key is a *type* error — check 2 catches
   that one case, this is the general gate. A full `next build` is still not run
   here: minutes, not seconds.

All checks always run, so you see **all** failures at once — and any that were
SKIPPED are named in the closing line, so a green never reads as "everything
ran".

## Activate (once per clone)

```bash
git config core.hooksPath .githooks
```

That's it — hooks are now active for this clone. (It's a local git setting; it
does not affect anyone else who clones the repo, by design.)

## Everyday use

- Commit freely — the gate runs at **push**, not on every commit.
- If the gate fails, the push is aborted and the failures are printed. Fix them and push again.

## Escape hatches

```bash
git push --no-verify          # bypass the gate once (docs-only / WIP branches)
git config --unset core.hooksPath   # disable the hooks entirely
```

## Run the checks manually

```bash
cd apps/backend && uv run pytest          # backend suite
python3 scripts/check_locale_parity.py    # locale parity (from repo root)
cd apps/frontend && npm run test          # frontend suite (vitest)
cd apps/frontend && npm run type-check    # tsc --noEmit
```

See [`docs/agent/testing-strategy.md`](../docs/agent/testing-strategy.md) for the
full testing strategy this gate enforces.
