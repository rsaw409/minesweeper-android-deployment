# Minesweeper Android Deployment

Trusted Web Activity (TWA) wrapper that packages [Minesweeper](https://minesweeper-60xh.onrender.com) for the Play Store, generated with [Bubblewrap](https://github.com/GoogleChromeLabs/bubblewrap). Package: `developer.rohitsaw.minesweeper`.

Kept separate from the app repo because release cycles differ: a TWA just loads the live PWA URL, so web/feature changes ship instantly with no Android release needed. An Android release here is only needed for wrapper-level changes (icon, target SDK, signing, app name).

**This repo only holds config, not a generated Android project.** [twa-manifest.json](twa-manifest.json) is the single source of truth; everything Bubblewrap normally generates from it (`app/`, root `build.gradle`/`settings.gradle`/`gradlew`, `store_icon.png`, etc.) is gitignored and gets (re)created fresh by CI from that file on every release run — see [.gitignore](.gitignore). That means publishing most wrapper changes is just: edit the JSON, commit, run the release workflow.

## Publishing an update to the already-released app

**If you only changed the web app (game logic, UI, React code, content):** do nothing here. The TWA loads the live PWA URL at runtime, so it's already live — no Android release needed.

**If you need to change something about the Android wrapper itself**, edit the relevant field(s) in [twa-manifest.json](twa-manifest.json), commit, and run the release workflow — CI regenerates the whole Android project from the file before building:

| What changed | Field(s) to update in `twa-manifest.json` | Extra step |
|---|---|---|
| App icon / maskable icon | `iconUrl`, `maskableIconUrl` | None |
| Splash screen background | `backgroundColor` | None |
| App/launcher name | `name`, `launcherName` | None |
| Theme / status bar / nav bar colors | `themeColor`, `themeColorDark`, `navigationColor`, `navigationColorDark` | None |
| PWA host URL / start path (rare) | `host`, `startUrl`, `webManifestUrl`, `fullScopeUrl` | Also update `assetlinks.json` in the PWA repo if the domain changed |
| User-visible version number (e.g. "3.0.0.1" → "3.0.0.2") | `appVersion` | None |
| Internal Play Store version code | *(don't touch)* | Handled automatically by CI — see below |
| Android target/compile/min SDK version, Play policy compliance bump | *(not a manifest field)* | Bump `BUBBLEWRAP_VERSION` in [release.yml](.github/workflows/release.yml) — see below |
| Anything else in `twa-manifest.json` (shortcuts, notifications, orientation, …) | see [Bubblewrap's manifest docs](https://github.com/GoogleChromeLabs/bubblewrap/blob/main/packages/cli/README.md) | None |

## Bumping the Android target SDK / Play policy compliance version

CI always regenerates the Android project by running `bubblewrap update` at a **pinned** Bubblewrap CLI version (`BUBBLEWRAP_VERSION` in [release.yml](.github/workflows/release.yml)), rather than "whatever's latest." That pin is deliberate: it's the only thing standing between a routine release and an unreviewed target SDK bump, since each Bubblewrap release bundles its own `compileSdkVersion`/`targetSdkVersion`. To pick up a new one (e.g. for a Play target API level deadline), bump that version number by hand, in its own reviewed change, then run the release workflow.

## Releasing

Run the **Build & Publish Android Release** workflow manually (Actions tab → Run workflow). It regenerates the Android project from `twa-manifest.json`, builds a signed `.aab`, and publishes it straight to the Play Store `production` track.

Required repo secrets:

| Secret | Value |
|---|---|
| `ANDROID_KEYSTORE_BASE64` | `base64 -i ~/minesweeper.jks \| pbcopy` |
| `ANDROID_KEYSTORE_PASSWORD` | keystore password |
| `ANDROID_KEY_PASSWORD` | key password |
| `ANDROID_KEY_ALIAS` | key alias (`my-key-alias` for the existing PWABuilder-issued key) |
| `PLAYSTORE_ACCOUNT_KEY` | Play Developer API service account JSON, as plain text |

Version code is computed automatically as `4 + <workflow run number>` (4 was the last version published via PWABuilder), so it always increases — CI patches `versionCode` into the freshly-generated `app/build.gradle` at build time, so there's nothing to hand-edit. Version name is static in [twa-manifest.json](twa-manifest.json)'s `appVersion` field — bump it by hand for a meaningful, user-visible release.

## Local build

The Android/Gradle project isn't checked in — generate it first, then build. Requires JDK 17, the Android SDK, and Node (for `npx`). See [twa-manifest.json](twa-manifest.json) for the signing key path/alias it expects, or override on the command line:

```bash
npx @bubblewrap/cli@1.25.0 update --skipVersionUpgrade   # materializes app/, gradlew, etc. from twa-manifest.json

BUBBLEWRAP_KEYSTORE_PASSWORD=... BUBBLEWRAP_KEY_PASSWORD=... \
  npx @bubblewrap/cli@1.25.0 build --signingKeyPath=~/minesweeper.jks --signingKeyAlias=my-key-alias
```

Pin the same Bubblewrap version CI uses (`BUBBLEWRAP_VERSION` in [release.yml](.github/workflows/release.yml)) so your local build matches what gets published.

The upload key lives at `~/minesweeper.jks` (SHA256 `05:6C:6B:B5:A8:B0:4B:43:4A:2C:E1:36:8C:42:2E:4B:D9:10:FC:20:B1:7E:E0:09:3E:FC:D0:61:D8:A0:73:86`, matching the fingerprint published in the app repo's `assetlinks.json`). Keep it backed up somewhere safe outside this repo — losing it means losing the ability to update the Play Store listing.
