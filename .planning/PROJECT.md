# ExplorerPatcher - Window Count Badge Fix

## What This Is

ExplorerPatcher is a Windows shell enhancement tool that injects into `explorer.exe` to restore classic Windows 10 UI elements (taskbar, Start menu, Alt+Tab) on Windows 11. This project scope is a targeted bugfix for a stale window count badge on the Windows 11 taskbar.

## Core Value

The window count badge on Win11 taskbar must accurately reflect the current number of windows for each app group, disappearing when only one window remains.

## Requirements

### Validated

- DLL injection into explorer.exe via dxgi.dll proxy loading -- existing
- Runtime function hooking with SlimDetours -- existing
- Win11 taskbar modifications and patches -- existing
- Window count badge display on taskbar icons -- existing
- Registry-based settings storage under HKCU\Software\ExplorerPatcher -- existing
- Multi-platform support (amd64, arm64, IA-32) -- existing

### Active

- [ ] Window count badge disappears when closing windows down to one on Win11 taskbar
- [ ] Badge count accurately reflects the current window count at all times

### Out of Scope

- Win10-style taskbar badge issues -- different code path, not reported
- New feature additions -- this is a bugfix-only scope
- GUI/settings changes -- no settings are involved in this bug
- Changes to other taskbar elements (system tray, Start menu, etc.)

## Context

**Bug description:** On the Windows 11 taskbar, when a user has multiple windows of the same application open, a count badge is displayed on the taskbar icon. When windows are closed down to a single window, the badge persists showing the old (stale) count instead of disappearing. The badge remains stuck until explorer.exe is restarted.

**Technical environment:**
- Core patching logic lives in `ExplorerPatcher/dllmain.c` (~14,500 lines)
- Win11 taskbar patches are in `#pragma region` sections within dllmain.c
- Related C++ modules: `ExplorerPatcher/TwinUIPatches.cpp`, `ExplorerPatcher/Taskbar10.cpp`
- Hooking via SlimDetours (`ExplorerPatcher/hooking.h`)
- Recent commit `f157168` addressed a related issue: "fix: deduplicate overlapping window count badges on Win11 taskbar"

**Known related work:**
- Commit `f157168` recently fixed deduplicate overlapping window count badges -- the stale badge bug may be related or a regression from that fix

## Constraints

- **Codebase style**: Monolithic C files; patches are organized by `#pragma region` within dllmain.c
- **Compatibility**: Must work across Windows 11 builds (22H2+), amd64 and arm64
- **No test suite**: Validation is manual testing only
- **Hooking mechanism**: Modifications must use the existing funchook/SlimDetours pattern

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Fix in existing Win11 taskbar code path | Bug is specific to Win11 taskbar, not Win10 style | -- Pending |
| Investigate recent badge dedup commit first | Commit f157168 touched badge code recently and may be related | -- Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? -> Move to Out of Scope with reason
2. Requirements validated? -> Move to Validated with phase reference
3. New requirements emerged? -> Add to Active
4. Decisions to log? -> Add to Key Decisions
5. "What This Is" still accurate? -> Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check -- still the right priority?
3. Audit Out of Scope -- reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-15 after initialization*
