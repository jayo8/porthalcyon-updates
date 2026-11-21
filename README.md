# Port Halcyon update channel

Release builds and content packs for Port Halcyon (`com.jadon.porthalcyon`). No source code lives here.
The game reads **https://github.com/jayo8/porthalcyon-updates/releases/latest/download/manifest.json**
on launch (quietly, at most every 20 min, fails silently offline) and from *Settings › Updates › Check for updates* or *Pause › Check for Updates*.

## How Hacky publishes updates

Script: `/workspace/game/tools/publish_update.sh` (needs `gh` logged in as jayo8). Every publish creates a new release marked *latest*
containing `manifest.json`, then re-downloads it and checks every SHA-256.

### Full APK update
1. Bump `versionCode` (must go up) and `versionName` in `android/app/build.gradle`.
2. `bash build.sh` (signs with the Port Halcyon key; cert SHA-256 `2246d54b…5265`).
3. `tools/publish_update.sh apk out/PortHalcyon-v3.apk "what's new"` → release `v<versionName>` with the APK + manifest.

The app downloads it with a progress bar, checks SHA-256, package name, signing key and versionCode, then hands it to
Android's installer (PackageInstaller session). Jadon taps **Update** once; saves are kept.
First time only: Android asks to allow *Install unknown apps* for Port Halcyon.

### Content pack (no APK, can be far bigger than 100 MB)
1. Put only the changed files in a folder laid out like `android/app/src/main/assets/www` (e.g. `game.js`, `assets/models/x.glb`).
   A pack overrides bundled files with the same path, and packs are **not** stacked: each pack must include everything changed since the APK.
2. `tools/publish_update.sh pack <folder> "what's new"` → release `v<versionName>-pack<N>`. The pack version auto-increments from the live manifest,
   `minAppVersionCode`/`maxAppVersionCode` default to the live APK's versionCode (override with `PACK_MIN_VC`/`PACK_MAX_VC`),
   and the zip is split into ≤1900 MB parts (`PART_MB`) to stay under GitHub's 2 GB per-file limit.

On the phone: per-part and whole-zip SHA-256 are verified, the zip is extracted with path checks into the app's private storage and switched in
with one atomic state-file rename; the previous pack is kept. "Restart now" or the next launch uses it. If the game fails to finish loading twice
with a new pack, it automatically rolls back to the previous pack (or the built-in content) and never offers that pack again.
Settings › Updates › *Use built-in content* undoes a pack by hand. Installing a newer APK ignores packs made for older versionCodes.

### Testing without touching players
`TEST_TAG=test-something tools/publish_update.sh …` publishes a **pre-release** (not seen by `latest`). Point a debug launch at it with
`adb shell am start -n com.jadon.porthalcyon/.GameActivity --es updManifest https://github.com/jayo8/porthalcyon-updates/releases/download/test-something/manifest.json --es updAction check|apk|pack`
(only URLs in this repo are accepted). Delete test releases afterwards: `gh release delete test-something --repo jayo8/porthalcyon-updates --yes --cleanup-tag`.
`DRY_RUN=1` builds `out/publish/` without publishing.

## manifest.json
```json
{ "schema": 1, "versionCode": 7, "versionName": "3.3", "url": "https://…/PortHalcyon-v3.3.apk", "sha256": "…", "size": 101422320, "notes": "…",
  "pack": { "version": 1, "minAppVersionCode": 7, "maxAppVersionCode": 7, "sha256": "<whole zip>", "size": 123, "notes": "…",
            "parts": [ { "url": "https://…/pack-1.zip.part00", "sha256": "…", "size": 123 } ] } }
```
`pack` is optional. Same top-level fields as Ember Guard's `version.json`.
