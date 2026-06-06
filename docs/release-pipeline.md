# Projectbluefin Promotion Pipeline Specification

All projectbluefin image repos (`bluefin`, `bluefin-lts`, `dakota`) **MUST** implement a
consistent, artifact-preserving promotion pipeline. This document is the canonical reference.

Derived from epic [#516](https://github.com/projectbluefin/common/issues/516).

## Core Principle

**Build once. Promote the artifact. Never rebuild for a higher environment.**

The image that passes e2e in the Gate stage must be the **exact binary artifact** (by digest)
that reaches production. Any divergence between the tested artifact and the shipped artifact
is a CD correctness violation.

## Pipeline Stages

### Stage 1 — Build

- **Trigger:** push to `main` (or `testing` branch for bluefin)
- **Action:** build the OCI image, push **`:$sha` only** — the stream tag (`:testing`) is
  **never** set at build time
- SBOM generation and cosign signing MUST cover the digest reference
- **Output:** immutable digest artifact uploaded as a GitHub Actions artifact for downstream jobs

### Stage 2 — Gate (post-build e2e)

- **Trigger:** `workflow_run` after every successful Build completes on `main`
- **Input:** `:$sha@digest` from the build artifact (immutable — cannot be overwritten)
- **Action:**
  - Run `smoke,common` e2e suites against the digest reference
  - On **pass**: `skopeo copy --all @digest → :testing` — promotes the verified image
  - On **fail**: open a P1 issue; `:testing` is **not updated** — the previous good image stays
- `:testing` always points to a digest that has passed gate e2e

### Stage 3 — Weekly Promote

- **Trigger:** cron **Tuesday 06:00 UTC** across all repos; `workflow_dispatch` bypasses the floor
- **Pipeline:**
  1. **7-day floor check** — blocked if last release was < 7 days ago (manual dispatch bypasses)
  2. **Resolve `:testing` digest** — lock exact digest + source SHA of the image to promote
  3. **Full e2e** (`developer,vanilla-gnome,software,common` or repo-equivalent) against `@digest`
  4. **Cosign verify** all flavor digests — abort if any flavor fails signature verification
  5. **TOCTOU SHA guard** — re-verify branch HEAD has not advanced since step 2; abort if changed
  6. **`skopeo copy --all @digest → :latest + :stable`** (NO REBUILD — same artifact by digest)
  7. **Fast-forward** `latest`/`stable` branches to the promoted source SHA
  8. **Generate release** notes

## Anti-patterns

These are **forbidden** in all projectbluefin promotion workflows:

| Anti-pattern | Why it is wrong |
|---|---|
| Tag `:testing` at build time before e2e | Users can pull a broken image before tests have run |
| Rebuild for a higher environment | Violates "build once" — the rebuilt image was never tested |
| Promote by tag (not digest) | Tag can be overwritten between lock and promote (TOCTOU race) |
| Skip cosign verify before promotion | Unsigned or tampered image can reach production silently |
| Promote without TOCTOU guard | Untested commits land in production if `main` advanced during e2e |
| Float a stream tag before e2e | Identical to tagging `:testing` before gate — same failure mode |

## Required e2e Suites by Stage

| Stage | Minimum suites | Notes |
|---|---|---|
| Gate (post-build) | `smoke,common` | Fast — runs on every push to main |
| Weekly Promote | `developer,vanilla-gnome,software,common` | Full — runs before every production promotion |
| Nightly monitor | `smoke,common,vanilla-gnome` | Against `:latest` — regression detection only |

Dakota uses `smoke,common` at both gate and promote stages (aligned to its available test scenarios).

## Compliance Matrix

Each repo MUST satisfy all items. ✅ = implemented, ⬜ = open gap (see [epic #516](https://github.com/projectbluefin/common/issues/516)).

| Requirement | bluefin | bluefin-lts | dakota |
|---|:---:|:---:|:---:|
| Build publishes `:$sha` only — no `:testing` at build time | ⬜ | ⬜ | ✅ |
| Gate e2e runs on `@digest` before `:testing` is tagged | ⬜ | ⬜ | ✅ |
| Weekly promotion cron is Tuesday 06:00 UTC | ✅ | ✅ | ⬜ |
| 7-day floor on weekly promotion | ✅ | ⬜ | ⬜ |
| Full e2e suite runs at weekly promotion time | ✅ | ⬜ | ⬜ |
| Cosign verify all flavors before final promotion | ✅ | ⬜ | ⬜ |
| TOCTOU SHA guard before final `skopeo copy` | ✅ | ⬜ | ⬜ |
| Promote by digest — no tag re-pull, no rebuild | ✅ | ⬜ | ✅ |
| Fast-forward release branches after promotion | ✅ | ✅ | ✅ |
| Promotion failure opens a P1 issue | ✅ | ⬜ | ⬜ |

## Gap Remediation

Sub-issues from [epic #516](https://github.com/projectbluefin/common/issues/516) track each open gap:

| Issue | Repo | Gap |
|---|---|---|
| [#518](https://github.com/projectbluefin/common/issues/518) | bluefin | Gate `:testing` tag behind post-build e2e |
| [#519](https://github.com/projectbluefin/common/issues/519) | bluefin-lts | 7-day promotion floor (blocked on bluefin-lts PR #73) |
| [#520](https://github.com/projectbluefin/common/issues/520) | dakota | Align weekly promotion cron to Tuesday |
| [#521](https://github.com/projectbluefin/common/issues/521) | dakota | Cosign verify before final promotion |
| [#522](https://github.com/projectbluefin/common/issues/522) | dakota | Full e2e suite at weekly promotion time |
| [#524](https://github.com/projectbluefin/common/issues/524) | all repos | TOCTOU SHA guard (bluefin ✅; dakota + LTS pending) |
| [#517](https://github.com/projectbluefin/common/issues/517) | bluefin-lts | Stop rebuilding for production — promote by digest (blocked on PR #73) |

## Reference Implementations

| Workflow | What it demonstrates |
|---|---|
| `projectbluefin/bluefin` `weekly-testing-promotion.yml` | 7-day floor, TOCTOU guard, cosign verify, full e2e, multi-flavor digest handling |
| `projectbluefin/dakota` `publish.yml` | Build→Gate→Promote pattern: `:$sha` only at build, e2e gate before `:testing` |
| `projectbluefin/bluefin` `post-testing-e2e.yml` | `workflow_run` trigger, digest-based gate, promote-to-`:testing` on success |
