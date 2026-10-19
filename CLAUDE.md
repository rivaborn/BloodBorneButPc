# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Bloodborne (PS4, CUSA03173, update 1.09) running natively on Windows: a WPF launcher plus a reworked engine. No game files are in the repo, and none may ever ship (see the release check below).

Three independent parts:

| Part                       | Language                                          | Licence          |
| -------------------------- | ------------------------------------------------- | ---------------- |
| `launcher/windows/`        | C# / WPF, .NET 8 (`net8.0-windows`), WPF-UI 4.3.0 | GPL-2.0-or-later |
| `engine/`                  | C/C++ (patch on an upstream tree), Python scripts | GPL-3.0-or-later |
| `tools/release-assets.ps1` | PowerShell                                        | —                |

## The engine is a patch, not a source tree

The engine source is **not checked in**. `engine/bbhost-changes.patch` is one large `git diff` (~276 files: `gpu/bbnet`, `gpu/shadps4`, `gpu/shim`, `src/runtime_*.c`, `scripts/`, `tests/`, `build.sh`, CI) against [DarkIzuku/bloodborne_pc](https://github.com/DarkIzuku/bloodborne_pc) at tag `v0.4.0-darkizuku-preview.1` (itself built on bbport + shadPS4).

- Engine changes are made by regenerating/editing this patch (e.g. commit 2d26649 touched only the patch). To work on engine code: clone upstream at that tag, `git apply` the patch, edit, then `git diff` back into `engine/bbhost-changes.patch`. `.gitattributes` marks `*.patch -text` — keep line endings byte-exact.
- **Line-ending trap:** the patch's `+` lines carry CRLF, but the author's real files are LF (the `index A..B` hashes in the patch are of the LF blobs; upstream is LF too). A plain `git apply` therefore yields ~230 CRLF files (including `build.sh`, which bash then fails on) and inflates the diff to +120k/−79k instead of the real +55.8k/−14.1k. Apply with `core.autocrlf=false`, then convert each patched file to LF where that makes its blob hash equal the patch's `B` hash (one 100%-similarity rename, `net/net_util.h`, also needs its original content back). Verify with `git diff <tag> HEAD | grep '^index '` matching the patch's own `index` lines.
- `engine/online/` is a readable copy of the online subsystem that also lives inside the patch (`gpu/bbnet/...`, `scripts/online.py`). Keep the two in step if you change either. `engine/online/SHADNET-PORT.md` documents provenance: shadPS4 v.0.20.0 network/NP/shadNet sources (changes marked `bbport`) and the `thd/` backend adapted from droogie/bbhost for The Hunter's Dream pool.
- `engine/win_compat.h` is copied to `gpu/` before building (newer libc++ dropped headers the FSR 3 SDK expects).
- `engine/README.md` is the changelog of every engine change and its `BB_*` environment switch / `bbport.ini` key. Add a row there when adding an engine behaviour or switch.

### Building the engine (MSYS2 **CLANG64** shell)

```bash
git clone --branch v0.4.0-darkizuku-preview.1 --recurse-submodules https://github.com/DarkIzuku/bloodborne_pc
cd bloodborne_pc
git apply /path/to/engine/bbhost-changes.patch
cp /path/to/engine/win_compat.h gpu/
CFLAGS="-Wno-error=unused-but-set-global" CXXFLAGS="-include $(cygpath -m "$PWD")/gpu/win_compat.h" bash build.sh
```

Output: `out/bb-probe.exe`. PGO with clang: `BB_PGO=generate`, then `BB_PGO=use`. `cygpath -m` is required: clang++ is a native Windows program and cannot open `/c/...` paths (the `$PWD` form in `engine/README.md` fails). The patch already contains `gpu/win_compat.h`, so the `cp` step is redundant. Submodules also need `core.autocrlf=false`, or `build.sh`'s LF patch to `fsr-vulkan` fails to apply.

A ready-made patched tree with these fixes, a Windows install script (`tools/install-windows.ps1`) and the line-ending repair (`tools/bbhost_eol_fix.py`) is `rivaborn/BloodbornePC`, branch `bbhost`.

Engine tests (in the patched tree; list mirrors its `.github/workflows/windows-offline-build.yml`): build a single test target with `ninja -C out/gpu <name>-test` and run `out/gpu/<name>-test.exe` (e.g. `occlusion-test`, `settings-test`, `engine-hooks-test`, `widescreen-test`). Python tests live in `tests/test_*.py` of the patched tree.

## Launcher (`launcher/windows/`)

```powershell
dotnet build launcher/windows/BloodborneLauncher.sln
dotnet run --project launcher/windows/LauncherBuilder -- --force --no-pause   # "Build Launcher.exe"
```

- `BloodborneLauncher/` — the WPF app. `Core/` holds logic (game launch, settings, updater, online, mods, trophies, save import, input maps, memory/hardware checks); `Views/` holds one class per page, built in code (not XAML) apart from `MainWindow.xaml` and `Themes/`.
- `LauncherBuilder/` — "Build Launcher.exe": fingerprints the launcher sources, and if changed publishes the launcher into `<repo>/LauncherApp/` with `build-stamp.json`, replaces itself, then starts the launcher. Flags: `--force`, `--check` (exit 2 if a rebuild is needed), `--no-launch`, `--no-pause`, `--wait-pid N`.
- `Shared/SourceStamp.cs` — linked into **both** projects so the builder and the launcher hash the same files the same way. Change it in one place only.
- `BloodbornePlay/` — tiny Win32 C stub (`main.c` + `play.rc`) that becomes `Bloodborne.exe`: runs `"Bloodborne Launcher.exe" --play <args>` beside it and waits (for shortcuts / Steam non-Steam-game entries). No build script for it in this repo.

### How the launcher drives the engine

The launcher does not run the engine directly; it starts `scripts\run_windows.py` with the package's bundled Python (`tools\python\python.exe`), which launches `out\bb-probe.exe` (`Core/GameRunner.cs`). A folder counts as a port/package if it has both `scripts\run_windows.py` and `out\bb-probe.exe` (`Core/Paths.cs`). Engine settings live in `bbport.ini` (plus `mods.json`, `patches.json`) in `BB_DATA_DIR` (default: the package folder; `BB_CONFIG` overrides the ini path). Launcher-only settings are per-user in `%APPDATA%\bbport-launcher\settings.json`. Most launcher options map to `bbport.ini` keys / `BB_*` variables documented in `engine/README.md`.

## Releases

```powershell
tools\release-assets.ps1 -Version 0.9.13 -Engine bbhost-0.4.0-fps23 -Package D:\BloodbornePC -Dist D:\dist
```

Builds a self-contained single-file launcher exe (what the updater downloads) and `BloodbornePC-v<Version>-win64.zip` from a working package folder. It strips dev files and everything generated from the player's dump (`out\` keeps only `.exe/.dll/.ico`; `user`, `logs`, `mods`, `network` excluded), resets `bbport.ini` binds to the EldenRingStyle layout, writes `package.json` (`version` + `engine`; the updater compares `engine` to decide between a launcher-only and a full-package update), and **refuses to produce the zip** if any game-derived file (eboot, `out\*.bin/elf/json/txt`, `user\`) slipped in. Publishing a GitHub release triggers `.github/workflows/discord-release.yml` (pre-release → beta channel, else stable).

Versions in the `.csproj` files (0.9.9) are stale defaults; the real version is passed via `-p:Version` at release time. Commit messages follow `0.9.NN: <user-visible changes>` for releases.

## Conventions

- Comments and docs are written as plain prose describing behaviour and the reason (often naming the in-game symptom it fixes); match that style.
- Code adapted from other forks (IFreemz, AwesomeObserver, Supermedo, deadinside28, droogie/bbhost) credits its author in the code — keep those credits.
