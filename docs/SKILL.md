# Common Skill Router

Agent entry point for `projectbluefin/common`. Load only the skill(s) that match your task.

## Task → Skill

| I need to... | Load |
|---|---|
| **First time / onboarding** | |
| Set up a new dev environment or clone a factory repo | `docs/skills/onboarding.md` |
| Understand CODEOWNERS, triagers, or branch protection | `docs/skills/governance.md` |
| **Session start** | |
| Run hive priority review at session start | `docs/skills/hive-review.md` |
| Understand cross-repo agent rules, branch targets, or sensitive paths | `docs/factory/agentic-model.md` |
| Need to know when to stop and ask a human | `docs/skills/human-gates.md` |
| **Issues and labels** | |
| Understand the issue lifecycle or label taxonomy | `docs/skills/label-workflow.md` |
| Check the PR queue or merge ruleset | `docs/skills/queue-dashboard.md` |
| Understand the hive system / kubestellar-bot loop | `docs/skills/hive.md` |
| Check on-call / hive state for the whole org | `docs/skills/hive.md` |
| **Bluefin image work** | |
| Add, remove, or modify packages (brew, flatpak, RPM) | `docs/skills/bluefin-packages.md` |
| Change a GNOME setting or dconf key | `docs/skills/dconf-consistency.md` |
| Understand what files are editable here vs submodule | `docs/skills/submodule-boundary.md` |
| Touch any image reference or registry path | `docs/skills/image-registry.md` |
| Understand Bluefin image variants and stream matrix | `docs/skills/bluefin-variants.md` |
| Work on the LTS variant | `docs/skills/bluefin-lts.md` |
| Work on `ublue-rollback-helper` | `docs/skills/rollback-helper.md` |
| **Build, CI, and release** | |
| Understand the build pipeline or PR workflow | `docs/skills/bluefin-build.md` |
| Change `.github/workflows/` | `docs/skills/ci-tooling.md` + `docs/skills/workflow-map.md` |
| Understand what each workflow does | `docs/skills/workflow-map.md` |
| Work on E2E test changes | `docs/skills/e2e-ci.md` |
| Debug post-merge E2E CI, MOTD, or brew-setup masking | `docs/skills/e2e-ci.md` |
| Understand the release process or stream tags | `docs/skills/bluefin-release.md` |
| Understand the common release and promotion pipeline | `docs/skills/release-promotion.md` |
| Understand the promotion pipeline (what gates exist today) | `docs/qa/PROMOTION_GATES.md` |
| Work on ISOs or installation media | `docs/skills/bluefin-iso.md` |
| Work on Renovate dependency updates | `docs/skills/bluefin-renovate.md` |
| **Security and QA** | |
| Understand the security model (COPR, cosign, secureboot) | `docs/skills/bluefin-security.md` |
| Understand QA coverage, test matrix, or running tests | `docs/skills/qa.md` |
| Submit a hardware test report | `docs/hardware-testing.md` |
| **Dakota** | |
| Any dakota task (add/remove/update package, CI, debugging) | `docs/skills/dakota-agent-quickstart.md` |
| **Knuckle installer** | |
| Work on the knuckle installer QA or e2e | `docs/skills/knuckle-qa.md` |
| Release the knuckle installer | `docs/skills/knuckle-release.md` |
| Test knuckle in a QEMU VM | `docs/skills/knuckle-testlab.md` |
| **Factory health and improvement** | |
| Understand factory open gaps and parity matrix | `docs/factory/README.md` |
| Improve the factory (gap audit, automation coverage) | `docs/skills/factory-improvement.md` |
| Work on the ACMM / factory maturity model | `docs/skills/acmm-audit-level2.md` |
| Understand the bonedigger lifecycle bot | `docs/skills/bonedigger.md` |
| Understand the skill-drift CI check | `docs/skills/skill-drift.md` |
| Need to write a skill update / unsure what counts | `docs/skills/skill-improvement.md` |
| Understand why Bluefin was rewritten (The Pattern) | `docs/factory/IMPROVEMENTS.md` |

## Improving skill docs

All files in `docs/skills/` are community-maintained operational knowledge. They live in this repo so any contributor can update them with a direct push to `main` (doc-only exception applies).

**When to update a skill:** any time a session surfaces a workaround, non-obvious pattern, or convention. See [`docs/skills/skill-improvement.md`](skills/skill-improvement.md) for the full mandate and checklist.

**Trail of Bits skill-improver** (optional tooling for structured updates):

```bash
npx skills add https://github.com/trailofbits/skills --skill skill-improver
# Then in your editor: /skill-improver docs/skills/<file>
```

For the full catalog of all skill files with descriptions, see [`docs/skills/INDEX.md`](skills/INDEX.md).

## Scope rules

- **Doc tasks**: modify only `docs/` and `AGENTS.md`. Do not create `.github/` workflow files unless the task is explicitly CI work.
- **CI tasks**: touch only `.github/` and update `docs/skills/` if learnings arise.
- **Changes here propagate to all downstream Bluefin variants.** Keep changes surgical.
