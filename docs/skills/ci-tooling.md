# CI tooling

## Floating-tag guard

**Scope:** org-wide pre-commit hook in all `projectbluefin` repos (`common`, `bluefin`, `bluefin-lts`, `dakota`, `knuckle`, `actions`)

The `no-floating-action-tags` hook blocks third-party GitHub Actions from being committed with floating refs.

**Regex:** `uses:(?!.*projectbluefin/).*@(main|master|latest|v[0-9])`

### What it blocks

Third-party actions must be pinned to a full commit SHA, with a human-readable version comment:

```yaml
uses: actions/checkout@abc123def456 # v4
uses: taiki-e/install-action@abc123def456 # v2
```

These floating refs are rejected:

```yaml
uses: actions/checkout@v4
uses: actions/checkout@main
uses: taiki-e/install-action@latest
```

### Why `projectbluefin/` is exempt

The negative lookahead intentionally skips `projectbluefin/*` actions because the org uses managed floating tags for its own automation:

- `projectbluefin/actions@v1` is an intentional managed floating tag for the org's shared composite action library
- `projectbluefin/bonedigger@main` is an intentional managed floating ref for the internal lifecycle bot

Those refs are advanced deliberately by project maintainers. The guard is specifically for third-party actions that must be SHA-pinned.

### Coverage

The hook scans both:

- `.github/workflows/*.yml`
- composite `action.yml` files

This closes the loophole where a workflow might be pinned correctly but a local composite action could still introduce a floating third-party dependency.

### Renovate vs pre-commit

These two protections do different jobs:

- **Renovate** updates existing SHA pins automatically once they are already tracked
- **The pre-commit hook** prevents new floating tags from being introduced in the first place

Use both. Renovate keeps pinned refs fresh; the hook enforces that refs are pinned at commit time.

## Skill drift detection

**Workflow:** `.github/workflows/skill-drift.yml`

`skill-drift.yml` is a PR gate used across projectbluefin repos. It calls the reusable workflow `projectbluefin/actions/.github/workflows/skill-drift-check.yml@v1` to flag when implementation changes land without corresponding skill-doc updates.

### Repo path mapping

| Repo | code-paths | skill-paths |
|---|---|---|
| common | `.github/workflows/**`, `system_files/**`, `Containerfile`, `Justfile` | `docs/skills/**`, `docs/*.md`, `AGENTS.md` |
| bluefin | `.github/workflows/**`, `build_files/**`, `Justfile`, `recipes/**` | `docs/skills/**`, `docs/*.md`, `AGENTS.md` |
| bluefin-lts | `.github/workflows/**`, `build_files/**`, `Justfile` | `docs/skills/**`, `docs/*.md`, `AGENTS.md` |
| dakota | `.github/workflows/**`, `build_files/**`, `Justfile`, `elements/**` | `docs/skills/**`, `docs/*.md`, `AGENTS.md` |
| knuckle | `.github/workflows/**`, `cmd/**`, `internal/**`, `Justfile`, `scripts/**` | `docs/skills/**`, `docs/*.md`, `AGENTS.md` |

### When it fires

A PR that touches any repo's `code-paths` without also touching one of its `skill-paths` triggers the check.

This is advisory, not a hard merge block, but it should be treated as a prompt to update documentation while the implementation context is still fresh.

## Renovate OCI digest tracking

`Containerfile` now has two OCI image pins tracked by Renovate:

1. `docker.io/library/alpine:latest@sha256:...` via Renovate's built-in `dockerfile` manager
2. `ghcr.io/ublue-os/bluefin-wallpapers-gnome:latest@sha256:...` via a custom regex manager in `.github/renovate.json5`

### Why both managers exist

- `FROM docker.io/library/alpine:latest@sha256:...` is a standard Dockerfile dependency, so the built-in `dockerfile` manager handles it
- `COPY --from=ghcr.io/ublue-os/bluefin-wallpapers-gnome:latest@sha256:...` is not covered by the default Dockerfile parser, so a custom regex manager tracks that digest

### Rule when adding more OCI pins

If you add new OCI image pins to `Containerfile`, also update `.github/renovate.json5` so Renovate can keep them current.

That applies to both:

- `FROM` instructions
- `COPY --from=` image references

If a pinned image is not represented in Renovate config, the digest will silently go stale.
