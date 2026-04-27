# Fullscreen Badge Hide Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Win11 taskbar window-count badge reliably hide when any foreground window covers an entire monitor (browser F11, borderless-fullscreen games, exclusive D3D), and reappear when fullscreen ends — including correct multi-monitor behavior.

**Architecture:** Single-function rewrite of `IsForegroundFullScreen()` in `ExplorerPatcher/dllmain.c`. The function gains an `HWND hTaskbarWnd` parameter and uses monitor-geometry as the authoritative signal (with `SHQueryUserNotificationState` as a fast positive shortcut for exclusive D3D). The unreliable "Shell_TrayWnd visible → not fullscreen" early-return is removed. Both badge-update call sites pass their taskbar HWND. No new dependencies, symbols, or platform branches.

**Tech Stack:** C (C11), Win32 (User32, Shell32), MSBuild + Visual Studio 2022 v143 toolset, amd64/arm64/IA-32 platforms. **No test suite** — validation is by manual repro per the spec.

**Spec:** [docs/superpowers/specs/2026-04-27-fullscreen-badge-hide-design.md](../specs/2026-04-27-fullscreen-badge-hide-design.md)

---

## File Structure

Only one source file is touched:

| File | Responsibility | Change |
|---|---|---|
| `ExplorerPatcher/dllmain.c` | Core hook DLL; contains `IsForegroundFullScreen()` and both badge update functions | Rewrite one function, update two call sites |

No new files. No header changes (the function is `static` and used only within `dllmain.c`).

---

## Task 1: Rewrite `IsForegroundFullScreen()`

**Files:**
- Modify: `ExplorerPatcher/dllmain.c:1351-1390`

The current implementation (for reference — this entire block is being replaced):

```c
// Check if a foreground window is covering the full screen (F11 fullscreen or D3D exclusive)
static BOOL IsForegroundFullScreen(void)
{
    // D3D exclusive fullscreen
    QUERY_USER_NOTIFICATION_STATE quns = QUNS_ACCEPTS_NOTIFICATIONS;
    if (SUCCEEDED(SHQueryUserNotificationState(&quns)))
    {
        if (quns == QUNS_RUNNING_D3D_FULL_SCREEN)
            return TRUE;
    }

    // If the taskbar is visible, we're not in fullscreen
    HWND hTray = FindWindowW(L"Shell_TrayWnd", NULL);
    if (hTray && IsWindowVisible(hTray))
    {
        RECT rcTray;
        if (GetWindowRect(hTray, &rcTray) && (rcTray.bottom - rcTray.top) > 0)
            return FALSE;
    }

    // Windowed fullscreen (F11 in browsers, etc.)
    HWND hFg = GetForegroundWindow();
    if (!hFg) return FALSE;

    // Ignore desktop and shell windows
    if (hFg == GetDesktopWindow() || hFg == GetShellWindow()) return FALSE;

    HMONITOR hMon = MonitorFromWindow(hFg, MONITOR_DEFAULTTONEAREST);
    if (!hMon) return FALSE;

    MONITORINFO mi = { sizeof(mi) };
    if (!GetMonitorInfoW(hMon, &mi)) return FALSE;

    RECT rcWnd;
    if (!GetWindowRect(hFg, &rcWnd)) return FALSE;

    // Window covers the entire monitor work area or screen area
    return (rcWnd.left <= mi.rcMonitor.left && rcWnd.top <= mi.rcMonitor.top &&
            rcWnd.right >= mi.rcMonitor.right && rcWnd.bottom >= mi.rcMonitor.bottom);
}
```

- [ ] **Step 1: Read the current function in context to confirm exact line bounds**

Use Read on `ExplorerPatcher/dllmain.c` lines 1345–1395 to confirm the function still occupies lines 1351–1390 (file is large and may have shifted slightly during edits).

- [ ] **Step 2: Replace the function with the new implementation**

Use Edit to replace the entire current function (from the leading comment `// Check if a foreground window is covering the full screen ...` through the closing `}`) with this new version:

