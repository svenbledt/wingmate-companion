# WingMate Companion

The lean in-flight desktop companion for **[WingMate](https://wingmate.bledt.dev)** — the
community flight-tracking platform for **Microsoft Flight Simulator 2024**. It streams your live
flight to WingMate and shows your in-flight readout, nearby traffic, "what I'm flying over", and
your SimBrief flight plan, right alongside the sim.

**This repository hosts the companion's releases and serves its automatic updates.** The app source
lives in the private WingMate monorepo; this repo is the download + auto-update home.

---

## Download & install

Grab the latest from the **[Releases page](https://github.com/svenbledt/wingmate-companion/releases/latest)**:

| You want… | Download |
|-----------|----------|
| **Normal install** (recommended) | **`WingMateCompanion-win-Setup.exe`** — run once; the app keeps itself up to date after that. |
| **No-install / portable** | **`WingMateCompanion-win-Portable.zip`** — unzip and run `WingMate.Companion.exe`. |

> **First run:** the app isn't code-signed yet, so Windows SmartScreen may warn — click **More info → Run anyway**. (Signing is planned.)

> The other files in each release (`RELEASES`, `releases.win.json`, `*.nupkg`) are the **automatic-update feed** — the app reads them itself. You don't need to download those.

---

## Automatic updates

Install once with `Setup.exe`; you won't run an installer again. New versions are detected on launch,
downloaded in the background (as small **delta** patches where possible), and applied at a **safe moment**:

- **At startup / pre-flight** — applied immediately, before you start flying.
- **When a flight ends** — applied then if it became available mid-session.
- **During an active flight** — you get a toast: *"Update ready — applying now will end your flight."* with **Apply now** / **Later**. "Later" applies at the next safe moment. Your flight is never interrupted without your say-so.

---

## For maintainers — how releases are made

Releases are built by a **self-hosted GitHub Actions runner** (the maintainer's licensed dev machine)
from the private WingMate monorepo, packaged with **[Velopack](https://velopack.io)** (`vpk`), and
published here.

**Why self-hosted (not cloud CI):** the companion links against the MSFS 2024 SimConnect SDK at build
time, and the SDK EULA doesn't grant redistributing those assemblies into cloud CI — so the build runs
on the machine that's licensed to have the SDK; only the built app is published.

**Ship a new version:**
```bash
# in the monorepo
git tag vX.Y.Z && git push origin vX.Y.Z
# then build + publish from the release pipeline
gh workflow run release.yml -R svenbledt/wingmate-companion -f ref=vX.Y.Z
```
The workflow checks out the monorepo at the tag, builds the client (SimConnect gate on), runs
`vpk download → pack → upload`, and publishes a GitHub Release (full + delta + `Setup.exe` + portable).

**One-time setup:** register a self-hosted Windows runner (labels `self-hosted, windows`) with `dotnet`
+ the MSFS 2024 SDK + `vpk` (`dotnet tool install -g vpk`); add the `MONOREPO_READ_TOKEN` secret (read
access to the private monorepo). The installed client points its update feed at this repo.

**Signing:** deferred — releases are currently unsigned (add `vpk` `--signParams`/`--signTemplate` once a
code-signing certificate is available).
