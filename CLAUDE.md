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

<!-- GSD:project-start source:PROJECT.md -->
## Project

**ExplorerPatcher - Window Count Badge Fix**

ExplorerPatcher is a Windows shell enhancement tool that injects into `explorer.exe` to restore classic Windows 10 UI elements (taskbar, Start menu, Alt+Tab) on Windows 11. This project scope is a targeted bugfix for a stale window count badge on the Windows 11 taskbar.

**Core Value:** The window count badge on Win11 taskbar must accurately reflect the current number of windows for each app group, disappearing when only one window remains.

### Constraints

- **Codebase style**: Monolithic C files; patches are organized by `#pragma region` within dllmain.c
- **Compatibility**: Must work across Windows 11 builds (22H2+), amd64 and arm64
- **No test suite**: Validation is manual testing only
- **Hooking mechanism**: Modifications must use the existing funchook/SlimDetours pattern
<!-- GSD:project-end -->

<!-- GSD:stack-start source:codebase/STACK.md -->
## Technology Stack

## Overview
## Languages
- C (C11) - Core patching logic, utilities, setup, weather host, start menu
- C++ (C++20, `stdcpp20`) - Newer modules using COM/WinRT interop
- C# fragments - Inline MSBuild tasks for AES key generation and file encryption in `ep_setup/ep_setup.vcxproj`
- MIDL - COM interface definition in `ep_weather_host_stub/ep_weather_host.idl`
## Runtime
- Windows 10/11 (10.0.x) - Minimum target varies by feature
- `WindowsTargetPlatformVersion` = `10.0` (latest SDK)
- OS version detection at runtime via `osutility.h` helpers (`IsWindows11()`, `IsWindows11Version22H2OrHigher()`, etc.)
- Release: `/MT` (MultiThreaded - static CRT, no vcredist dependency)
- Debug: `/MTd` (MultiThreadedDebug)
## Build System
- Solution file: `ExplorerPatcher.sln`
- VCProjectVersion: 16.0 (ExplorerPatcher), 17.0 (ep_gui)
- PlatformToolset: `v143` (all projects)
- Multi-platform builds: IA-32, amd64, arm64
- ARM64X (hybrid) binaries supported via `WithArm64XBinaries` property
- ARM64EC platform configuration present in ExplorerPatcher core
# Initialize submodules (first time)
# Restore NuGet packages
# Build zlib dependency
# Build per platform
# Post-build
- Generator: `Visual Studio 17 2022`
- Built for x64 and ARM64 via `BuildDependenciesRelease.bat`
- Static CRT via `CMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded`
## Target Platforms
| Platform | Solution Name | Output Name | Notes |
|----------|---------------|-------------|-------|
| x64 / amd64 | `x64` | `ExplorerPatcher.amd64.dll` | Primary platform, `WITH_MAIN_PATCHER` defined |
| ARM64 | `ARM64` | `ExplorerPatcher.arm64.dll` | `WITH_MAIN_PATCHER` defined, ARM64X support |
| ARM64EC | `ARM64EC` | `ExplorerPatcher.arm64ec.dll` | Emulation-compatible ARM64 |
| Win32 / IA-32 | `Win32` | `ExplorerPatcher.IA-32.dll` | Subset build only (no `WITH_MAIN_PATCHER`) |
## Compiler Settings
- Warning Level: 3 (`/W3`)
- SDL Checks: enabled (`/sdl`)
- Conformance Mode: true (`/permissive-`)
- Language Standard: C++20 (`/std:c++20`)
- Multi-Processor Compilation: enabled (`/MP`)
- RTTI: disabled (`/GR-`) in ExplorerPatcher core
- Whole Program Optimization (LTCG)
- Function-Level Linking (`/Gy`)
- Intrinsic Functions (`/Oi`)
- COMDAT Folding + Optimize References (linker)
- Preprocessor: `NDEBUG`, `WINRT_NO_SOURCE_LOCATION`
- Forced include: `debug.h`
- Preprocessor: `_DEBUG`
- Win32: `__stdcall` calling convention
- x64/ARM64/ARM64EC: `__cdecl` calling convention
- x64/ARM64: `WITH_MAIN_PATCHER` preprocessor define
## NuGet Packages
- `KNSoft.SlimDetours` 1.1.4-beta - Runtime function hooking (inline detours)
- `Microsoft.Windows.ImplementationLibrary` (WIL) 1.0.250325.1 - Windows utility helpers (RAII wrappers, error handling)
- `Microsoft.Web.WebView2` 1.0.3405.78 - Chromium-based web view for weather widget
## Git Submodules
| Submodule | Path | Source | Purpose |
|-----------|------|--------|---------|
| libvalinet | `libs/libvalinet` | github.com/valinet/libvalinet | Hooking (IAT patch, exe inject), PDB resolution, internet helpers, toast notifications, OS version, memory search |
| sws | `libs/sws` | github.com/valinet/sws | Simple Window Switcher (Win10-style Alt+Tab) |
| zlib | `libs/zlib` | github.com/madler/zlib | Compression for setup package (zip/unzip) |
| ep_dwm | `ep_dwm` | github.com/valinet/ep_dwm | Windows service for rounded corner removal |
| ExplorerPatcher-L10N | `ExplorerPatcher-L10N` | github.com/valinet/ExplorerPatcher-L10N | Localization resource files |
## Linked System Libraries
| Library | Purpose | Used In |
|---------|---------|---------|
| `Psapi.lib` | Process status API | `dllmain.c`, `StartMenu.h`, `StartMenuSettings.cpp` |
| `Shlwapi.lib` | Shell lightweight utility | `dllmain.c`, `SettingsMonitor.h`, `StartMenu.h`, `StartupSound.cpp`, `updates.cpp` |
| `UxTheme.lib` | Visual styles/theming | `dllmain.c` |
| `Propsys.lib` | Property system | `dllmain.c` |
| `Comctl32.lib` | Common controls | `dllmain.c`, `HideExplorerSearchBar.h` |
| `Dwmapi.lib` | Desktop Window Manager | `dllmain.c` |
| `Dbghelp.lib` | Debug help (PDB symbols) | `dllmain.c` |
| `Userenv.lib` | User environment | `dllmain.c` (SMA patch report) |
| `Gdiplus.lib` | GDI+ drawing | `dllmain.c` |
| `ntdll.lib` | NT internal functions | `StartMenu.h` |
| `Wininet.lib` | Internet functions | `updates.h`, `utility.c` |
| `Winmm.lib` | Multimedia (delay-loaded) | `StartupSound.cpp` |
| `Wtsapi32.lib` | Terminal Services | `StartupSound.cpp` |
| `Oleacc.lib` | Accessibility | `TaskbarCenter.h` |
| `Version.lib` | Version info | `queryversion.h` |
| `Rstrtmgr.lib` | Restart Manager | `utility.h` |
| `zs.lib` | zlib static | `ep_setup.c` |
- `Winmm.dll` (configured in ExplorerPatcher.vcxproj linker settings)
## Key Output Artifacts
| Project | Output | Type |
|---------|--------|------|
| ExplorerPatcher | `ExplorerPatcher.{amd64,arm64,IA-32}.dll` | Core DLL (deployed as `dxgi.dll`) |
| ep_gui | `ep_gui.dll` | Settings GUI DLL |
| ep_setup | `ep_setup.exe` | Installer executable |
| ep_setup_patch | `ep_setup_patch.exe` | Post-build utility |
| ep_dwm | `ep_dwm_svc.exe` | Rounded corner removal service |
| ep_startmenu | `ep_startmenu.dll` | Start menu customization |
| ep_weather_host | `ep_weather_host.dll` | Weather widget (WebView2) |
| ep_weather_host_stub | `ep_weather_host_stub.dll` | Weather COM stub |
| ep_extra | `ep_extra.dll` (chainloader) | User extension loader |
| ep_extra_valinet.win7alttab | DLL | Win7 Alt-Tab module |
| ep_generate_release_name | `ep_generate_release_name.exe` | Build tool |
| ep_generate_release_description | `ep_generate_release_description.exe` | Build tool |
| ep_taskbar | `ep_taskbar.{0-5}.dll` | Taskbar variants (downloaded from separate repo during CI) |
## Version Scheme
- Format: `VER_MAJOR.VER_MINOR.VER_BUILD_HI.VER_BUILD_LO`
- Current: `26100.4946.69.7`
- Major/Minor correspond to Windows build numbers
- Pre-release flag: `VS_FF_PRERELEASE`
## Configuration
- `WithEncryption` MSBuild property enables AES-256 encryption of setup payload
- Key derived from git commit hash via SHA-256 + seeded PRNG
- Rijndael implementation in `ep_setup/rijndael-alg-fst.h`
## Platform Requirements
- Visual Studio 2022 (Community or higher) with C++ desktop workload
- Windows 10 SDK (latest)
- CMake (for zlib build)
- NuGet CLI
- Git with submodule support
- MSYS2 (for CI patches to sws)
- Windows 10 or Windows 11 (x64 or ARM64)
- No external runtime dependencies (static CRT)
- Administrator privileges for installation
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

