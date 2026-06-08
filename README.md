# WingMate companion — release pipeline (Phase 19b)

This folder holds the **ready-to-drop-in release pipeline** for the WingMate desktop companion
(`client/`, WPF .NET 8). It is authored in the monorepo for review, but it is meant to be **copied
into a SEPARATE personal-GitHub repo** (e.g. `<user>/wingmate-companion-releases`) that hosts the
GitHub Releases feed the installed app updates from.

- `release.yml` → copy to `.github/workflows/release.yml` in the release repo.
- `README.md` (this file) → the setup + runbook.

Cross-references:
- Design spec: [`docs/superpowers/specs/2026-06-08-desktop-self-updater-design.md`](../superpowers/specs/2026-06-08-desktop-self-updater-design.md)
- Roadmap: `PLAN.md` → **Phase 19 — Desktop self-updater** (19a = client Velopack integration; 19b = this pipeline).

---

## What this is

The companion uses [Velopack](https://velopack.io) for install-once + auto-update. Each version is
published as a **GitHub Release** (a full package, a delta against the prior release, and a
`Setup.exe` bootstrapper). The installed client's `GitHubSource` feed polls that repo's releases,
downloads new versions, and applies them at a safe window (see the client orchestrator, Phase 19a).

### Why a SEPARATE repo

The org repo (`WingMateMSFS/wingmate`) is at its GitHub Actions minutes cap. The release pipeline
lives on the maintainer's **personal** GitHub account so it runs independently of org limits. The
release repo holds only this workflow (and later, signing config) — it does **not** hold client
source. It checks out the monorepo at a pinned tag and builds from there, so `client/` stays
single-sourced in the monorepo.

### Why a SELF-HOSTED runner (not cloud CI)

The MSFS 2024 SDK EULA (11/2019) does **not** explicitly grant redistributing the SimConnect client
assemblies (`Microsoft.FlightSimulator.SimConnect.dll` + native `SimConnect.dll`) — §2(e) bars
publishing/distributing "the Software", and the docs' "redistributable" wording is about add-on
content packages, not the SimConnect DLLs. So we do **not** vendor Microsoft's DLLs into a public
cloud CI. Instead the build runs on a self-hosted runner on the maintainer's machine, which is
already licensed to have the SDK installed. The SDK never leaves that machine; only the built app is
published — the same `SimConnect.dll`-bundled redistribution the client already does by copying the
native DLL next to its exe at build time.

---

## One-time setup

1. **Create the personal repo**, e.g. `<user>/wingmate-companion-releases` (can be public — only the
   built artifacts land here; no source).
2. **Add the workflow:** copy `release.yml` from this folder to `.github/workflows/release.yml` in
   the release repo.
3. **Register a self-hosted Windows runner** on the dev machine: repo **Settings → Actions →
   Runners → New self-hosted runner** (Windows). Apply the labels **`self-hosted`** and
   **`windows`** (the workflow targets `runs-on: [self-hosted, windows]`). Run it as a service so
   tag pushes trigger builds unattended.
4. **Install the Velopack CLI** on the machine:
   ```pwsh
   dotnet tool install -g vpk
   ```
   (Ensure the global tools dir is on `PATH` so the runner can invoke `vpk`.)
5. **Install / verify the MSFS 2024 SDK** at the csproj default path **`C:\MSFS 2024 SDK\`** (SDK
   1.6.9). The csproj resolves:
   - managed: `C:\MSFS 2024 SDK\SimConnect SDK\lib\managed\Microsoft.FlightSimulator.SimConnect.dll`
   - native:  `C:\MSFS 2024 SDK\SimConnect SDK\lib\SimConnect.dll`

   The monorepo's machine-local `client/Directory.Build.props` (which sets `SimConnectSdkAvailable`
   + the SDK paths) is **gitignored**, so the checked-out copy on the runner does NOT have it. The
   workflow therefore forces the gate with **`-p:SimConnectSdkAvailable=true`** and relies on these
   default paths. If your SDK lives elsewhere, add to the publish step:
   `-p:MsfsSdkManagedPath=... -p:MsfsSdkNativePath=...`.
6. **Add the repo secret `MONOREPO_READ_TOKEN`** (release repo → **Settings → Secrets and variables
   → Actions**): a fine-grained PAT (or deploy key) with **read** access to the private
   `WingMateMSFS/wingmate`. `GITHUB_TOKEN` is provided automatically by Actions (scoped to the
   release repo) — used for the delta fetch and the release upload; no setup needed.

---

## Configure the client (one-time, in the monorepo)

Point the installed app at the release feed. In `client/src/WingMate.Companion/appsettings.json`,
set `WingMate:UpdateFeedUrl` to the release repo URL:

```json
{
  "WingMate": {
    "UpdateFeedUrl": "https://github.com/<user>/wingmate-companion-releases"
  }
}
```

An empty `UpdateFeedUrl` disables the updater (the default until the repo exists). This must be set
before the version you cut is the one users install.

---

## Release flow

1. **Tag the monorepo** with the version you're shipping:
   ```pwsh
   git tag v1.2.3 && git push origin v1.2.3
   ```
   The release repo is what hosts the workflow, but the **tag pattern `v*`** trigger fires on tags
   pushed to the release repo. For a tag-driven run, push the `v*` tag in the release repo; to build
   a specific monorepo commit/tag, prefer **`workflow_dispatch`** (Actions → release → Run workflow)
   and pass the monorepo **`ref`** to build (e.g. `v1.2.3`).
2. The self-hosted runner:
   - checks out `WingMateMSFS/wingmate` at the ref,
   - publishes `client/` in Release with the SimConnect gate on,
   - `vpk download github` fetches the prior release (delta base; non-fatal on the first release),
   - `vpk pack` builds the **full + delta + `Setup.exe`**,
   - `vpk upload github --publish` publishes the GitHub Release.
3. Installed clients see the new release on their next feed check and stage it for a safe apply.

---

## Validation runbook (first real run — human-driven)

1. **Cut the first release** (any `v0.x.y`). Confirm the release repo gets a published Release with
   `Setup.exe`, the full `.nupkg`, and a `RELEASES` manifest (no delta yet — first release).
2. **Install** the downloaded `Setup.exe`. (Unsigned for now → **SmartScreen warns on first install
   only**; "More info → Run anyway".) Launch the companion; confirm it runs and the Velopack
   bootstrap did not interfere with the in-flight readout.
3. **Bump + tag again** (`v0.x.(y+1)`); let the workflow build + publish. Confirm this release now
   includes a **delta** package alongside the full one.
4. **Verify auto-update behavior** on the already-installed client (Phase 19a orchestrator):
   - **Idle / pre-flight:** with no active flight, the staged update applies at a safe window
     (startup or when the session is Idle) and the app restarts on the new version.
   - **During an active flight:** the client must **NOT** silently apply mid-flight. When the
     download finishes during streaming, the **"Apply now / Later"** toast appears (stating that
     applying now ends the flight). "Later" keeps it staged; it then applies at **flight-end** (when
     the session returns to Idle) or next startup. "Apply now" applies immediately.
5. Confirm the post-update version string reflects the new tag.

---

## Deferred

- **Code signing.** Releases are unsigned until a code-signing certificate exists; SmartScreen warns
  on first install only. When a cert is available, add `vpk pack` sign params (verified flag names,
  Velopack docs):
  - Windows signtool: `--signParams "/td sha256 /fd sha256 /f cert.pfx /tr <rfc3161-timestamp-url>"`
  - or cross-platform JSign: `--signTemplate "jsign ... {{file}}"`
- **Auto-release on every monorepo push** (tag-driven for now).
- **GitHub-hosted / vendored-DLL CI** (blocked on written Microsoft permission to redistribute the
  SimConnect assemblies — not pursued).

---

## vpk CLI notes (verified against Velopack docs via Context7)

- `vpk download github` — `--repoUrl` (required), `--token`, `--outputDir/-o` (default `Releases`),
  `--pre`, `--channel/-c`, `--timeout`. The workflow passes `--token ${{ secrets.GITHUB_TOKEN }}`
  (the snippet omitted it) and an explicit `--outputDir` matching the pack output so the delta base
  is found.
- `vpk pack` — `-u/--packId` (req), `-v/--packVersion` (req), `-p/--packDir` (req), `-o/--outputDir`
  (default `Releases`), `-e/--mainExe`, `--delta <BestSize|BestSpeed|None>` (default `BestSpeed`),
  `-i/--icon`, `--releaseNotes`, `--channel/-c`. Windows signing uses `--signParams` / `--signTemplate`
  (the `--signAppIdentity`/`--notaryProfile` flags are macOS-only).
- `vpk upload github` — `--repoUrl` (required), `--token`, `--outputDir/-o` (source dir, default
  `Releases`), `--publish`, `--pre`, `--merge`, `--releaseName`, `--tag`, `--targetCommitish`,
  `--channel/-c`. The workflow points `--outputDir` at the same dir `vpk pack` wrote to.
