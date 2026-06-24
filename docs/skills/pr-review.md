---
name: pr-review
version: "1.0"
last_updated: 2026-06-24
tags: [review, testing, contributing]
description: "Reviewer's guide for PRs in projectbluefin/common — PR type taxonomy, per-type review checklist, how to use the lab for verification, CI gate interpretation, and lab test patterns. Use when reviewing any incoming PR to common."
metadata:
  type: procedure
---

# PR Review Guide — projectbluefin/common

`projectbluefin/common` is the shared OCI layer consumed by every downstream variant (bluefin, bluefin-lts, dakota). A broken `system_files/shared/` change cascades to all three simultaneously. Consistent, thorough review prevents those regressions from reaching users.

This guide documents PR type taxonomy, per-type review checklists, lab testing patterns, CI gate interpretation, and test quality standards — built from live review sessions on PRs #760, #767, #768, #769, and #785.

---

## PR type taxonomy

Identify the PR type first. It determines blast radius, review depth, and whether lab testing is needed.

| Type | Paths touched | Blast radius | Lab needed? |
|---|---|---|---|
| `systemd unit` | `system_files/shared/**/*.service`, `system_files/shared/**/*.timer`, `system_files/shared/**/*.path` | ALL variants | Yes |
| `systemd unit (nvidia)` | `system_files/nvidia/**/*.service` | nvidia images only | nvidia VM |
| `shell script` | `system_files/shared/usr/libexec/**`, `system_files/shared/usr/bin/**` | ALL variants | If behavior-changing |
| `dconf / GSettings` | `system_files/shared/**/*.gschema.override`, `system_files/shared/**/*.d/*.conf` | ALL variants | Optional |
| `OEM hardware hook` | `system_files/shared/usr/share/ublue-os/user-setup.hooks.d/**`, `system_files/shared/usr/share/ublue-os/system-setup.hooks.d/**` | Scoped to DMI gate | If DMI gate changed |
| `test addition` | `tests/**` | None (tests only) | No — run `just test` |
| `CI workflow` | `.github/workflows/**` | CI pipeline | No, but needs maintainer review |
| `doc / skill update` | `docs/**`, `AGENTS.md` | None | No — doc-only exception, push direct to main |
| `Containerfile` | `Containerfile` | ALL variants | Yes |

---

## Universal checklist

Apply to every PR regardless of type:

- [ ] `just check` passes (Justfile lint)
- [ ] `pre-commit run --all-files` passes (JSON/YAML/TOML hygiene, actionlint, SHA pinning)
- [ ] PR title follows Conventional Commits format (`feat:`, `fix:`, `chore(deps):`, `docs:`, etc.)
- [ ] All `uses:` references to external GitHub Actions are SHA-pinned with a version comment — no `@main`, `@latest`, or `@v*` floating tags
- [ ] If `system_files/shared/`: blast radius acknowledged — change affects bluefin, bluefin-lts, AND dakota
- [ ] `system_files/` changes have lab verification or passing E2E CI before merge

---

## Per-type review checklists

### systemd unit

