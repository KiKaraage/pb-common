# Automation Audit — Implementation Roadmap

> **Status as of 2026-06-11:** All phases except Phase 5 (ISO, `iso` repo out of scope) are deployed. The artifact YAML files in this directory are live in their target repos — do **not** re-deploy them. They are kept as reference only.
>
> Prioritized by impact. Highest automation gain first.

## Quick Reference

| Phase | Status | Artifact(s) | Deployed in |
|---|---|---|---|
| **1** | ✅ | `actions-v1-tag-update.yml` | [actions#154](https://github.com/projectbluefin/actions/pull/154) |
| **2** | ✅ | `cliff.toml` + `release-with-cliff.yml` | [common#592](https://github.com/projectbluefin/common/pull/592) |
| **3** | ✅ | `retry-action.yml` | [actions#155](https://github.com/projectbluefin/actions/pull/155) |
| **4** | ✅ | `check-token-health-action.yml` | [actions#156](https://github.com/projectbluefin/actions/pull/156) |
| **5** | 🔴 | `iso-auto-rebuild.yml` + `iso-dispatch-snippet.yml` | Out of scope — iso repo |
| **6** | ✅ | `build-upgraded.yml` | [common#595](https://github.com/projectbluefin/common/pull/595) |
| **7** | ✅ | `dakota-cache-warm.yml` | [dakota#782](https://github.com/projectbluefin/dakota/pull/782) |

---

## Phase 1: v1 Tag Auto-Update ✅ DEPLOYED

**What:** After every merge to `main` in `projectbluefin/actions`, auto-update the `v1` tag.

**Steps:**
1. Copy `actions-v1-tag-update.yml` to `actions/.github/workflows/`
2. Pin the `actions/checkout` SHA (already done in artifact)
3. PR to `actions` → human merges (actions merge is intentional human gate)
4. After merge: the workflow will self-update v1 going forward

**Validation:** After first merge, check `git log v1 --oneline -1` matches latest main.

**Risk:** None. Worst case: tag points to main anyway (which is the desired state).

---

## Phase 2: git-cliff Changelog ✅ DEPLOYED

**What:** Replace raw `git log` in `common/release.yml` with structured git-cliff output.

**Steps:**
1. Copy `cliff.toml` to `common/` root
2. Replace `common/.github/workflows/release.yml` with `release-with-cliff.yml`
3. Test: `git cliff --latest` locally to verify output
4. Push directly to main (doc-adjacent: the release workflow is a factory improvement)

**Validation:** Run `git cliff --latest` and verify it produces categorized output.

**Risk:** Low. `git-cliff` binary must be available at runtime — handled by `taiki-e/install-action`.

---

## Phase 3: Retry Composite Action ✅ DEPLOYED

**What:** Add a reusable retry-with-backoff action to `projectbluefin/actions`.

**Steps:**
1. Create `actions/actions/retry/action.yml` from `retry-action.yml` artifact
2. PR to `actions` → human merges
3. After merge + v1 tag update: available to all repos

**Validation:** Test with intentional failure: `command: "false"` should retry 3 times then fail.

**Risk:** None. Opt-in — no existing workflow changes until repos adopt it.

---

## Phase 4: Token Health Check ✅ DEPLOYED

**What:** Composite action that validates token auth/scopes/rate-limit at workflow start.

**Steps:**
1. Create `actions/actions/check-token-health/action.yml` from artifact
2. PR to `actions` → human merges
3. Add to `reusable-renovate.yml` as first step (validates RENOVATE_TOKEN)

**Validation:** Test with expired/revoked token → should fail with clear error message.

**Risk:** None. Fails fast with actionable error instead of cryptic downstream failure.

---

## Phase 5: ISO Auto-Rebuild 🔴 OUT OF SCOPE

**What:** Automatically rebuild ISOs when :stable is promoted.

**Steps:**
1. Create dispatch token (GitHub App or PAT with `repo` scope on `iso` repo)
2. Store as org secret: `ISO_DISPATCH_APP_ID` + `ISO_DISPATCH_PRIVATE_KEY` (or `ISO_DISPATCH_TOKEN`)
3. Add `iso-auto-rebuild.yml` to `iso/.github/workflows/`
4. Add dispatch job from `iso-dispatch-snippet.yml` to:
   - `bluefin/.github/workflows/execute-release.yml`
   - `bluefin-lts/.github/workflows/execute-release.yml`
5. Test with `workflow_dispatch` → verify ISO build triggers

**Validation:** Trigger a test promotion → verify ISO workflow fires within 5 minutes.

**Design decision required:** Token type (App vs. PAT). App is more secure but more setup.

**Risk:** Medium. Cross-repo dispatch requires correct token permissions. Test in staging first.

---

## Phase 6: Supply Chain Upgrade ✅ DEPLOYED

**What:** Keyless signing + SBOM + SLSA L2 + CVE scan for `common/build.yml`.

**Deployed:** [common#595](https://github.com/projectbluefin/common/pull/595) — keyless OIDC live, `SIGNING_SECRET` removed from org secrets. `build-upgraded.yml` is now the live `build.yml`.

**Bugs fixed during deployment:**
- **Permissions starvation:** `id-token: write` was missing at the calling workflow level — OIDC tokens were silently empty, causing cosign to fail with an unhelpful auth error.
- **Wrong `workflow_run` trigger:** workflow was listening for `workflow_run: [build]` but the actual workflow name is `Build and Push` — trigger never fired.

**Verification:**
```bash
# Verify keyless signature
cosign verify \
  --certificate-identity-regexp "https://github.com/projectbluefin/common/" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/projectbluefin/common:latest

# Verify SBOM attached
oras discover ghcr.io/projectbluefin/common:latest

# Verify attestation
gh attestation verify oci://ghcr.io/projectbluefin/common:latest --repo projectbluefin/common
```

---

## Phase 7: Dakota Cache Warm ✅ DEPLOYED

**What:** Scheduled builds to keep BST remote cache hot.

**Deployed:** [dakota#782](https://github.com/projectbluefin/dakota/pull/782) — scheduled cache-warm workflow live.

**Validation:** After first run, check if subsequent `build.yml` runs complete faster.

---

## Dependency Graph

```
Phase 1 (v1 tag)        ✅ deployed — actions#154
Phase 2 (git-cliff)     ✅ deployed — common#592
Phase 3 (retry action)  ✅ deployed — actions#155
Phase 4 (token health)  ✅ deployed — actions#156
Phase 5 (ISO dispatch)  🔴 out of scope — iso repo not authorized
Phase 6 (supply chain)  ✅ deployed — common#595
Phase 7 (cache warm)    ✅ deployed — dakota#782
```

All deployable phases are live. Phase 5 remains blocked on iso repo authorization.

---

## Success Metrics

| Before Audit | After All Phases (current) |
|---|---|
| 91% automated | ~97% automated |
| 11 manual touchpoints | 4 intentional human gates |
| Key-based signing | Keyless OIDC signing ✅ |
| No SBOM/provenance | SBOM + SLSA L2 ✅ |
| Manual ISO builds | Manual ISO builds (Phase 5 out of scope) |
| Raw git log changelog | Structured changelog ✅ |
| No retry/self-heal | Retry + token health ✅ |

---

## Deployed — Tracking Issues Filed and Closed

All tracking issues below were filed in `projectbluefin/common` and resolved:

1. `feat(ci): auto-update v1 tag in actions repo` — ✅ [actions#154](https://github.com/projectbluefin/actions/pull/154)
2. `feat(ci): integrate git-cliff for structured changelogs` — ✅ [common#592](https://github.com/projectbluefin/common/pull/592)
3. `feat(ci): add retry composite action to actions` — ✅ [actions#155](https://github.com/projectbluefin/actions/pull/155)
4. `feat(ci): add token health check action` — ✅ [actions#156](https://github.com/projectbluefin/actions/pull/156)
5. `feat(ci): automate ISO rebuilds on stable promotion` — 🔴 out of scope (iso repo)
6. `feat(ci): supply chain upgrade (keyless OIDC + SBOM + SLSA L2)` — ✅ [common#595](https://github.com/projectbluefin/common/pull/595)
7. `feat(ci): dakota BST cache-warm workflow` — ✅ [dakota#782](https://github.com/projectbluefin/dakota/pull/782)