## Overview
## Language Split: C vs C++
- Core hooking logic and DLL entry points (`ExplorerPatcher/dllmain.c`)
- Registry monitoring (`ExplorerPatcher/SettingsMonitor.c`)
- COM vtable-based interface consumption in C style (`ExplorerPatcher/StartMenu.c`, `ExplorerPatcher/ImmersiveFlyouts.c`)
- Utility functions (`ExplorerPatcher/utility.c`)
- Setup and auxiliary tools (`ep_setup/ep_setup.c`, `ep_extra/worker.c`)
- GUI logic (`ep_gui/GUI.c`)
- WRL (Windows Runtime C++ Template Library) COM class implementations (`ExplorerPatcher/InputSwitch.cpp`, `ExplorerPatcher/Taskbar10.cpp`, `ExplorerPatcher/TwinUIPatches.cpp`)
- WIL (Windows Implementation Libraries) error macros (`RETURN_IF_FAILED`, `RETURN_HR`)
- Localization with `std::vector` and `std::algorithm` (`ExplorerPatcher/Localization.cpp`)
- Startup sound playback (`ExplorerPatcher/StartupSound.cpp`)
- Update checking (`ExplorerPatcher/updates.cpp`)
#ifdef __cplusplus
#endif
#ifdef __cplusplus
#endif
## Naming Patterns
- `BOOL bOldTaskbar`, `DWORD dwIMEStyle`, `HWND hWndServiceWindow`, `HANDLE hShell32`
- Prefix `b` = boolean, `dw` = DWORD, `h` = handle, `p` = pointer, `wsz`/`sz` = string
- Global settings variables are file-scope in `ExplorerPatcher/dllmain.c` (lines ~150-240)
- `LoadSettings()`, `EntryPoint()`, `TerminateShellExperienceHost()`
- Hook functions: `<OriginalFunction>Hook` (e.g., `RtlQueryFeatureConfigurationHook`)
- Original function pointers: `<OriginalFunction>Func` (e.g., `RtlQueryFeatureConfigurationFunc`)
- OS version helpers: `IsWindows11()`, `IsWindows11Version22H2OrHigher()` (in `ExplorerPatcher/osutility.h`)
- `typedef struct _Setting { ... } Setting;` (`ExplorerPatcher/SettingsMonitor.h`)
- `typedef struct _LoadSymbolsResult { ... } LoadSymbolsResult;` (`ExplorerPatcher/symbols.h`)
- COM interfaces: `ITaskGroup`, `ITrayUI`, `IInputSwitchControl`
- `#define REGPATH "Software\\ExplorerPatcher"` (`ExplorerPatcher/def.h`)
- `#define REFRESHUI_NONE 0b000000` (binary literals for bitmasks)
- `#define CHECKFOREGROUND_ELAPSED_TIMEOUT 300`
- Timeout constants: `POPUPMENU_SAFETOREMOVE_TIMEOUT`, `POPUPMENU_BLUETOOTH_TIMEOUT`
- `ZZLaunchExplorer`, `ZZRestartExplorer`, `ZZTestBalloon`, `ZZGUI`
## Code Style
## Import Organization
#include <Shlwapi.h>
#pragma comment(lib, "Shlwapi.lib")
#include <Uxtheme.h>
#pragma comment(lib, "UxTheme.lib")
#include <wrl/implements.h>
#include <wrl/client.h>
#include <wil/result_macros.h>
#include <wil/resource.h>
## Preprocessor Usage Patterns
#if WITH_MAIN_PATCHER
#endif
#if defined(_M_X64)
#elif defined(_M_ARM64)
#endif
#if defined(DEBUG) | defined(_DEBUG)
#endif
#ifdef _WIN64
#pragma comment(linker, "/export:DllGetClassObject=_DllGetClassObject")
#else
#pragma comment(linker, "/export:DllGetClassObject=__DllGetClassObject@12")
#endif
## Function Hooking Patterns
### 1. SlimDetours (inline hooking via `funchook_prepare`)
### 2. IAT Patching (via `VnPatchIAT`)
## Error Handling
- NULL/INVALID_HANDLE_VALUE checks for handles
- `ERROR_SUCCESS` comparison for registry operations
- `rv != 0` for funchook operations (with `printf` logging)
## Memory Management
## COM Usage Patterns
## Registry Access Patterns
## String Handling
- `wcscat_s`, `wcscpy_s`, `swprintf_s` for safe string operations
- `wcslen`, `wcscmp`, `_wcsicmp` for comparisons
- `L"literal"` prefix for wide string literals
- `WCHAR wszBuffer[MAX_PATH]` for stack buffers
## Logging and Debugging
## Code Organization Within Monolithic Files
#pragma region "Windows 10 Taskbar Hooks"
#pragma endregion
#pragma region "Shell_TrayWnd subclass"
#pragma endregion
- `"Generics"` (line 316) -- utility functions
- `"Service Window"` (line 666) -- internal message window
- `"Windows 10 Taskbar Hooks"` (line 974) -- taskbar hook implementations
- `"Popup menu hooks"` (line 3573) -- context menu interception
- `"Load Settings from registry"` (line 6784) -- settings deserialization
- `"Fix taskbar for classic theme and set Explorer window hooks"` (line 8272)
- `"Explorer Registry Hooks"` (line 9663) -- registry redirect hooks
- `"Crash counter system"` (line 11177) -- crash detection
- `"Loader for alternate taskbar implementation"` (line 11415) -- ep_taskbar loading
## Comments
## Function Design
## Module Design
## Key Conventions Summary
| Convention | Rule |
|---|---|
| Language | C for core logic, C++ for WRL COM classes |
| Naming (vars) | Hungarian prefix + camelCase (`bEnabled`, `dwCount`, `hWnd`) |
| Naming (functions) | PascalCase (`LoadSettings`, `EntryPoint`) |
| Naming (hooks) | `OriginalFunc` / `OriginalHook` suffix pair |
| Naming (exports) | `ZZ` prefix for rundll32 exports |
| Naming (macros) | UPPER_SNAKE_CASE |
| Bracing | Allman style (brace on own line) |
| Organization | `#pragma region` blocks in monolithic files |
| Error handling | `SUCCEEDED(hr)` for COM; NULL checks for Win32; `printf` for diagnostics |
| Memory | `malloc`/`free` in C; `ComPtr<T>` in C++ |
| COM (C) | Manual vtable calls via `->lpVtbl->Method()` |
| COM (C++) | WRL `RuntimeClass` + `ComPtr` |
| Registry | `RegQueryValueExW` / `RegGetValueW` against `HKCU\Software\ExplorerPatcher` |
| Strings | Unicode (`WCHAR`) exclusively; `wcscat_s`, `swprintf_s` |
| Platform guards | `#if defined(_M_X64)` / `#elif defined(_M_ARM64)` |
| Build guards | `#if WITH_MAIN_PATCHER` for full vs subset builds |
| Logging | `printf()` / `wprintf()` to optional console |
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