- [ ] `WantedBy=` target matches the intended boot path:
  - `multi-user.target` for network services, update daemons, background tasks — runs on all boots including headless
  - `graphical.target` only if the service genuinely requires a graphical session (e.g., GNOME-specific D-Bus activation)
  - **Do not downgrade `multi-user.target` to `graphical.target`** unless the service truly needs GNOME — this silently drops the service on non-graphical boots (PR #767: `flatpak-appstream-firstboot.service` was incorrectly changed to `graphical.target`)
- [ ] `StartLimitBurst=` and `StartLimitIntervalSec=` are in `[Unit]`, not `[Service]` — systemd ignores them in `[Service]` (PR #767: these were in wrong section)
- [ ] `Restart=on-failure` interaction with conditions:
  - `ConditionACPower=true` — if AC condition fails (not met), systemd exits `SERVICE_SKIP_CONDITION` (exit 0 from systemd's perspective), which does NOT trigger `Restart=on-failure`. Transient condition-skip is silent.
  - `ExecCondition=` — same behavior: skip-condition exits are not failures. A service that installs, then fails a later step, may be silently skipped on restart if the install step's check returns non-zero on re-entry (PR #769: GL runtime already installed → ExecCondition exits non-zero → unit skips on Restart)
- [ ] `StartLimitBurst=1` combined with rate-limiting window: if the service can be legitimately triggered multiple times in the window (e.g., brief AC plug-in fires udev → service starts → AC unplugged before activation → real plug-in later is silently dropped), consider whether the rate limit is correct (PR #768)
- [ ] `After=` includes ordering dependencies that `Wants=` alone does not provide — `Wants=network-online.target` does not guarantee network is up before the unit starts; add `After=network-online.target` if ordering matters (PR #768: `uupd.timer` missing `After=`)
- [ ] `TimeoutStartSec=` is adequate for the work being done (e.g., large flatpak updates need 900s+, not 600s)
- [ ] `RemainAfterExit=yes` is appropriate — correct for one-shot setup services that should stay "active" after completion
- [ ] `[Install]` section: its absence is intentional for udev-started units (document in review); its presence is required for timer-started or manually-enabled units
- [ ] **Enable mechanism present**: if `[Install]` / `WantedBy=` is set, confirm a preset file (`.preset`) or systemctl enable call will activate it in the built image — a unit file without an enable mechanism ships as a no-op (PR #767: preset existed in source but image pre-dated its merge, so service was disabled)
- [ ] Service name reflects current behavior — if a "firstboot" service was generalized to run on every boot, the name should be updated (PR #767: service is no longer firstboot-only)

### shell script

- [ ] `shellcheck` passes — at minimum, run `shellcheck -S warning <file>`
- [ ] SC1091 suppression (`. /path/to/sourced/file`): use `# shellcheck source=/path` annotation, not a blanket disable
- [ ] Variable quoting — unquoted variables in conditionals and loops are common failure modes
- [ ] Error handling — `set -euo pipefail` or explicit checks; silent failures in hooks cause hard-to-debug regressions
- [ ] D-Bus / systemd calls in user hooks: `systemctl --user` needs `$DBUS_SESSION_BUS_ADDRESS` — confirm environment is available at hook execution time

### dconf / GSettings

- [ ] Override file AND lock file change together — see [`dconf-consistency.md`](dconf-consistency.md)
- [ ] Lock file path matches override key exactly (key in `00-common` → lock entry in `00-common.conf`)
- [ ] `just check` catches parity mismatches via the dconf consistency checker

### OEM hardware hook

See the full OEM pattern section below. Quick checklist:

- [ ] Version-script guard wraps idempotent-but-not-safe work; purely idempotent work lives outside it
- [ ] DMI gate is scoped to the exact target hardware: `chassis_vendor` + `sys_vendor` + `product_name` (all three)
- [ ] Version bump acknowledged: existing users re-run the versioned block — all code inside must be safe to re-run (brew install is idempotent; dconf writes and systemctl enables are safe to re-run)
- [ ] WirePlumber fragments: written to user fragment dir (`~/.config/wireplumber/wireplumber.conf.d/`), not global system dir
- [ ] Hook is executable and follows naming convention (`NN-description`)
- [ ] **Brew availability early-exit**: brew-independent payloads (config file copies, WirePlumber fragments, etc.) are placed BEFORE the `BREW_BIN` check, not after it — if they are after, they silently don't run on first login when brew isn't yet available (PR #760)

### test addition

See the test quality checklist below. Quick checklist:

- [ ] Full argv asserted, not substring membership — catches missing flags, wrong spawn wrapper, wrong ordering
- [ ] Key kwargs verified (`start_new_session=True` for background spawns)
- [ ] Mock expands full received command line in bats (not a collapsed label like `"mock: grubby update"`)
- [ ] Test covers failure path, not only happy path
- [ ] `just test` passes locally

### CI workflow

- [ ] All external `uses:` SHA-pinned — pre-commit enforces this, but verify in review
- [ ] New composite actions sourced from `projectbluefin/actions` where a reusable exists — do not inline logic that already lives there
- [ ] No inline supply chain steps (signing, SBOM, provenance) — consume composite actions from `projectbluefin/actions` instead
- [ ] Sensitive paths (secrets, GHCR push, cosign) need maintainer eyes
- [ ] Workflow name follows existing conventions in the repo

---

## Lab testing guide

See [`lab-testing.md`](lab-testing.md) for the full runbook. Summary for PR review:

### When is lab testing required?

| Change type | Lab required? |
|---|---|
| `system_files/shared/` — systemd units | Yes — all 3 variants |
| `system_files/shared/` — scripts / hooks | Yes if behavior-changing |
| `system_files/nvidia/` | Yes — **nvidia image variant only** (non-nvidia baseline only confirms service is absent; use bluefin-dx or equivalent to verify actual changes) |
| `tests/` only | No — `just test` locally |
| `docs/` only | No |
| `Containerfile` | Yes — image must compose |

### Scope by changed path

- `system_files/shared/` → test on **bluefin**, **bluefin-lts**, and **dakota** images
- `system_files/nvidia/` → test on **bluefin-dx** (or any nvidia-enabled image)
- `system_files/bluefin/` → **bluefin** and **bluefin-lts** only (not dakota)

### What to verify

```bash
# After booting the lab VM, check for failures:
systemctl --failed
journalctl -p warning -b

# For a specific unit:
systemctl cat <unit-name>.service
systemctl status <unit-name>.service
journalctl -u <unit-name>.service -b
```

> ⚠️ **Always check `systemctl is-enabled` in the baseline.** A clean boot and empty `systemctl --failed` does NOT mean the service is working — it may simply not be enabled. If a unit is disabled, it never runs and produces no journal output. This is silent: no errors, no warnings, just a no-op.
>
> ```bash
> systemctl is-enabled <unit-name>.service
> # "disabled" means it will never run at boot regardless of WantedBy
> ```
>
> If the service is disabled in the baseline, the review must also confirm there is a preset file or explicit `WantedBy=` + want symlink that will enable it in the built image. A unit file shipping without an enable mechanism means the change does nothing for users until the preset is also present.
>
> **Common scenario:** a preset file is added in the same or a prior PR but the current testing image was built before it merged — the service appears disabled in the lab even though the preset is correct in source. Always cross-check the preset file in the repo against the running image state.

### Expected QEMU noise (ignore these)

- `nvidia-persistenced` errors — NVIDIA driver not present in QEMU
- `systemd-oomd` warnings — memory pressure in constrained VMs
- VirtIO / KVM device messages

### Baseline vs delta

**Always establish a baseline before the PR merges.** Boot the current testing image, record the state of the units/files the PR touches, then re-verify after rebuild. This catches unintended regressions and confirms all new artifacts landed.

**Step 1 — collect baseline** (pre-merge, on current testing image):

```bash
# For a systemd unit PR — capture current state of every touched unit/file
systemctl cat uupd.timer 2>/dev/null || echo "MISSING"
systemctl cat uupd.service 2>/dev/null || echo "MISSING"
cat /usr/lib/systemd/system/uupd.service.d/10-bluefin.conf 2>/dev/null || echo "MISSING"
cat /usr/lib/udev/rules.d/99-uupd-on-ac.rules 2>/dev/null || echo "MISSING"
systemctl cat uupd-on-ac.service 2>/dev/null || echo "MISSING"
```

**Step 2 — merge PR, wait for rebuild** (`bluefin:testing` rebuilds automatically on push to main)

**Step 3 — verify delta** (post-merge, on new testing image):

```bash
# Confirm every expected artifact is present and has the right content
systemctl cat uupd.timer          # check OnCalendar value
systemctl cat uupd.service        # should still be static (no [Install])
systemctl is-enabled uupd.timer   # should still be enabled
cat /usr/lib/systemd/system/uupd.service.d/10-bluefin.conf  # new drop-in
cat /usr/lib/udev/rules.d/99-uupd-on-ac.rules               # new udev rule
systemctl cat uupd-on-ac.service                             # new unit
```

#### Worked example — PR #768 (uupd AC-aware scheduling)

**Baseline state** (bluefin:testing before PR, workflow `pr768-uupd-baseline-lxknq`):

| Artifact | Baseline state |
|---|---|
| `uupd.timer` | **Exists** — daily at 04:00, `Persistent=true`, `RandomizedDelaySec=15m` |
| `uupd.service` | Exists, static (no `[Install]`), timer-driven — correct |
| `uupd.service.d/10-bluefin.conf` | **MISSING** — PR adds it |
| `99-uupd-on-ac.rules` | **MISSING** — PR adds it |
| `uupd-on-ac.service` | **MISSING** — PR adds it |
| `uupd-manual.service` | Exists, untouched by PR |
| `ConditionACPower=` on uupd.service | **Absent** — drop-in adds it |

PR #768 **replaces** the existing daily timer with a 6h schedule — this is a deliberate behavior change, not an error. Knowing the baseline prevents false-alarming on "timer changed".

**Post-merge verification checklist for PR #768:**

```bash
# 1. Timer fires every 6h
systemctl cat uupd.timer | grep OnCalendar
# expected: OnCalendar=*-*-* 00,06,12,18:00

# 2. Drop-in adds ConditionACPower
cat /usr/lib/systemd/system/uupd.service.d/10-bluefin.conf | grep ConditionACPower
# expected: ConditionACPower=true

# 3. udev rule present
ls -la /usr/lib/udev/rules.d/99-uupd-on-ac.rules

# 4. AC-triggered unit present
systemctl cat uupd-on-ac.service

# 5. Timer still enabled, uupd.service still static
systemctl is-enabled uupd.timer       # enabled
systemctl cat uupd.service | grep '\[Install\]'  # should be absent (timer-driven)
```

#### Worked example — PR #769 (NVIDIA flatpak runtime sync)

**Baseline state** (bluefin:testing non-nvidia, workflow `pr769-nvidia-check-thx78`):

| Artifact | Baseline state |
|---|---|
| `ublue-nvidia-flatpak-runtime-sync.service` | **ABSENT** — nvidia overlay not applied to non-nvidia image |
| `/sys/module/nvidia/version` | **NOT FOUND** — correct for QEMU |
| nvidia units in `systemctl --failed` | None |

**Verdict:** Green baseline. The service's `ConditionPathExists=/sys/module/nvidia/version` means PR changes (`TimeoutStartSec` 600→900, added `flatpak update`) are completely inert on non-nvidia images. Zero regression risk to non-nvidia users.

> ⚠️ **NVIDIA post-merge testing requires an nvidia image variant.** The non-nvidia baseline only confirms the service is absent as expected. To verify the actual changes landed, use a bluefin-dx or other nvidia-enabled image — see the nvidia section below.

**Post-merge verification checklist for PR #769** (must run on a **nvidia image build**, not baseline non-nvidia):

```bash
# 1. TimeoutStartSec bumped to 900
systemctl cat ublue-nvidia-flatpak-runtime-sync.service | grep TimeoutStartSec
# expected: TimeoutStartSec=900

# 2. flatpak update step present in the sync script
grep "flatpak update" /usr/libexec/ublue-nvidia-flatpak-runtime-sync
# expected: at least one match

# 3. Service not in failed state on first boot with nvidia
systemctl --failed | grep nvidia
# expected: no output
```

#### Worked example — PR #767 (flatpak appstream every-boot)

**Baseline state** (bluefin:testing, 3 workflows, all Succeeded):

| Artifact | Baseline state |
|---|---|
| `flatpak-appstream-firstboot.service` | Exists, unit file matches pre-PR content |
| `systemctl is-enabled flatpak-appstream-firstboot.service` | **`disabled`** — no want symlink anywhere |
| Journal for the service | `-- No entries --` — never ran at boot |
| `ConditionPathExists=!/var/lib/flatpak/.appstream-refreshed` | Present (firstboot guard, PR removes it) |
| `ExecStartPost=/bin/touch ...` | Present (flag file creator, PR removes it) |
| `StartLimitBurst=3` location | In `[Service]` — misplaced (PR correctly moves to `[Unit]`) |
| `/var/lib/flatpak/.appstream-refreshed` flag file | Absent (fresh VM — correct) |
| Preset `02-flatpak-appstream-firstboot.preset` | In repo source, but **not yet active** in this image build |

**Critical finding:** The service is **disabled** in the current testing image. The preset file exists in the repo but the image was built before it merged — so neither the old firstboot-only behavior nor the new every-boot behavior is active or verifiable yet. A clean lab boot here produces no journal output and no failures, but it is entirely a no-op — not a green signal.

**Open question for PR author:** Is the preset landing in the same PR? If not, the every-boot behavior won't activate until a subsequent build includes the preset.

**Post-merge verification checklist for PR #767** (requires a rebuilt image that includes the preset):

```bash
# 1. Service is now enabled
systemctl is-enabled flatpak-appstream-firstboot.service
# expected: enabled

# 2. Firstboot guard removed — no ConditionPathExists line
systemctl cat flatpak-appstream-firstboot.service | grep ConditionPathExists
# expected: no output

# 3. StartLimitBurst in [Unit] not [Service]
systemctl cat flatpak-appstream-firstboot.service
# expected: StartLimitBurst=3 appears after [Unit] header, not after [Service] header

# 4. WantedBy target confirmed (verify graphical.target issue was addressed)
systemctl cat flatpak-appstream-firstboot.service | grep WantedBy
# expected: WantedBy=multi-user.target

# 5. Service ran this boot
journalctl -u flatpak-appstream-firstboot.service -b
# expected: entries showing appstream refresh

# 6. No flag file created (every-boot, not one-shot)
ls /var/lib/flatpak/.appstream-refreshed 2>/dev/null && echo "EXISTS" || echo "absent (correct)"
# expected: absent (correct)
```

---

## Quick-start lab test YAML

Copy-paste these to submit targeted lab tests via the Argo MCP or `kubectl apply`. Always lint first with `argo-mcp-lint_workflow` before submitting.

### systemd unit / shared script — all 3 variants

Submit one per variant. Use `smoke` suite for a fast first pass; add `system` if you need full bootc contract verification.

```yaml
# bluefin:testing — smoke + system
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: pr-lab-bluefin-
  namespace: argo
spec:
  workflowTemplateRef:
    name: bluefin-qa-pipeline
  arguments:
    parameters:
    - name: image
      value: ghcr.io/projectbluefin/bluefin
    - name: image-tag
      value: testing
    - name: suites
      value: smoke,system
    - name: namespace
      value: bluefin-test
```

For lts: set `image: ghcr.io/projectbluefin/bluefin-lts` and `image-tag: lts-testing`.

### NVIDIA overlay — non-nvidia baseline check

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: pr-lab-nvidia-baseline-
  namespace: argo
spec:
  workflowTemplateRef:
    name: bluefin-qa-pipeline
  arguments:
    parameters:
    - name: image
      value: ghcr.io/projectbluefin/bluefin
    - name: image-tag
      value: testing
    - name: suites
      value: smoke
    - name: namespace
      value: bluefin-test
```

> ⚠️ This only confirms the nvidia service is absent on non-nvidia images (correct). To verify the actual change, run on a bluefin-dx or nvidia-enabled image variant after merge.

### Log collection pattern

Poll and collect logs immediately — log pods are recycled after workflow completion:

```bash
# Poll until Succeeded/Failed
argo_get_workflow name=<workflow-name> namespace=argo

# Collect WHILE Running or immediately after Succeeded
argo_logs_workflow name=<workflow-name> namespace=argo

# Key commands to run inside the VM (via workflow steps or virsh guest-exec):
systemctl --failed --no-pager
journalctl -p warning -b --no-pager -n 200
systemctl is-enabled <unit-name>.service
systemctl cat <unit-name>.service
```

> ⚠️ Do NOT use `argo_wait_workflow` — it issues a blocking MCP call that times out before most workflows complete. Use `argo_get_workflow` to poll.

### Stale image gotcha

If the containerdisk was built before a recent PR merged, new files from that PR won't be present even though they're in the source. Always cross-check:

```bash
# Check when the current testing image was built
skopeo inspect docker://ghcr.io/projectbluefin/bluefin:testing | jq '.Created'

# Cross-check: when did the PR that added the file merge?
gh pr view <N> --repo projectbluefin/common --json mergedAt
```

If the containerdisk predates the PR, the lab baseline is stale. Wait for a rebuild (nightly at 02:00 UTC) or note it clearly in the report.

---

## CI gate interpretation

| Check name | What it means | If it fails |
|---|---|---|
| `Build and push image` | OCI image composes successfully from Containerfile | Blocking — fix build error |
| `Compose PR test image` | common layer composed for E2E gate | GHCR 504/timeout = transient, re-trigger the workflow |
| `E2E — composed common suite` | GNOME common AT-SPI suite on composed image | Investigate test output; may be flaky — re-run once |
| `test` | pytest + bats unit tests | Blocking — fix test failures |
| `pre-commit` | SHA pinning, JSON/YAML/TOML hygiene, actionlint | Blocking — run `pre-commit run --all-files` locally |
| `ghost-lab` | PR-specific lab test on KubeVirt cluster | Stale pending = needs requeue (see below) |
| Renovate `dependencies` | Automated dependency update PR | Auto-merge on CI pass; no code review needed |

### Transient failures

- **GHCR 504 on `Compose PR test image`**: This is a transient GitHub Container Registry timeout, not a code problem. Re-trigger the workflow with `gh workflow run` or the GitHub UI.
- **`ghost-lab` stale pending**: The lab check can get stuck in a "pending" loop. Requeue via `gh api` (see below).

---

## ghost-lab check — requeue a stale pending check

The `ghost-lab` status check is set via the GitHub Checks API from the lab cluster. If it gets stuck in a pending state:

```bash
# Check current status
gh api repos/projectbluefin/common/commits/<SHA>/check-runs \
  --jq '.check_runs[] | select(.name == "ghost-lab") | {status, conclusion, html_url}'

# Requeue by re-triggering the lab dispatch workflow
gh workflow run ghost-lab-dispatch.yml \
  --repo projectbluefin/common \
  --ref <branch-name>

# If the check run itself needs to be reset to queued:
gh api repos/projectbluefin/common/check-runs/<check-run-id> \
  -X PATCH \
  --field status=queued
```

If the lab is unavailable, a maintainer can manually set the check to `success` with a note:

```bash
gh api repos/projectbluefin/common/statuses/<SHA> \
  -X POST \
  --field state=success \
  --field context=ghost-lab \
  --field description="manually cleared — lab unavailable"
```

---

## OEM hook review pattern

OEM hardware first-boot hooks live in `system_files/shared/usr/share/ublue-os/user-setup.hooks.d/`. Full context is in [`oem-hardware-hooks.md`](oem-hardware-hooks.md).

### Version-script guard

```bash
# Pattern: work inside the guard re-runs when version bumps
run_if_new_version() {
    local version="$1"
    # ... checks ~/.local/share/ublue-os/user-setup-complete against $version
}

run_if_new_version "20260101" << 'EOF'
  # Non-idempotent or expensive work — runs only when version > stored
  dconf write /org/gnome/... "'value'"
  systemctl --user enable some.service
EOF

# Idempotent-safe work — lives OUTSIDE the guard, runs every boot
if is_this_hardware; then
    brew install --cask some-app 2>/dev/null || true
fi
```

- **Inside guard**: dconf writes, systemctl enables, one-time migration tasks
- **Outside guard**: brew installs (idempotent), configuration checks, WirePlumber fragment installs (idempotent copy)

**Version bump implication**: when version changes, ALL existing users re-run the guarded block. Code inside must be safe to re-run — dconf writes and systemctl enables are idempotent by nature, but destructive operations (rm -rf, overwriting config files) need extra care.

### DMI gates

Scope to exact hardware using all three DMI fields:

```bash
if [[ "$(cat /sys/class/dmi/id/chassis_vendor)" == "Framework" ]] && \
   [[ "$(cat /sys/class/dmi/id/sys_vendor)" == "Framework Computer LLC" ]] && \
   [[ "$(cat /sys/class/dmi/id/product_name)" == "Framework Desktop" ]]; then
    # hardware-specific setup
fi
```

- Using only `product_name` can match unrelated hardware with the same name across vendors
- `chassis_vendor` + `sys_vendor` + `product_name` together uniquely identifies the target
- **QEMU lab caveat**: `/sys/class/dmi/id/product_name` in QEMU VMs is typically `"Standard PC (i440FX + PIIX, 1996)"` or similar — DMI gate will correctly NOT fire. Verify this in lab to confirm the install block is a no-op on non-target hardware.

### WirePlumber fragments

WirePlumber user-space audio profiles must be written to the **user** fragment directory:

```
~/.config/wireplumber/wireplumber.conf.d/    ← correct (user-setup hook)
/usr/share/wireplumber/wireplumber.conf.d/   ← wrong for user hooks (system-level, would need root)
```

The copy should be idempotent — check if the target exists and matches before writing, or use `install -m 644` which overwrites safely.

### Brew availability early-exit trap

User-setup hooks typically check for brew early and exit if it isn't available yet. This creates a **silent failure mode** for brew-independent work placed after the check:

```bash
# Common hook structure — DANGER ZONE
BREW_BIN="$(brew --prefix)/bin/brew" 2>/dev/null
if [[ ! -x "${BREW_BIN}" ]]; then
    exit 0   # ← early exit: brew not ready yet
fi

# ... versioned brew installs ...

# WirePlumber config copy — has NO brew dependency
# BUG: this never runs on first login when brew isn't present yet
install -m 644 "${SCRIPT_DIR}/51-hardware.conf" \
    "${HOME}/.config/wireplumber/wireplumber.conf.d/"
```

**Fix**: move brew-independent payloads (config file copies, DMI-gated fragments) **before** the `BREW_BIN` early-exit:

```bash
# Correct order: brew-independent work first
if is_target_hardware; then
    install -m 644 "${SCRIPT_DIR}/51-hardware.conf" \
        "${HOME}/.config/wireplumber/wireplumber.conf.d/"
fi

# Then gate on brew
BREW_BIN="$(brew --prefix)/bin/brew" 2>/dev/null
if [[ ! -x "${BREW_BIN}" ]]; then
    exit 0
fi

# brew-dependent versioned work below
```

**Review signal**: if a hook installs a config file after a `BREW_BIN` check and the config has no brew dependency, flag it — the config silently won't install on first login (PR #760: `51-framework-desktop.conf` hit this).

---

## Test quality checklist

Derived from review of PR #785 (unit tests for check-oci-refs, bazaar-hook, hardware hooks, nvidia-flatpak-sync).

### Python unit tests (pytest / mock)

**Assert full argv, not substring membership:**

```python
# BAD — misses --cask flag, wrong positional order, wrong spawn wrapper
assert "brew" in " ".join(mock_popen.call_args_list[0][0][0])

# GOOD — catches all of these
mock_popen.assert_called_once_with(
    ["brew", "install", "--cask", "some-app"],
    start_new_session=True,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
)
```

**Verify key kwargs:**
- `start_new_session=True` — required for background spawns that must survive the parent
- `env=` override — if the script sets environment variables, assert they are passed
- `cwd=` — if the script changes directory before the call

**Cover failure paths:**
- Non-zero return code from subprocess
- Missing file / path not found
- Permission error

**Mock granularity:**
- Mock at the level of the function under test, not the underlying OS call — `patch("module.subprocess.Popen")` not `patch("subprocess.Popen")` to avoid cross-module leakage

### Bats tests (shell)

**Echo the full received command line from mocks:**

```bash
# BAD — collapses all grubby calls to the same label, loses argument content
function grubby() { echo "mock: grubby update"; }

# GOOD — emits full argv, allows assertion on specific arguments
function grubby() { echo "mock: grubby $*"; }
```

Then assert against specific content:

```bash
run some-script-under-test
assert_output --partial "grubby --update-kernel=ALL --args=blacklist=nouveau"
```

**Framework mock pitfall**: if the bats test framework provides a mock helper that collapses argv to a label, do not use it for commands where the argument list matters. Write a minimal function mock instead.

**Assert `--now` and `--args` content:**

```bash
# BAD — discards content
assert_output --partial "systemctl enable"

# GOOD — verifies the full invocation including --now and service name
assert_output --partial "systemctl enable --now some.service"
```

---

## PR review report template

Use this format when reporting findings from a review session. One table row per PR, followed by per-PR detail blocks for any findings.

### Summary table

| PR | Title | Type | Code review | Lab result | CI | Verdict |
|---|---|---|---|---|---|---|
| #NNN | short title | `systemd unit` / `OEM hook` / etc | ✅ Clean / ⚠️ N findings | ✅ Green / ⚠️ Stale image / ❌ Failed | ✅ / ❌ transient | ✅ Ready / ⚠️ Needs fix / 🔁 Re-trigger CI |

### Per-PR detail block

```
### PR #NNN — <title>

**Type:** <PR type from taxonomy>
**Branch:** <branch-name>
**CI:** <status — all green / compose failed (transient GHCR 504) / ghost-lab pending>

**Code review findings:**
- [SEVERITY] Description — file:line — suggested fix

**Lab baseline** (workflow: <name>, phase: <Succeeded/Failed>):
| Artifact | Baseline state |
|---|---|
| `unit-name.service` | ABSENT / EXISTS (content: ...) |
| `systemctl is-enabled` | enabled / disabled |
| `systemctl --failed` | Clean / mcelog only (QEMU noise) |

**Post-merge verification:**
```bash
# paste the exact commands to verify after rebuild
```

**Verdict:** ✅ Ready to merge / ⚠️ Fix needed before merge / 🔁 Re-trigger CI / 🔬 Needs nvidia lab
```

### Example report — 2026-06-24 session

| PR | Title | Type | Code review | Lab result | CI | Verdict |
|---|---|---|---|---|---|---|
| #785 | unit tests: check-oci-refs, bazaar-hook, hw hooks | `test addition` | ⚠️ 2 findings (weak assertions) | N/A | ✅ | ⚠️ Strengthen assertions |
| #767 | flatpak appstream refresh every boot | `systemd unit (shared)` | ⚠️ 1 finding (`graphical.target` contextually OK; metered guard missing) | ⚠️ Service disabled in image — preset not yet built in | ❌ CI stale (GHCR 504) | ⚠️ Add metered guard, re-trigger CI |
| #768 | uupd AC-aware update scheduling | `systemd unit (shared)` | ⚠️ 1 finding (rate-limit consumes burst on brief AC plug) | ✅ Baseline captured — 3 new artifacts absent as expected | ✅ | ⚠️ Rate-limit logic needs review |
| #769 | nvidia flatpak runtime sync | `systemd unit (nvidia)` + `skill doc` | ⚠️ 1 finding (failed flatpak update not retried on restart) | ✅ Green — service absent on non-nvidia (correct) | ⚠️ ghost-lab stale pending | ⚠️ Fix retry logic; requeue ghost-lab |
| #760 | Framework Desktop WirePlumber OEM hook | `OEM hardware hook` + `skill doc` | ⚠️ 1 finding (wireplumber install after BREW_BIN check — won't run on first login) | ⚠️ Stale image — 20-oem-brew.sh not in testing build | ✅ | ⚠️ Move wireplumber install before brew check |
| #790 | chore(deps): update actions/cache | `dependencies` | N/A (Renovate) | N/A | ✅ | ✅ Auto-merge on CI pass |
| #789 | chore(deps): update taiki-e/install-action digest | `dependencies` | N/A (Renovate) | N/A | ✅ | ✅ Auto-merge on CI pass |

---

## Review sign-off checklist

Before approving any `system_files/` PR:

- [ ] Universal checklist passed
- [ ] Per-type checklist completed for all changed paths
- [ ] CI is green (or transient failures identified and re-triggered)
- [ ] `ghost-lab` check is not stale-pending (requeue if needed)
- [ ] Lab verification done or E2E CI pass accepted as equivalent
- [ ] Skill file update committed in the same PR if a new pattern was discovered
- [ ] PR title is Conventional Commits
- [ ] Attribution trailers present on AI-authored commits (convention, not a gate)