```c
// Check if a foreground window covers the entire monitor that hTaskbarWnd lives on.
// Detects F11/borderless fullscreen and D3D exclusive fullscreen. The unreliable
// "Shell_TrayWnd visible → not fullscreen" early-return has been removed because the
// Win11 auto-hidden taskbar still reports IsWindowVisible == TRUE while slid off-screen.
// hTaskbarWnd may be NULL; in that case the foreground window's monitor is used.
static BOOL IsForegroundFullScreen(HWND hTaskbarWnd)
{
    // 1. Fast positive: exclusive D3D fullscreen
    QUERY_USER_NOTIFICATION_STATE quns = QUNS_ACCEPTS_NOTIFICATIONS;
    if (SUCCEEDED(SHQueryUserNotificationState(&quns)) && quns == QUNS_RUNNING_D3D_FULL_SCREEN)
        return TRUE;

    // 3. Foreground window required for the geometry check
    HWND hFg = GetForegroundWindow();
    if (!hFg) return FALSE;

    // 2. Resolve target monitor — prefer the taskbar's monitor; fall back to foreground's
    HMONITOR hMonTarget = hTaskbarWnd ? MonitorFromWindow(hTaskbarWnd, MONITOR_DEFAULTTONULL) : NULL;
    if (!hMonTarget) hMonTarget = MonitorFromWindow(hFg, MONITOR_DEFAULTTONULL);
    if (!hMonTarget) return FALSE;

    // 4. Filter shell / desktop / our own / explorer-owned windows
    if (hFg == GetDesktopWindow() || hFg == GetShellWindow()) return FALSE;

    WCHAR wszClass[64] = { 0 };
    if (GetClassNameW(hFg, wszClass, ARRAYSIZE(wszClass)))
    {
        if (!wcscmp(wszClass, L"Shell_TrayWnd") ||
            !wcscmp(wszClass, L"Shell_SecondaryTrayWnd") ||
            !wcscmp(wszClass, L"WorkerW") ||
            !wcscmp(wszClass, L"Progman") ||
            !wcscmp(wszClass, MSTASKLISTWCLASS_BADGE_OVERLAY_CLASS))
        {
            return FALSE;
        }
    }

    DWORD dwFgPid = 0;
    GetWindowThreadProcessId(hFg, &dwFgPid);
    if (dwFgPid && dwFgPid == GetCurrentProcessId()) return FALSE;

    // 5. Same-monitor requirement (multi-monitor: don't hide badges on other displays)
    HMONITOR hMonFg = MonitorFromWindow(hFg, MONITOR_DEFAULTTONULL);
    if (hMonFg != hMonTarget) return FALSE;

    // 6. Geometry check: foreground window covers the entire monitor
    MONITORINFO mi = { sizeof(mi) };
    if (!GetMonitorInfoW(hMonTarget, &mi)) return FALSE;

    RECT rcWnd;
    if (!GetWindowRect(hFg, &rcWnd)) return FALSE;

    return (rcWnd.left   <= mi.rcMonitor.left   &&
            rcWnd.top    <= mi.rcMonitor.top    &&
            rcWnd.right  >= mi.rcMonitor.right  &&
            rcWnd.bottom >= mi.rcMonitor.bottom);
}
```

Notes for the engineer:
- `MSTASKLISTWCLASS_BADGE_OVERLAY_CLASS` is already defined earlier in the same file (around line 1175) as `L"EP_TaskbarBadgeOverlay"`, so no new define is needed.
- `ARRAYSIZE` is the standard Win32 macro from `winnt.h`/`shellapi.h`; already in scope here.
- The numeric step comments (1, 3, 2, 4, 5, 6) match the spec's logical decision order — they're slightly out of order in code only because we need `hFg` before resolving the fallback target monitor.

- [ ] **Step 3: Verify no other call sites exist**

Run from `q:/src/ExplorerPatcher`:

```bash
rtk grep -n "IsForegroundFullScreen" ExplorerPatcher/dllmain.c
```

Expected: exactly three lines — the definition and two call sites at approximately lines 1625 and 1935. If a fourth match exists, stop and reconcile before proceeding.

---

## Task 2: Update both call sites to pass the taskbar HWND

**Files:**
- Modify: `ExplorerPatcher/dllmain.c:1625` (in `MSTaskListWClass_UpdateBadgeOverlay`)
- Modify: `ExplorerPatcher/dllmain.c:1935` (in `Win11Taskbar_UpdateBadgeOverlay`)

- [ ] **Step 1: Update the Win10-style taskbar call site**

In `MSTaskListWClass_UpdateBadgeOverlay`, find the line:

```c
    if (IsForegroundFullScreen())
```

…inside the block beginning with the comment `// Hide badge when a fullscreen window is active (D3D exclusive or F11 windowed fullscreen)` (around line 1624–1625). Change it to:

```c
    if (IsForegroundFullScreen(hTaskListWnd))
```

`hTaskListWnd` is the function's HWND parameter and is already in scope.

- [ ] **Step 2: Update the Win11 taskbar call site**

In `Win11Taskbar_UpdateBadgeOverlay`, find the line:

```c
    if (IsForegroundFullScreen())
```

…inside the block beginning with the comment `// Hide badge when a fullscreen window is active` (around line 1934–1935). Change it to:

```c
    if (IsForegroundFullScreen(hShellTrayWnd))
```

`hShellTrayWnd` is the function's HWND parameter and is already in scope.

- [ ] **Step 3: Verify all call sites now pass an HWND**

```bash
rtk grep -n "IsForegroundFullScreen" ExplorerPatcher/dllmain.c
```

Expected output (line numbers may shift slightly):

```
1352:static BOOL IsForegroundFullScreen(HWND hTaskbarWnd)
1625:    if (IsForegroundFullScreen(hTaskListWnd))
1935:    if (IsForegroundFullScreen(hShellTrayWnd))
```

No bare `IsForegroundFullScreen()` (with empty parens) should remain. If one does, the build will fail with `error C2198: 'IsForegroundFullScreen': too few arguments for call`.

---

## Task 3: Build verification (amd64 Release)

**Files:** none modified (build only)

This project has no automated test suite — the build is the only mechanical gate before manual validation.

