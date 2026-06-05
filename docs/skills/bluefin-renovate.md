---
name: bluefin-renovate
description: Bluefin Renovate dependency update handling — use when reviewing or merging Renovate PRs, configuring Renovate behavior, or understanding how automated dependency updates work across bluefin repos.
---

# Bluefin Renovate Skill

## Powerlevel

- **Level:** 1


Handles automated dependency updates via Renovate across Bluefin repositories.

Load with: `cat ~/src/skills/bluefin-renovate/SKILL.md`

## When to Use

- A **major** version Renovate PR did not automerge and needs review
- Configuring Renovate behavior in `projectbluefin/renovate-config`
- Understanding why Renovate did or did not open a PR
- Checking grouping, scheduling, or version-bump strategy
- Debugging a Renovate config validation error

## When NOT to Use

- **Merging digest/pin/patch/minor Renovate PRs** — these automerge when CI passes; do not touch them
- Manually bumping a dependency version yourself (let Renovate handle it)
- General package additions — use `cat ~/src/skills/bluefin-packages/SKILL.md`
- CI failures caused by a Renovate PR — use `cat ~/src/skills/bluefin-ci/SKILL.md`

## How It Works

Renovate scans repos for outdated dependencies and opens PRs automatically.
Config lives at `castrojo/renovate-config` (upstream: `projectbluefin/renovate-config`).

## Automerge Behavior (org-wide)

Configured in `projectbluefin/renovate-config` → `org-inherited-config.json`:

| Update type | Automerge? |
|---|---|
| `digest` | ✅ Yes — merges when CI passes |
| `pin` | ✅ Yes — merges when CI passes |
| `patch` | ✅ Yes — merges when CI passes |
| `minor` | ✅ Yes — merges when CI passes |
| `major` | ❌ No — requires human/agent review |

Schedule: Renovate runs **Mondays 00:00–03:00 UTC** only.
`rebaseWhen: "never"` — Renovate will not rebase PRs when the base branch updates.

**Agents: do not spend tokens merging digest/pin/patch/minor PRs. They handle themselves.**

## Renovate PR Workflow — major updates only

1. Renovate opens a PR against `testing` (configured via `baseBranchPatterns: ["testing"]` in `.github/renovate.json5`)
2. CI runs — `validate` check must pass
3. Review breaking changes manually
4. Approve with `gh pr review $N --approve`
5. Squash-merge directly: `gh pr merge $N --squash` (no merge queue on `testing`)

⚠️ **If Renovate PRs are targeting `main` instead of `testing`**, the `baseBranchPatterns` field in `.github/renovate.json5` is wrong. Fix it:
```json5
"baseBranchPatterns": ["testing"],  // NOT "main"
```
PRs already in the `main` merge queue cannot have their base changed — they must be dequeued first via GraphQL:
```bash
node_id=$(gh pr view $N --repo projectbluefin/bluefin --json id --jq .id)
gh api graphql -f query="mutation { dequeuePullRequest(input: { id: \"${node_id}\" }) { mergeQueueEntry { id } } }"
# Then: gh pr edit $N --base testing
```

## Common Renovate Sources

| Source | What it updates |
|---|---|
| Containerfile `FROM` | Base image versions |
| GitHub Actions `uses:` | Action versions |
| Brew formulas | Formula versions in tap repos |
| `.github/workflows/` | Runner images, action pins |

## Renovate Config Repo

`~/src/renovate-config` — clone of `projectbluefin/renovate-config`

This is where all org-wide Renovate behavior lives. Clone directly (no fork needed for maintainers):
```bash
git clone https://github.com/projectbluefin/renovate-config.git ~/src/renovate-config
```

Changes to org-wide behavior go in `org-inherited-config.json`. Repo-specific overrides stay in the repo's own `renovate.json` or `.github/renovate.json5`.

## Grouping and Scheduling

Renovate groups minor/patch updates. Major version bumps get separate PRs.
- **Schedule:** weekly, Monday 00:00–03:00 UTC (`"* 0-3 * * 1"`)
- **Automerge:** digest/pin/patch/minor auto-land when CI passes (org-wide default)
- **Rebase:** never — PRs are not rebased on base-branch updates

## ⛔ Lint Before Every Commit (Hard Rule)

Any time you touch `org-inherited-config.json`, `renovate-config.json`, or `renovate.json`, run the validator locally **before** committing:

```bash
cd ~/src/renovate-config
npx --yes renovate-config-validator --strict org-inherited-config.json
```

If `npx` is unavailable or slow, run it via container:

```bash
docker run --rm -v "$PWD":/work -w /work renovate/renovate renovate-config-validator --strict org-inherited-config.json
```

**Do not push a branch without a clean validator pass.** The CI validate step runs `renovate-config-validator --strict` with the latest Renovate — a local pass prevents embarrassing CI failures on the PR.

Known validator gotcha: `managerFilePatterns` is the correct field name inside `customManagers` (not `fileMatch`). Verify against the Renovate docs if the validator rejects a field — the error message names the offending field explicitly.

**Node version gotcha for CI `ci.yml` in `projectbluefin/renovate-config`**: The `Setup Node.js` step must use `node-version: '24'` (not `latest`). Renovate v43+ declares `engines: {node: '^24.11.0'}`. `node-version: latest` resolves to Node 25.x which falls outside that range — npm then falls back to an old Renovate version that incorrectly rejects `managerFilePatterns`. Fixed in `feature/fix-ci-validator-node-version`.

**validate-renovate.yml pattern (authoritative)**: The upstream Renovate docs recommend one command, no wrapper actions:
```yaml
- name: Validate Renovate config
  run: npx --yes --package renovate -- renovate-config-validator --strict
```
- No filename argument — auto-discovers `.github/renovate.json5` from default locations and validates as repo config (not global)
- Passing a filename causes the validator to treat the file as global self-hosted config → fails on valid repo fields
- Do NOT use `suzuki-shunsuke/github-action-renovate-config-validator` — it is a third-party wrapper not mentioned in upstream docs and has broken twice on renovate version changes
- Do NOT pin `@latest` or `@43.x.y` — the unversioned `renovate` package resolves to latest per npm

**Renovate App config error (issue #325 pattern)**: When `mergeraptor` opens "Action Required: Fix Renovate Configuration", the config has a validation error. Run `renovate-config-validator --strict` locally to get the exact error — it names the offending field. Common cause: field renamed between Renovate versions (e.g. `fileMatch` → `managerFilePatterns`).

## Dakota tarball automation (UNSAFE without post-update hook)

`track-bst-sources.yml` in dakota manually handles 7 GitHub release tarballs (brew-tarball, wallpapers, fzf, glow, gum, gtk4-layer-shell, tailscale, uupd) that are **not** covered by Renovate. Do not add naive Renovate regex managers for these — BST elements require the `ref: sha256` to match the fetched tarball. Renovate can only update the version string; the sha256 would become stale and break builds.

Safe automation requires `postUpgradeTasks` (Renovate exec platform) to run `bst source track` after version bump, or an atomic wrapper script. See castrojo/dakota issue #162.

`jetbrains-mono-nerd-font` IS already Renovate-covered. `git_repo` BST refs are intentionally BST-managed.

## Learnings

### Renovate PyPI URL bug — path hash not updated (2026-05-22, dakota PR #478)

Renovate bumps BST `tar` elements that use `pypi:` URLs by swapping the version string in the **filename only**. It does NOT update the content-hash path prefix that PyPI encodes in every URL.

PyPI URL structure:
```
https://files.pythonhosted.org/packages/<sha256[:2]>/<sha256[2:4]>/<sha256[4:]>/<filename>
```
The 64-char hex path is the SHA256 of that specific tarball. A new release has a completely different SHA256, so its URL is at a different path. Renovate's naive version-string substitution produces a 404.

**Symptom:** `validate` fails with `HTTP Error 404: Not Found` on the PyPI URL.

**Fix:** Get the correct URL and ref from PyPI's JSON API:
```bash
curl -s "https://pypi.org/pypi/<package-name>/<version>/json" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for url in data['urls']:
    if url['filename'].endswith('.tar.gz'):
        print('URL:', url['url'])
        print('SHA256:', url['digests']['sha256'])
"
```
Then update the BST element manually:
```yaml
- kind: tar
  url: pypi:<path-from-url>/filename-X.Y.Z.tar.gz
  ref: <sha256-from-digests>
```
The `ref` for a BST `tar` source is the SHA256 of the tarball content — same value as PyPI's `digests.sha256`.

**Applies to:** any BST element using `pypi:` URL scheme. Currently: `elements/plugins/buildstream-plugins-community.bst`. Close the bad Renovate PR and open a replacement.
