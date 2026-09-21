# Minesweeper Android Deployment

Trusted Web Activity (TWA) wrapper that packages [Minesweeper](https://minesweeper-60xh.onrender.com) for the Play Store, generated with [Bubblewrap](https://github.com/GoogleChromeLabs/bubblewrap). Package: `developer.rohitsaw.minesweeper`.

Kept separate from the app repo because release cycles differ: a TWA just loads the live PWA URL, so web/feature changes ship instantly with no Android release needed. An Android release here is only needed for wrapper-level changes (icon, target SDK, signing, app name).

## Releasing

Run the **Build & Publish Android Release** workflow manually (Actions tab → Run workflow). It builds a signed `.aab` and publishes it straight to the Play Store `production` track.

Required repo secrets:

| Secret | Value |
|---|---|
| `ANDROID_KEYSTORE_BASE64` | `base64 -i ~/minesweeper.jks \| pbcopy` |
| `ANDROID_KEYSTORE_PASSWORD` | keystore password |
| `ANDROID_KEY_PASSWORD` | key password |
| `ANDROID_KEY_ALIAS` | key alias (`my-key-alias` for the existing PWABuilder-issued key) |
| `PLAYSTORE_ACCOUNT_KEY` | Play Developer API service account JSON, as plain text |

Version code is `4 + <workflow run number>` (4 was the last version published via PWABuilder), so it always increases. Version name is static in [twa-manifest.json](twa-manifest.json)'s `appVersion` field — bump it by hand for a meaningful release.

## Updating for a new Android target SDK / Play policy requirement

This is the whole reason this exists instead of going back to pwabuilder.com each year:

```bash
npm install -g @bubblewrap/cli   # or use npx
bubblewrap update --skipVersionUpgrade
```

This regenerates the project from [twa-manifest.json](twa-manifest.json) using whatever `compileSdkVersion`/`targetSdkVersion` the installed Bubblewrap version bundles. Review the diff, commit, and run the release workflow.

## Local build

Requires JDK 17 and the Android SDK. See [twa-manifest.json](twa-manifest.json) for the signing key path/alias it expects, or override on the command line:

```bash
BUBBLEWRAP_KEYSTORE_PASSWORD=... BUBBLEWRAP_KEY_PASSWORD=... \
  npx @bubblewrap/cli build --signingKeyPath=~/minesweeper.jks --signingKeyAlias=my-key-alias
```

The upload key lives at `~/minesweeper.jks` (SHA256 `05:6C:6B:B5:A8:B0:4B:43:4A:2C:E1:36:8C:42:2E:4B:D9:10:FC:20:B1:7E:E0:09:3E:FC:D0:61:D8:A0:73:86`, matching the fingerprint published in the app repo's `assetlinks.json`). Keep it backed up somewhere safe outside this repo — losing it means losing the ability to update the Play Store listing.