- [ ] **Step 1: Ensure the zlib dependency has been built once**

If `build/Release/x64/zs.lib` does not already exist (first-time build of this checkout), run from a Visual Studio 2022 Developer Command Prompt:

```cmd
BuildDependenciesRelease.bat
```

If it already exists, skip this step.

- [ ] **Step 2: Build the amd64 Release configuration**

From a VS 2022 Developer Command Prompt, in `q:\src\ExplorerPatcher`:

```cmd
nuget restore ExplorerPatcher.sln
msbuild /m /p:Configuration=Release /p:Platform=amd64 ExplorerPatcher.sln
```

Expected: `Build succeeded.` with `0 Error(s)`. Output DLL at `build/Release/x64/ExplorerPatcher.amd64.dll`.

If the build fails with a compile error inside `dllmain.c` near `IsForegroundFullScreen`, re-read the new function carefully against Task 1 Step 2 — common causes are a typo in a class-name literal, a missed bracket, or `MSTASKLISTWCLASS_BADGE_OVERLAY_CLASS` having been moved elsewhere in the file.

- [ ] **Step 3: Commit**

```bash
rtk git add ExplorerPatcher/dllmain.c
rtk git commit -m "fix: hide window-count badge when a foreground window covers the monitor

Rewrites IsForegroundFullScreen so monitor geometry is the authoritative
signal. Removes the unreliable 'Shell_TrayWnd visible -> not fullscreen'
early-return: the Win11 auto-hidden taskbar still reports IsWindowVisible
== TRUE while slid off-screen, which masked F11 / borderless-fullscreen
cases. Filters shell, desktop, explorer-owned, and our own overlay
windows, and requires the foreground window to be on the same monitor as
the taskbar instance being updated so multi-monitor setups only hide
badges on the affected display.

Spec: docs/superpowers/specs/2026-04-27-fullscreen-badge-hide-design.md"
```

---

## Task 4: Manual validation

**Files:** none modified

The project has no test suite; this checklist is the validation gate per the spec. Deploy the freshly-built DLL by the project's normal procedure (replace `dxgi.dll` in the EP install dir or run via the existing setup), restart explorer, then run each scenario.

- [ ] **Step 1: Browser F11 fullscreen**
  Open Edge or Chrome on a YouTube video, press **F11**.
  Expected: badge disappears within ~300 ms. Press **F11** again → badge reappears within ~300 ms.

- [ ] **Step 2: D3D exclusive fullscreen**
  Launch any DXGI exclusive-fullscreen application (e.g., a game set to "Fullscreen" not "Borderless").
  Expected: badge hidden while in-game; reappears on Alt-Tab to desktop.

- [ ] **Step 3: Borderless-fullscreen game**
  Run a game in borderless-windowed fullscreen on a supported display.
  Expected: badge hidden.

- [ ] **Step 4: Maximized non-fullscreen window**
  Maximize Notepad (or any standard app — taskbar reserves screen space, so the maximized rect is smaller than the monitor rect).
  Expected: badge stays **visible**.

- [ ] **Step 5: Multi-monitor isolation** (skip if only one monitor)
  Put a fullscreen video on monitor 1; leave monitor 2 with normal windowed apps.
  Expected: monitor 1's taskbar badges hidden; monitor 2's taskbar badges still visible.

- [ ] **Step 6: Auto-hide taskbar with no fullscreen app**
  Settings → Personalization → Taskbar → "Automatically hide the taskbar" = on. No fullscreen app running.
  Expected: badges remain visible (the geometry-only check is independent of taskbar visibility).

- [ ] **Step 7: Win10-style taskbar mode**
  In ExplorerPatcher Properties, switch to the Windows 10 taskbar (`bOldTaskbar = 1`). Repeat Step 1.
  Expected: same hide/restore behavior on the `MSTaskListWClass` path.

- [ ] **Step 8: Re-show on exit**
  After every "hidden" case above, exit fullscreen and confirm the badge returns within ~300 ms. (If it doesn't, investigate stale `pData->prevBadgeCount` state in the overlay update — but per the spec the hide-only-when-fullscreen path leaves `prevBadgeCount` untouched, so the next non-fullscreen tick should re-render.)

If all eight steps pass, the fix is complete.

---

## Notes for the implementer

- **Do not refactor anything else.** The bug is one function and two call-site changes. Resist the urge to clean up nearby code, the badge timer cadence, or the existing 300 ms polling.
- **Do not add new settings or registry keys.** Behavior is unconditionally improved — there is no opt-in.
- **Do not add ARM64 / IA-32 builds to the verification step.** amd64 Release is sufficient mechanical proof; the changed code is platform-neutral standard Win32 and will build identically on the other configs. CI handles cross-platform.
- **No header file changes.** `IsForegroundFullScreen` is `static` and used only inside `dllmain.c`. No external declarations exist.
- **`HWND` parameter is allowed to be NULL.** The function falls back to the foreground window's monitor when the caller passes NULL. (Both current call sites always pass a non-NULL HWND, but the guard makes the helper safe to call from anywhere later.)
