# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ExplorerPatcher is a Windows shell enhancement tool that injects into `explorer.exe` to restore classic Windows 10 UI elements (taskbar, Start menu, Alt+Tab) on Windows 11. It operates via DLL injection using `dxgi.dll` proxy loading and runtime function hooking with SlimDetours.

## Build Commands

**Prerequisites:** Visual Studio 2022 (v143 toolset), CMake, NuGet, MSYS2 (for CI patches)

```bash
# Initialize submodules (required first time)
git submodule update --init --recursive

# Restore NuGet packages
nuget restore ExplorerPatcher.sln

# Build zlib dependency (run from VS Developer Command Prompt)
BuildDependenciesRelease.bat   # or BuildDependenciesDebug.bat

# Build specific platform (from VS Developer Command Prompt)
msbuild /m /p:Configuration=Release /p:Platform=amd64 ExplorerPatcher.sln
msbuild /m /p:Configuration=Release /p:Platform=arm64 /p:WithArm64XBinaries=true ExplorerPatcher.sln
msbuild /m /p:Configuration=Release /p:Platform=IA-32 ExplorerPatcher.sln

# Post-build: generate dxgi.dll and patch setup
cp build/Release/x64/ExplorerPatcher.amd64.dll build/Release/x64/dxgi.dll
build/Release/x64/ep_setup_patch.exe build/Release/x64/ExplorerPatcher.amd64.dll build/Release/x64/ep_setup.exe
```

There is no test suite. Validation is done via manual testing and CI multi-platform builds.

## Architecture

### Injection Mechanism

The core DLL (`ExplorerPatcher.amd64.dll` / `ExplorerPatcher.arm64.dll`) is deployed as `dxgi.dll` into the explorer directory. Windows loads it as a proxy DLL when `explorer.exe` starts. On load, it hooks internal Windows functions using SlimDetours (via `hooking.h` which wraps the funchook API over `KNSoft/SlimDetours`).

### Symbol Resolution

Many hooks target unexported Windows functions found via PDB symbol download. `symbols.h` defines symbol names for `explorer.exe`, `twinui.pcshell.dll`, `StartDocked.dll`, and `StartUI.dll`. The symbol system downloads/caches PDBs from the Microsoft symbol server to resolve function addresses at runtime.

### Solution Projects

| Project | Output | Purpose |
|---------|--------|---------|
| `ExplorerPatcher/` | Core DLL | Main patcher — hooks explorer.exe functions. `dllmain.c` is the primary file (~500KB) |
| `ep_gui/` | GUI DLL | Settings/properties window. `GUI.c` is the main file (~190KB) |
| `ep_setup/` | Setup EXE | Installer/uninstaller with Rijndael encryption for embedded payloads |
| `ep_setup_patch/` | Utility EXE | Patches setup executable with the built DLL |
| `ep_dwm/` | Service (submodule) | Windows service to remove rounded corners |
| `ep_startmenu/` | DLL | Start menu customization |
| `ep_weather_host/` | DLL | Weather widget COM server |
| `ep_weather_host_stub/` | DLL | Weather host stub |
| `ep_extra/` | DLL | Custom DLL chainloader for user extensions |
| `ep_extra_valinet.win7alttab/` | DLL | Windows 7 Alt-Tab module |
| `ep_generate_release_name/` | Build tool | Generates release version string |
| `ep_generate_release_description/` | Build tool | Generates release notes |

### Key Source Files

- **`ExplorerPatcher/dllmain.c`** — Central hub; all taskbar, file explorer, immersive UI, and system tray patches live here
- **`ExplorerPatcher/hooking.h`** — SlimDetours wrapper providing funchook-compatible API
- **`ExplorerPatcher/symbols.h`** — Symbol name definitions for hooked Windows DLLs
- **`ExplorerPatcher/def.h`** — Project-wide constants (registry path `Software\ExplorerPatcher`, CLSID, paths)
- **`ExplorerPatcher/utility.h`** — Shared utility functions
- **`ExplorerPatcher/SettingsMonitor.c`** — Registry change monitoring
- **`ep_gui/GUI.c`** — Properties window UI (Win32 native)

### Configuration Storage

All settings are stored in the Windows registry under `HKCU\Software\ExplorerPatcher` (modern path) and `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\ExplorerPatcher` (legacy/Start menu path).

### Dependencies (Git Submodules)

- `libs/libvalinet` — Utility library (hooking, PDB resolution, internet, toast notifications)
- `libs/sws` — Simple Window Switcher (Alt+Tab implementation)
- `libs/zlib` — Compression (built via CMake, requires `BuildDependenciesRelease.bat`)
- `ep_dwm` — Rounded corner removal service
- `ExplorerPatcher-L10N` — Localization strings

## Development Notes

- The codebase is primarily C with some C++ files for newer components (InputSwitch, TwinUIPatches, Taskbar10, etc.)
- `dllmain.c` and `GUI.c` are intentionally monolithic — patches are grouped by Windows component within these files
- `#if WITH_MAIN_PATCHER` guards separate code paths for the main DLL vs. smaller subset builds
- Platform-specific code uses `_M_ARM64` preprocessor checks
- Version is defined in `version.h` as `VER_MAJOR.VER_MINOR.VER_BUILD_HI.VER_BUILD_LO`
- The `ep_taskbar` component is downloaded from a separate release repository during CI, not built from this source
