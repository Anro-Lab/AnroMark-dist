# AnroMark-dist

Public distribution channel for **AnroMark**.

## Purpose

This repository is a write-only distribution endpoint. It hosts the Windows desktop installer artifacts that both the AnroMark Launcher and the in-app `electron-updater` auto-update system use.

- **No source code lives here.** Source repo: [`Anro-Lab/AnroMark-win`](https://github.com/Anro-Lab/AnroMark-win).
- **Anonymous read** is allowed — end users do not need GitHub credentials.
- **Issues are enabled here, but product problems belong in the source repo:**
  https://github.com/Anro-Lab/AnroMark-win/issues. Audit failures are reported there too, not here.

## Release Flow

Stable releases (`vX.Y.Z`) are published to this repo **only by CI**:

```
AnroMark-win production build (build-electron-complete.bat, production mode)
    └─> creates release vX.Y.Z on Anro-Lab/AnroMark-win
        (ZIP + NSIS installer + portable + latest.yml)
    └─> AnroMark-win .github/workflows/notify-dist-release.yml
        sends repository_dispatch `audit_release` to this repo
                          │
                          ▼
    .github/workflows/audit-and-publish.yml (this repo)
        1. Guard: refuse unless the last completed audit-self-test run succeeded
        2. Download the release assets from the AnroMark-win release
           (must exist and must not be a draft)
        3. Run the audit gate: forbidden paths, gitleaks, trufflehog,
           size policy, integrity
        4. PASS  -> publish vX.Y.Z here (all assets)
           FAIL  -> nothing is published; an issue is opened on AnroMark-win
```

Any step failing before the audit (release not found, download error, ...) also opens the issue on AnroMark-win. A manual dry run is possible with `gh workflow run audit-and-publish.yml -f tag=vX.Y.Z -f dry_run=true`: it audits but publishes nothing and opens no issue. Tags containing `-` are always forced to dry run.

Stale drafts (>24h, never published) are cleaned by `stale-draft-cleanup.yml` daily.

### Exceptions to "stable releases come only from CI"

- **Runtime tags** (`ollama-runtime-*`, `llama-runtime-*`) are uploaded **by hand**. They hold upstream binaries and do not go through the audit gate. Integrity is enforced by the SHA-256 pins that the app checks before using a download: `electron/src/services/ollama-pins.json` in AnroMark-win (for `ollama-runtime-*`). There is no pins file for `llama-runtime-*` in AnroMark-win yet.
- **Test prereleases** (`vX.Y.Z-test.N`, beta channel) are **today uploaded directly to this repo by the build script in test mode, without this audit**. Moving them behind the audit is planned but not done.

## Audit Gate Summary

| Check | Tool | Failure mode |
|-------|------|--------------|
| Secret scan | gitleaks (`audit-rules/gitleaks.toml`) + trufflehog (verified only) | Any hit -> FAIL |
| Service-account JSON | Custom regex `"type"\s*:\s*"service_account"` + `*-firebase-adminsdk-*.json` | Hit -> FAIL |
| Forbidden paths | `audit-rules/forbidden-paths.txt` (`*.ts`, `*.env`, `src/`, `*.pem`, `*.key`, `tsconfig.json`, `.git/`) | Hit -> FAIL |
| Size policy | `audit-rules/size-policy.yml` (warn >=50%, fail >=100% delta of the ZIP vs the previous stable `vX.Y.Z` release's ZIP; runtime and test tags are ignored; skipped with a warning annotation if no baseline exists) | >=100% -> FAIL |
| Integrity | Must contain `win-unpacked/` dir + `.exe` + `resources/app.asar` (Electron structure) | Missing -> FAIL |

The audit runs on the unpacked ZIP only. The audit workflow is itself protected by a metamorphic self-test (`audit-self-test.yml`, daily and on pushes touching the audit files) that runs the gate against fixed clean / dirty fixtures. If a fixture gives the wrong result, the self-test fails, an alarm issue is opened on AnroMark-win, and `audit-and-publish` refuses to publish until the last completed self-test run is green (a missing or unreadable result also blocks publishing).

## Version Mapping

For any tag `vX.Y.Z`:
- AnroMark source tag: `Anro-Lab/AnroMark-win@vX.Y.Z`
- Dist artifact: this repo's release `vX.Y.Z`
- Launcher embedded default manifest points to the latest known-good release here.

## Policies

- **No source code.** PRs adding `.go`, `.ts`, `.tsx`, `.py`, `.js`, or any `src/` content will be rejected.
- **Issues about the product are filed in the source repo** (link above).
- **Stable releases are created only by the `audit-and-publish` workflow.** Manual creation of a stable `vX.Y.Z` release is against policy. The only exceptions are the hand-uploaded runtime tags and the directly uploaded test prereleases described under Release Flow.

## Release Asset Format

Each stable release (`vX.Y.Z`) contains **four assets**. Test prereleases (`vX.Y.Z-test.N`) carry the same four, except that the channel file is `beta.yml` instead of `latest.yml`, plus the sideload packages `AnroMark-vX.Y.Z-test.N-win32-x64.msix` / `.appx` (and `AnroMark-codesign.cer` for signed builds). The stable channel file is `latest.yml`.

### 1. `AnroMark-vX.Y.Z-win32-x64.zip` — App bundle for the Go Launcher

Contains `win-unpacked/` with the full Electron application. The Go Launcher downloads this ZIP and installs it in-place.

```
AnroMark-vX.Y.Z-win32-x64.zip
└── win-unpacked/
    ├── AnroMark.exe          ← main Windows executable
    ├── locales/
    ├── resources/
    │   ├── app.asar         ← Electron app bundle
    │   └── ...
    └── *.dll / *.pak / ...
```

> **Integrity check** runs against this ZIP: must contain `win-unpacked/`, an `.exe`, and `resources/app.asar`.

### 2. `AnroMark-vX.Y.Z-win32-x64.exe` — NSIS Installer

Full NSIS installer for the app. Used by:
- **In-app auto-update** (`electron-updater` fallback path): if `latest.yml` lookup fails, the updater finds this asset, downloads it, and spawns it with UAC elevation via PowerShell `Start-Process -Verb RunAs`.
- **Manual installation** by end users or IT admins.

> Filename (hyphenated, `v`-prefixed version, `win32-x64` platform tag) matches the `url:` field in `latest.yml`.

### 3. `AnroMark-vX.Y.Z-win32-x64-portable.exe` — Portable build

Standalone portable executable for users who cannot install. Not consumed by the auto-updater.

### 4. `latest.yml` — electron-updater version manifest

Generated by `electron-builder`. Consumed by `electron-updater`'s primary update path:

```yaml
version: X.Y.Z
files:
  - url: AnroMark-vX.Y.Z-win32-x64.exe
    sha512: <base64-sha512>
path: AnroMark-vX.Y.Z-win32-x64.exe
sha512: <base64-sha512>
releaseDate: 'YYYY-MM-DDTHH:MM:SS.sssZ'
```

`electron-updater` fetches this file via the GitHub provider, compares the version, prompts the user, then downloads and verifies (SHA-512) the NSIS installer before running it.

## Release Notes

> Historical notes. This section records an old release (v0.10.17) and does not describe the current flow. Current release notes live on each GitHub release.

### v0.10.17 (history)

**Auto-Update Flow Fixes** — resolved multiple issues that prevented in-app updates from completing:

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Update appeared to complete but app stayed on old version | `app.quit()` is async; Electron process kept file lock; NSIS installer couldn't overwrite EXE | Changed to `app.exit(0)` (immediate); PowerShell `Wait-Process` waits for process to fully exit before launching installer |
| SHA-512 checksum mismatch | `electron-builder` generates `latest.yml` SHA-512 before code signing; signing changes EXE bytes | Build script recomputes SHA-512 of signed EXE and updates `latest.yml` after signing |
| Publisher verification failure | `publisherName: "Anrotec"` only matches CN field; cert DN is `CN=AnroOne Pro Desktop` | Updated `publisherName` to full DN; also disabled verification via `_verifyUpdateCodeSignature` override |
| Double percentage in splash screen | `updateProgress` always appended `percent%` even when `customStatus` already contained a percentage | Added early return when `customStatus` is provided |

---

## Runner

All workflows run on `ubuntu-latest` (GitHub-hosted).
