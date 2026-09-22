# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this project is

This is a **Trusted Web Activity (TWA) wrapper** that packages the [Minesweeper PWA](https://minesweeper-60xh.onrender.com) as a native Android app for the Google Play Store, built with [Bubblewrap](https://github.com/GoogleChromeLabs/bubblewrap). It does not contain any game logic or React source — it's purely an Android shell that opens the live PWA URL in a full-screen Chrome Custom Tab / TWA.

Package ID: `developer.rohitsaw.minesweeper`

This repo is intentionally kept **separate from the PWA's own repo**, because their release cycles differ:
- Changes to the web app (game logic, UI, React code) ship instantly with no Android release, since the TWA just loads the live URL.
- A release from *this* repo is only needed for wrapper-level changes: app icon, splash screen, target/min SDK version, signing config, app name, theme colors.

## Repo layout — this repo holds config, not a generated project

[twa-manifest.json](twa-manifest.json) is the **only** source of truth for the wrapper. Everything Bubblewrap generates from it — `app/` (Gradle module, Java glue classes, resources, icons), root `build.gradle`/`settings.gradle`/`gradlew`/`gradle/`, `store_icon.png`, `manifest-checksum.txt` — is **gitignored** (see [.gitignore](.gitignore)) and does not live in version control. CI regenerates the whole Android project fresh from `twa-manifest.json` on every release run (see below), and it may still exist on disk locally from a previous local build, but it is never the thing to edit or to trust as current.

Committed files are just: `twa-manifest.json`, [.github/workflows/release.yml](.github/workflows/release.yml), `README.md`, `CLAUDE.md`, `.gitignore`.

**Consequence for any config fix**: the fix always belongs in `twa-manifest.json` (icon URLs, colors, name, host, `appVersion`, etc.), never in `app/build.gradle` or `app/src/main/res/**` directly — those files may not even exist in a fresh checkout, and even if present locally, edits to them are never committed and get silently blown away by the next `bubblewrap update`/CI run.

## How releases work

The **Build & Publish Android Release** GitHub Actions workflow runs automatically on every push to `main` (`on: push: branches: [main]` in [release.yml](.github/workflows/release.yml)) — **there is no manual trigger and no review gate**. Any push to `main`, including a docs-only or `twa-manifest.json` change, builds and publishes a new production release. It:
1. Runs `bubblewrap update --skipVersionUpgrade` to regenerate the entire Android project from `twa-manifest.json`, at a **pinned** Bubblewrap CLI version (`BUBBLEWRAP_VERSION` env var in [release.yml](.github/workflows/release.yml)).
2. Patches `versionCode` into the freshly-generated `app/build.gradle`.
3. Builds a signed `.aab` using Bubblewrap CLI and the upload keystore (from secrets).
4. Publishes it directly to the Play Store `production` track via the Play Developer API.

**Why `BUBBLEWRAP_VERSION` is pinned**: each Bubblewrap release bundles its own `compileSdkVersion`/`targetSdkVersion`. Since the whole project is regenerated on every run, an unpinned `npx @bubblewrap/cli` would mean the Android target SDK could silently drift on any routine release. The pin makes an SDK/Play-policy bump a deliberate, reviewed one-line change to `release.yml` instead.

**Version code**: `4 + <workflow run number>` — computed and patched into the generated `app/build.gradle` at build time (`VERSION_CODE_BASE` env var in the workflow). There's no committed `versionCode` to hand-edit; it doesn't exist until CI generates it. 4 was the last versionCode published via the old pwabuilder.com flow, so this scheme guarantees monotonic increase from there.

**Version name**: static, set by hand in `twa-manifest.json`'s `appVersion` — bump it for a meaningful, user-visible version change.

Required GitHub Actions secrets: `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_PASSWORD`, `ANDROID_KEY_ALIAS`, `PLAYSTORE_ACCOUNT_KEY`. See [README.md](README.md) for details on each.

The upload keystore (`~/minesweeper.jks` locally, base64-encoded in CI secrets) is the only thing that lets future updates land on the existing Play Store listing — it is not stored in this repo and must stay backed up outside it.

## Working conventions for this repo

- Config changes (icon, colors, name, host, version name, etc.) are a one-file edit: change `twa-manifest.json` and push to `main` — that push ships to production automatically, there's no separate release step.
- **Any push to `main` publishes to the Play Store production track immediately** — this was a deliberate choice (auto-publish over a manual/reviewed gate), so treat every commit to `main` in this repo as a live release, not a draft. Land wrapper changes on a branch/PR first if they need review before shipping.
- To bump the Android target/compile SDK version (e.g. for a Play policy deadline), bump `BUBBLEWRAP_VERSION` in `release.yml` by hand, in its own reviewed change — don't add a manifest field for this, it isn't one. Merging that change ships a release using the new SDK immediately.
- Never suggest committing `app/`, `build.gradle`, `settings.gradle`, `gradlew`, `gradle/`, `store_icon.png`, or `manifest-checksum.txt` — they're gitignored on purpose.
- Local builds need JDK 17, the Android SDK, and Node; the project must be generated with `bubblewrap update` before it can be opened/built (see [README](README.md)) since it isn't checked in.
