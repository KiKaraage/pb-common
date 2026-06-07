---
name: factory-improvement
description: "Self-improving factory loop for projectbluefin — systematically closes gaps between design and implementation until every automatable step is automated. Use when identifying automation gaps, proposing factory workflows, or auditing ACMM maturity level."
metadata:
  type: procedure
---

# Factory Improvement — Self-Improving Loop

## Contents
- [When to Use](#when-to-use)
- [When NOT to Use](#when-not-to-use)
- [Mission](#mission)
- [Human Gates — Intentional and Non-Negotiable](#human-gates--intentional-and-non-negotiable)
- [The Improvement Loop](#the-improvement-loop)
- [Pipeline Uniformity Checklist](#pipeline-uniformity-checklist)
- [E2E Gate Matrix](#e2e-gate-matrix)
- [Finding Open Gaps](#finding-open-gaps)
- [What "Done" Looks Like](#what-done-looks-like)

---

## When to Use

- Dedicated "improve the factory" sessions
- After an architectural review surfaces gaps
- Onboarding a new repo into the factory standard
- Quarterly health check: design vs. reality drift

## When NOT to Use

- Fixing a specific bug (use the repo's AGENTS.md + relevant skill)
- Reviewing a single PR (use `hive-review` or `queue-dashboard`)
- Active incident response — fix the incident first, then run this loop

---

## Mission

**Full automation of everything that does not require a human judgment call.**

The factory should be self-healing: issues flow through the pipeline, agents handle
the mechanical work, and humans make only the decisions requiring accountability,
context, or trust. Every manual step that *could* be automated is a reliability tax.

---

## Human Gates — Intentional and Non-Negotiable

Never automate these. Never propose automating them without explicit maintainer approval:

| Gate | Why it must be human |
|---|---|
| `/approve` comment — lifecycle moves the issue directly to `status/queued` | Prioritization judgment; agent scope assignment |
| PR merge approval (1 human reviewer per CODEOWNERS) | Accountability; trust for org-critical changes |
| `hive/p0` and `hive/p1` label assignment | Release impact judgment |
| Production promotion decisions (Tuesday 06:00 UTC, N=7 floor) | Final go/no-go for user-facing changes — automation handles the gate, but human review of the e2e results is the last word |
| `/unclaim` on stale PRs | Judgment on abandoned vs. still-active claim |
| Any write or automated action to `ublue-os/*` namespace | Absolute prohibition — includes issues, PRs, reports, webhooks, dispatch. Reads only. |

Everything else is automatable.

---

## The Improvement Loop

```
MEASURE → TRIAGE → IMPLEMENT → CAPTURE → VERIFY → LOOP
```

### MEASURE

```bash
~/src/hive-status

# P0 blockers across the factory
gh search issues --label "hive/p0" --owner projectbluefin --state open \
  --json number,title,repository

# P1 this cycle
gh search issues --label "hive/p1" --owner projectbluefin --state open \
  --json number,title,repository

# All open factory gaps (not yet hive-escalated)
gh search issues --label "ai-context" --owner projectbluefin --state open \
  --json number,title,repository
```

### TRIAGE

For each open gap:
- Human gate? → **SKIP** (log it, do not touch)
- Doc gap? → **IMMEDIATE** (cheapest fix, push directly to main)
- CI/tooling gap? → file as GitHub issue (`kind/improvement`, `area/ci`); humans set priority
- Cross-repo gap? → assess blast radius before acting

> ⚠️ Do **not** self-apply `hive/p0`, `hive/p1`, or `status/queued` labels. Priority and queue admission are human decisions; agents file the issue and stop.

### IMPLEMENT

- Work highest-blast-radius gap first
- Prefer: doc fix > CI change > code change
- Max 4 open PRs at once
- Always `just check` + `pre-commit run --all-files` before commit

### CAPTURE

When you discover a gap:

1. File a GitHub issue in `projectbluefin/common`
2. Required: exactly one `kind/*` label + at least one `area/*` label (lifecycle guard enforces this before any `/approve`)
3. Add `ai-context` if the gap is an AI/LLM context blindspot that affects agent reliability
4. Write a clear description — what is broken, what the fix looks like, whether it's automatable
5. Stop. Do not self-apply `hive/p*` or `status/queued` — priority and queue admission are human decisions

```bash
# Example: file a factory CI gap
gh issue create --repo projectbluefin/common \
  --title "ci: skill-drift not wired in testsuite" \
  --label "kind/improvement,area/ci" \
  --body "..."
```

### VERIFY

Would a fresh agent reading only the skills avoid the gap just closed?
If no → the skill is still incomplete.

---

## Pipeline Uniformity Checklist

Each factory repo (`common`, `bluefin`, `bluefin-lts`, `dakota`, `testsuite`, `actions`)
must have ALL of:

| Requirement | Check command |
|---|---|
| `AGENTS.md` present | `gh api repos/projectbluefin/{repo}/contents/AGENTS.md` |
| `lifecycle-caller.yml` wired | `gh api repos/projectbluefin/{repo}/contents/.github/workflows/lifecycle-caller.yml` |
| `skill-drift.yml` wired | `gh api repos/projectbluefin/{repo}/contents/.github/workflows/skill-drift.yml` |
| Hive labels present | `gh label list --repo projectbluefin/{repo} \| grep hive` |
| pre-commit config present | `gh api repos/projectbluefin/{repo}/contents/.pre-commit-config.yaml` |
| Squash-only merge | `gh repo view projectbluefin/{repo} --json squashMergeAllowed,mergeCommitAllowed` |

Missing any row = a gap to close.

---

## E2E Gate Matrix

| Repo | Pre-merge | Post-merge | Promotion |
|---|---|---|---|
| common | `pr-e2e.yml` (composed + common suite) | `e2e.yml` | `promotion-candidate-e2e.yml` |
| bluefin | PR smoke gate | post-merge common suite | Tuesday 06:00 UTC, N=7 floor, broad e2e suite |
| bluefin-lts | PR validation | post-merge e2e (PR #70) | promotion smoke + failure issue reporting (PR #70) |
| dakota | PR CI | post-merge gate | — |

Gaps in this matrix = testing blind spots. File issues for missing gates.

---

## Documentation Single-Source-of-Truth

Each rule must exist in exactly ONE location. Other files should have a one-line pointer.

| Rule | Canonical location |
|---|---|
| ublue-os prohibition | `common/AGENTS.md` |
| Issue lifecycle table | `docs/skills/label-workflow.md` |
| PR comment policy | `docs/factory/agentic-model.md` |
| Branch targets by repo | `docs/factory/agentic-model.md` |
| Session start ritual | `common/AGENTS.md` (+ pointer in agentic-model.md) |
| Task→skill routing | `docs/SKILL.md` |

---

## Finding Open Gaps

Factory gaps are tracked as GitHub issues. Do not maintain gap lists in this doc — they drift. Always query GitHub for the current state:

```bash
# P0 blockers (fix before next promotion)
gh search issues --label "hive/p0" --owner projectbluefin --state open \
  --json number,title,repository

# P1 this cycle
gh search issues --label "hive/p1" --owner projectbluefin --state open \
  --json number,title,repository

# AI/LLM context gaps (affect agent reliability)
gh search issues --label "ai-context" --owner projectbluefin --state open \
  --json number,title,repository

# Ready-to-claim CI improvements
gh search issues --label "status/queued,area/ci" --owner projectbluefin --state open \
  --json number,title,repository
```

---

## What "Done" Looks Like

- [ ] Every factory repo has identical infrastructure (AGENTS.md, lifecycle-caller.yml, skill-drift, hive labels, pre-commit, squash-only)
- [ ] Every pipeline stage has a gate: pre-merge CI, post-merge e2e, promotion smoke
- [ ] All rules exist in exactly one canonical location with one-line pointers elsewhere
- [ ] Renovate is running across all repos
- [ ] No known AI blindspot (ACMM audit) is unmitigated
- [ ] Only the human gates listed above remain manual

---

## Session Close

After each improvement session:

1. For each gap discovered: file a GitHub issue (see CAPTURE above)
2. For ACMM blindspot items resolved: update `docs/skills/acmm-audit-level2.md`
3. For significant improvements shipped: append to `docs/factory/IMPROVEMENTS.md`

Do **not** maintain gap lists in this skill file. GitHub issues are the live backlog.

This skill is the operating procedure for the improvement loop, not the backlog itself.
