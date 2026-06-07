---
name: image-registry
description: "projectbluefin OCI image registry reference — all production images published at ghcr.io/projectbluefin/. Use when looking up image paths, tags, or registry structure."
metadata:
  type: reference
---

# Image Registry

All Bluefin images are published to `ghcr.io/projectbluefin/`. The org migration from `ublue-os` is complete — `projectbluefin` is fully standalone.

## Registry paths

| Registry path | Status | Notes |
|---|---|---|
| `ghcr.io/projectbluefin/bluefin:stable` | ✅ Production | Main Bluefin stable stream |
| `ghcr.io/projectbluefin/bluefin:latest` | ✅ Production | Main Bluefin latest stream |
| `ghcr.io/projectbluefin/bluefin:testing` | ✅ Testing | PR gate + E2E candidate |
| `ghcr.io/projectbluefin/bluefin-lts:stable` | ✅ Production | LTS stream |
| `ghcr.io/projectbluefin/bluefin-lts:testing` | ✅ Testing | LTS E2E candidate |
| `ghcr.io/projectbluefin/common` | ✅ Active | Shared layer (this repo) |
| `ghcr.io/projectbluefin/dakota` | ✅ Active | Dakota image |

## How rollback-helper derives the registry path

```bash
IMAGE_VENDOR="$(jq -r '."image-vendor"' < /usr/share/ublue-os/image-info.json)"
IMAGE_REGISTRY="ghcr.io/${IMAGE_VENDOR}"
```

`image-vendor` is set at build time via `00-image-info.sh`. The helper reads it dynamically — do not hardcode the registry path.

## Build-time ublue-os source (wallpapers only)

The Containerfile pulls wallpaper artwork from `ghcr.io/ublue-os/bluefin-wallpapers-gnome` as a **build-time COPY source**. This is a read-only upstream artwork dependency and does not violate the ublue-os prohibition. The production image tree and all runtime registries are fully under `ghcr.io/projectbluefin/`. See [`containerfile.md`](containerfile.md) for details.