## Pattern Overview
- The core DLL masquerades as `dxgi.dll` to be loaded by `explorer.exe` at startup via Windows DLL search order
- Once loaded, it installs inline hooks (via SlimDetours) and IAT patches (via libvalinet) on internal Windows functions
- Symbol addresses for unexported functions are resolved at runtime by downloading PDB files from the Microsoft symbol server
- All user settings are stored in the Windows registry under `HKCU\Software\ExplorerPatcher`
- The project targets three platforms: x64 (amd64), ARM64, and IA-32 (Win32)
## Layers
- Purpose: Intercept DLL loading by Windows so the patcher DLL gets loaded into `explorer.exe`, `StartMenuExperienceHost.exe`, and `ShellExperienceHost.exe`
- Location: `ExplorerPatcher/dxgi_imp.h`
- Contains: Proxy function stubs for all 18 DXGI exports that forward to the real `dxgi.dll`
- Depends on: Real `dxgi.dll` from `System32`
- Used by: `ExplorerPatcher/dllmain.c` (included at line 280, initialized at line 14345)
- Purpose: Install inline function hooks on Windows internal functions
- Location: `ExplorerPatcher/hooking.h`
- Contains: A funchook-compatible wrapper around KNSoft/SlimDetours. `funchook_prepare()` calls `SlimDetoursInlineHook()`. The `funchook_create()`, `funchook_install()`, `funchook_uninstall()`, `funchook_destroy()` functions are no-ops or stubs -- hooking is immediate on `funchook_prepare()`.
- Depends on: `KNSoft/SlimDetours` (NuGet package)
- Used by: `ExplorerPatcher/dllmain.c`, `ExplorerPatcher/Taskbar10.cpp`, `ExplorerPatcher/TwinUIPatches.cpp`
- Purpose: Redirect imported function calls in loaded DLLs (Import Address Table patching)
- Location: `libs/libvalinet` (provides `VnPatchIAT` and `VnPatchDelayIAT`)
- Contains: Functions to patch the IAT of loaded modules like `shell32.dll`, `ExplorerFrame.dll`, `twinui.pcshell.dll`
- Used by: `ExplorerPatcher/dllmain.c` `InjectBasicFunctions()` (line 10683)
- Purpose: Resolve addresses of unexported Windows functions by downloading and parsing PDB files
- Location: `ExplorerPatcher/symbols.h`, `ExplorerPatcher/symbols.c`
- Contains: Symbol name definitions for `explorer.exe` (6 symbols), `twinui.pcshell.dll` (7 symbols), `StartDocked.dll` (5 symbols), `StartUI.dll` (1 symbol). The `LoadSymbols()` function checks cached offsets in the registry; `DownloadSymbols()` fetches PDBs from Microsoft's symbol server.
- Depends on: `libs/libvalinet` (provides `VnDownloadSymbols`, `VnGetSymbols`)
- Used by: `ExplorerPatcher/dllmain.c` `Inject()` function
- Purpose: Contains all the actual behavioral patches organized by Windows component
- Location: `ExplorerPatcher/dllmain.c` (14,489 lines)
- Contains: 40+ `#pragma region` sections, each targeting a specific Windows feature (taskbar, Start menu, context menus, system tray, volume flyout, etc.)
- Depends on: All other layers
- Used by: Called from `Inject()`, `InjectStartMenu()`, `InjectShellExperienceHost()`
- Purpose: Read/write user configuration from the Windows registry
- Location: `ExplorerPatcher/dllmain.c` `LoadSettings()` (line 6792), `ExplorerPatcher/SettingsMonitor.c`
- Contains: ~60 global `DWORD` variables read from `HKCU\Software\ExplorerPatcher`. `SettingsMonitor` uses `RegNotifyChangeKeyValue` to watch 12 registry keys for changes and call `LoadSettings()` on the fly.
- Depends on: Windows Registry API
- Used by: All patch functions read from the global setting variables
- Purpose: Provides the ExplorerPatcher Properties window for user configuration
- Location: `ep_gui/GUI.c` (4,281 lines), `ep_gui/GUI.h`
- Contains: Win32 native UI (no frameworks), uses custom owner-draw rendering. Reads settings definition from `.reg` files (`ep_gui/resources/settings.reg`, `ep_gui/resources/settings10.reg`). Supports dark mode and accessibility.
- Depends on: `ExplorerPatcher/utility.h`, `ExplorerPatcher/Localization.h`, `ep_weather_host`
- Used by: Launched from the core DLL via `LaunchPropertiesGUI()` which calls `ZZGUI` export
- Purpose: Installer and uninstaller
- Location: `ep_setup/ep_setup.c`, `ep_setup_patch/ep_setup_patch.c`
- Contains: The setup extracts DLLs from an embedded encrypted (Rijndael) zip payload, copies them to `%ProgramFiles%\ExplorerPatcher`, registers the DWM service, creates shortcuts. `ep_setup_patch` is a build-time tool that embeds the built DLL into the setup executable.
- Depends on: zlib/minizip, Rijndael (`ep_setup/rijndael-alg-fst.c`), `ExplorerPatcher/utility.h`
## Data Flow
- All settings are global `DWORD`/`BOOL` variables at the top of `ExplorerPatcher/dllmain.c` (lines 149-198)
- State is centralized in the registry; the DLL reads but does not cache beyond global variables
- The GUI (`ep_gui`) writes directly to registry keys, and `SettingsMonitor` picks up changes in real-time
## Key Abstractions
- Purpose: Provides a funchook-compatible API wrapping SlimDetours
- Examples: `ExplorerPatcher/hooking.h`
- Pattern: The `funchook_prepare(funchook, &target_func, hook_func)` pattern replaces `target_func` with `hook_func` and stores the original in `target_func`. This is the primary mechanism for intercepting internal Windows functions.
- Purpose: Redirects calls to imported functions in a specific DLL
- Examples: Used extensively in `InjectBasicFunctions()` at `ExplorerPatcher/dllmain.c:10683`
- Pattern: `VnPatchIAT(hModule, "source.dll", "FunctionName", ReplacementFunction)` -- patches the IAT of `hModule` so calls to `FunctionName` imported from `source.dll` go to `ReplacementFunction` instead
- Purpose: Locate specific byte sequences in loaded modules for patching
- Examples: `ExplorerPatcher/utility.h` `FindPattern()` (line 953), used in `PatchContextMenuOfNewMicrosoftIME()` (line 817)
- Pattern: `FindPattern(baseAddr, size, bytePattern, mask)` -- mask uses `x` for exact match, `?` for wildcard
- Purpose: Hold resolved offsets for all hooked unexported functions
- Examples: `ExplorerPatcher/symbols.h` (line 56-62)
- Pattern: Packed struct with arrays of `DWORD` offsets per target DLL. Cached in registry subkeys under `HKCU\Software\ExplorerPatcher\{module_name}`.
- Purpose: Control which features of the alternate taskbar DLL to activate
- Examples: `ExplorerPatcher/dllmain.c` (line 11450-11462)
- Pattern: Bitmask flags (`EPTF_Taskbar`, `EPTF_ClassicContextMenu`, `EPTF_WinBlueLauncher`, `EPTF_AudioFlyout`)
## Entry Points
- Location: `ExplorerPatcher/dllmain.c:14426`
- Triggers: Called by `explorer.exe` and `ShellExperienceHost.exe` when they initialize DXGI
- Responsibilities: Calls `EntryPoint(DLL_INJECTION_METHOD_DXGI)`, then forwards to the real DXGI function
- Location: `ExplorerPatcher/dllmain.c:14432`
- Triggers: Called by `StartMenuExperienceHost.exe` during DXGI initialization
- Responsibilities: Same injection trigger path for the Start Menu host process
- Location: `ExplorerPatcher/dllmain.c:14446` (linker export)
- Triggers: COM activation when the DLL is registered as a COM server
- Responsibilities: Calls `EntryPoint(DLL_INJECTION_METHOD_COM)` for non-explorer processes
- Location: `ExplorerPatcher/dllmain.c:14483`
- Triggers: Called via `rundll32.exe` to open the properties GUI
- Responsibilities: Forwards to `LaunchPropertiesGUI()` which loads `ep_gui.dll`
- Location: Called at `ExplorerPatcher/dllmain.c:12659`
- Triggers: After main hook installation completes
- Responsibilities: Loads `ep_extra.dll` from `C:\Windows`, calls its `ep_extra_EntryPoint` function
## Error Handling
- Functions return `BOOL` or `HRESULT` to indicate success/failure
- Null pointer checks before every API call (e.g., `if (!hModule) return;`)
- `funchook_prepare()` return value checked against `HOOKING_SUCCESS` (0)
- Failed symbol downloads are reported via Windows toast notifications (using `DownloadNotificationXML` in `ExplorerPatcher/symbols.c`)
- Crash counter system (`ExplorerPatcher/dllmain.c:11177`) detects repeated crashes and disables EP after a configurable threshold
- Extensive use of `global_rovi.dwBuildNumber` checks to apply patches only to compatible Windows builds
- `osutility.h` provides `IsWindows11()`, `IsWindows11Version22H2OrHigher()`, etc.
- `#if defined(_M_ARM64)` / `#if defined(_M_X64)` preprocessor guards for platform-specific binary patches
- `PickTaskbarDll()` in `ExplorerPatcher/utility.h:1022` selects the correct `ep_taskbar.*.dll` variant based on the OS build number
## Cross-Cutting Concerns
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, or `.github/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
