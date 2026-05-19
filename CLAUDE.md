# Working on this project

## Branch policy — HARD RULE

**No direct commits to `main`.** All changes land via PR from a feature branch — `main` is protected, even the maintainer goes through the PR flow.

Worktree mechanics, branch/PR cleanup, and templated paths are governed by `~/.claude/CLAUDE.git.md`. Don't restate them here.

## Release flow

This repo uses [release-please](https://github.com/googleapis/release-please).

- Every push to `main` (necessarily via PR merge — see above) triggers the
  `Release Please` workflow.
- release-please keeps an open `chore(main): release vX.Y.Z` PR continuously
  up to date with the proposed next version and an auto-generated
  `CHANGELOG.md` derived from conventional-commit messages.
- That PR **auto-merges** once `test` is green (configured in
  `.github/workflows/auto-merge.yml`). You don't manually merge it.
- The release-please merge tags `vX.Y.Z`. The `release.yml` workflow fires
  on the tag → `build.yml` produces `dist/` → `gh-release` uploads to a
  GitHub release → `publish-aur` pushes the new PKGBUILD to AUR.

Effectively: every PR merge to `main` ships a release. There's no "save up
a few commits then release" intermediate state — `main` is always shipped.

### Version bump rules (conventional commits)

| commit type | bump | shown in CHANGELOG |
|---|---|---|
| `feat:` | minor (0.X.0) | yes |
| `fix:` | patch (0.0.X) | yes |
| `feat!:` or `BREAKING CHANGE:` in body | major (X.0.0) | yes |
| `perf:`, `revert:` | patch (0.0.X) | yes |
| `docs:`, `refactor:`, `chore:`, `ci:`, `build:`, `test:` | none | hidden |

> Note: release-please's default versioning strategy couples CHANGELOG visibility with bump-eligibility — *any* non-`hidden` commit type that's not `feat:`/breaking triggers a patch bump (see [`src/versioning-strategies/default.ts`](https://github.com/googleapis/release-please/blob/main/src/versioning-strategies/default.ts)). That's why `docs:` and `refactor:` are kept `"hidden": true` in `release-please-config.json` — making them visible would mean every docs-only PR cuts a release.

## Test-driven changes — HARD RULE

**Every fix or feature PR must include a test that would fail without the
code change.**

Auto-merge is enabled on every non-draft PR (`.github/workflows/auto-merge.yml`);
it fires the moment the `test` status check is green. Without TDD, a PR can
land before any test captures the bug behavior — which means future
regressions slip in silently. TDD is what closes that loop.

The workflow:

1. **Write the failing test first**, against the broken state. Run your
   test command. Confirm it fails — and that the failure message points
   at the bug, not an unrelated assertion.
2. **Write the minimum code to make it pass.** Re-run. Confirm green.
3. **Sanity check** before pushing: stash your code change, re-run the
   test, watch it fail. Pop the stash, re-run, watch it pass. If the test
   passes both ways, the test isn't actually catching what you fixed —
   rewrite it.

```sh
git stash --keep-index -- <your-code-files>
<test command>    # the new test should FAIL here
git stash pop
<test command>    # and pass here
```

When TDD is impractical, prefix the commit explicitly to opt out:

- `docs:` — markdown / comments / inline docstrings, no behavior change
- `ci:` / `build:` — workflows, packaging that aren't unit-testable
- `refactor:` — pure restructuring with no observable behavior change
  (the existing test suite is your safety net)

For `feat:`, `fix:`, `perf:` — TDD is non-negotiable. PR description
should call out which test would have failed pre-fix.

## Commit conventions

- **Format:** `<type>(<scope>): <subject>` per
  [conventional commits](https://www.conventionalcommits.org/). Subject
  under ~70 chars; body explains the *why*.
- **No `--no-verify`** to bypass pre-commit hooks. If a hook fails,
  investigate and fix the underlying issue.

## Before committing — verify hook tooling

`.githooks/pre-commit` (see BURN-AFTER-READING.md for setup) must be able
to find its tooling on PATH. Subagent worktrees routinely lack mise's
activation and will fail loudly if your hook depends on mise-managed
tools. Before any `git commit`, verify the formatter/linter your hook
runs is actually reachable:

```sh
command -v <your-formatter> >/dev/null && echo "ok: $(which <your-formatter>)" \
  || echo "MISSING — run: eval \"$(mise activate bash)\" (or zsh)"
```

Don't `--no-verify` to bypass — if the hook can't run, that's a setup
problem to fix, not a check to skip.
