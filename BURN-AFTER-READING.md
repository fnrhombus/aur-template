# BURN AFTER READING

One-time setup to get the publish chain working in the new repo. Once each
step below is done, **delete this file** in your first PR — leaving it
around makes future cloners think setup is still pending.

This file contains **only process steps**, no secret values. Anything in
`<ALL CAPS>` is a thing you produce, not a thing this template provides.

## 1. AUR account + package

If this is the first AUR package you publish, do these once globally:

1. Create an AUR account at <https://aur.archlinux.org/register>.
2. Generate an ED25519 SSH keypair dedicated to AUR pushes:
   ```sh
   ssh-keygen -t ed25519 -C 'aur-push' -f ~/.ssh/aur-push
   ```
3. Register the **public** half on your AUR account:
   <https://aur.archlinux.org/account/> → "SSH Public Key".
4. Store the **private** half somewhere you actually recover from — your
   password manager's "secure note" / file-attachment slot is the right
   place. Don't leave it sitting unencrypted in `~/.ssh/` if the host
   doesn't already keep `~/.ssh/` encrypted.

Per-package (do once for each new AUR package you publish):

1. Decide the package name. Convention for binary releases is
   `<project>-bin`; source-built packages use the bare name.
2. Initialize the AUR repo via SSH:
   ```sh
   git clone ssh://aur@aur.archlinux.org/<PKGNAME>-bin.git
   ```
   The first clone will be empty — that's expected; AUR creates the repo
   on first SSH access. The publish-aur job pushes the initial PKGBUILD on
   your first tag.

## 2. GitHub repo secrets

Settings → Secrets and variables → Actions → New repository secret. Add
**all four**:

| Secret name | Value | Where it comes from |
|---|---|---|
| `AUTOMERGE_PAT` | A personal access token | Create at <https://github.com/settings/tokens>. Scope: `repo` + `workflow`. Why a PAT instead of `GITHUB_TOKEN`: GitHub suppresses workflow triggers for `GITHUB_TOKEN`-actored events, which breaks the auto-merge → release-please chain. |
| `AUR_USERNAME` | Author name on AUR commits | Whatever name you want on the AUR git log (e.g. your AUR account name or full name). |
| `AUR_EMAIL` | Author email on AUR commits | Same email registered on your AUR account, or the GitHub noreply form. |
| `AUR_SSH_PRIVATE_KEY` | Private key from step 1 | Paste the entire contents of `~/.ssh/aur-push` (the file without `.pub`) — including the `-----BEGIN OPENSSH PRIVATE KEY-----` header and trailing newline. Retrieve from your password manager. |

**Never paste these into any file in this repo.** They live in GitHub
secrets only. The workflows reference them as `${{ secrets.NAME }}`.

## 3. GitHub repo settings

Settings → General:

- ✅ **Allow auto-merge** — required for `auto-merge.yml` to enable
  auto-merge on the release PR.
- ✅ **Automatically delete head branches** — keeps PR branches tidy
  after merge.

Settings → Actions → General → Workflow permissions:

- ✅ **Read and write permissions** — `release-please` needs write.
- ✅ **Allow GitHub Actions to create and approve pull requests**
  — release-please opens PRs.

Settings → Branches → Add branch protection rule for `main`:

- Branch name pattern: `main`
- ✅ **Require a pull request before merging** (no direct pushes)
- ✅ **Require status checks to pass before merging** → add `test` to
  the required checks list (this is the check name produced by
  `.github/workflows/test.yml`).
- ✅ **Do not allow bypassing the above settings** (optional but
  recommended — keeps you honest).

## 4. Fill in the stubs

Two workflow files in `.github/workflows/` will `exit 1` until you fill
them in. Both are language-specific:

- **`build.yml`** — produce a `dist/` artifact (per-platform archives +
  `checksums.txt`). Contract is documented at the top of the file.
- **`test.yml`** — your unit/integration tests + linters + formatters.
  Must produce a status check named `test` (the default name from the
  workflow's `name:` field is fine).

Plus customize:

- **`packaging/aur/PKGBUILD`** — replace every `<PLACEHOLDER>`. Maintain
  the `package()` body to match what your `build.yml` puts in the
  archive.
- **`release-please-config.json`** — change `"release-type": "simple"`
  to your language's value (`"go"`, `"rust"`, `"node"`, etc.) and update
  `"package-name"`. The `"simple"` default works for any project but
  language-specific types know about idiomatic version-file locations.
- **`LICENSE`** — replace the `<YEAR> <YOUR NAME>` line.
- **`CLAUDE.md`** — adjust the project name and any project-specific
  conventions.
- **`README.md`** — write what the project actually is.

## 5. First release

Once 1–4 are done:

1. Make a `feat:` commit on a branch and open a PR.
2. Auto-merge enables; once `test` is green it merges.
3. Release-please opens `chore(main): release 0.1.0`. Auto-merge enables.
4. That release PR merges → tag `v0.1.0` is pushed → `release.yml` fires.
5. `build.yml` runs → `gh-release` uploads the artifacts → `publish-aur`
   pushes the first PKGBUILD.

If anything in this chain fails, fix the failing workflow and re-run via
the Actions tab → workflow_dispatch on `release-please.yml` if needed.

## 6. Delete this file

Once the first release ships clean, open a PR that deletes
`BURN-AFTER-READING.md`. Future cloners shouldn't have to ask whether
setup is done.
