# Expo Release Story 2

Continuous deployment on `main` using EAS Workflows, fingerprint-aware builds, OTA updates, and Maestro smoke tests.

## How it works

Every push to `main` triggers two workflows. One runs, the other skips — controlled by the `isReleased` flag in `app.json`.

### Note about isReleased flag

The `isReleased` flag is a contrivance in order to make it easy to demonstrate the workflow. Realisticially, we would query some other source to determine if a build was actually "released"- for instance status in App Store Connect. However, even this probably isn't sufficient. You may want to consider code "released" once it's waiting for review, for instance. 

### Version scheme

- **`x.y.0`** — native version / app store build (also the runtime version)
- **`x.y.1+`** — OTA update patch versions

The runtime version is always `major.minor.0`, derived from the app version. Patch numbers are reserved for OTA updates.

### Pre-release (`isReleased: false`)

**Workflow:** `pre-release-deploy.yaml`

This is the active path while preparing a new native release. On every push to `main`:

1. **Fingerprint** the project
2. **Check for an existing iOS build** matching the fingerprint + runtime version
3. If no match: **build** a new iOS binary and **submit** to TestFlight
4. If match found: **repack** the existing build with the latest JS bundle and submit

This means native code changes produce new builds, while JS-only changes reuse the existing binary — saving build time and costs.

When `isReleased: true`, this workflow shows a "Skipped" message.

### Released (`isReleased: true`)

**Workflow:** `released-deploy.yaml`

This is the active path once the app is live in the store. On every push to `main`:

1. **Fingerprint** the project
2. **Find an iOS production build** matching the fingerprint + runtime version
3. **Verify the build exists** — if no match, the workflow **fails** (native changes are blocked in released state)
4. **Get or build an iOS simulator build** for testing
5. **Run a Maestro smoke test** on the simulator build
6. **Resolve the next patch version** (e.g., `1.0.3` → `1.0.4`)
7. **Publish the OTA update** to the production channel

When `isReleased: false`, this workflow shows a "Skipped" message.

### Switching between states

- Start a new version: set `isReleased: false` and bump the version in `app.json`
- After the store build is approved: set `isReleased: true`
- All subsequent pushes to `main` deploy as OTA updates until the next version bump

### PR previews

**Workflow:** `pr-preview.yaml`

On every pull request, an OTA update is published to a branch-specific channel (`pr-<number>`) so changes can be tested in a development build before merging.

### Channel surfing

The Settings screen includes a channel surfing picker that lets you switch between update channels (`production`, `pr-1`, `pr-2`, `pr-3`) at runtime using `expo-updates` header overrides. This is useful for testing PR updates on a production build.

## Key files

| File | Purpose |
|------|---------|
| `app.json` | Base config — `extra.isReleased` controls workflow behavior |
| `app.config.js` | Dynamic config — derives runtime version, supports multi-brand |
| `eas.json` | Build profiles (`production`, `production-simulator`, `development`) |
| `.eas/workflows/pre-release-deploy.yaml` | Build + submit workflow |
| `.eas/workflows/released-deploy.yaml` | OTA update workflow with Maestro gate |
| `.eas/workflows/pr-preview.yaml` | PR preview updates |
| `maestro/smoke-test.yaml` | Minimal smoke test (launch app) |
| `fingerprint.config.js` | Excludes volatile config sections from fingerprint |
