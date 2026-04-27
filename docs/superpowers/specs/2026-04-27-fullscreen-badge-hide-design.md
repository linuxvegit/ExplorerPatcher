# Fullscreen Badge Hide — Design

**Date:** 2026-04-27
**Component:** ExplorerPatcher / Win11 window-count badge overlay
**File touched:** `ExplorerPatcher/dllmain.c`
**Status:** Approved

## Problem

The Win11 taskbar window-count badge stays visible when the user enters fullscreen
mode (browser F11, borderless-fullscreen games, video players, exclusive D3D).
The badge should disappear in those scenarios — like Windows' own taskbar auto-hide
on rude windows — and reappear when fullscreen ends.

## Root cause

`IsForegroundFullScreen()` in `ExplorerPatcher/dllmain.c` (lines 1351–1390) has two
defects:

1. It early-returns `FALSE` whenever `Shell_TrayWnd` reports `IsWindowVisible == TRUE`.
   On Win11, the auto-hidden taskbar window normally stays "visible" and is just
   slid off-screen, so this check masks every borderless-fullscreen / F11 case.
2. `SHQueryUserNotificationState() == QUNS_RUNNING_D3D_FULL_SCREEN` only fires for
   true exclusive D3D. Modern games (borderless-windowed) and browser/video F11
   never trigger it.

The two badge-update paths
(`MSTaskListWClass_UpdateBadgeOverlay`, `Win11Taskbar_UpdateBadgeOverlay`) already
call `IsForegroundFullScreen()` on every 300 ms timer tick, so fixing the
detection function alone is sufficient — no plumbing changes are required.

## Goal

Rewrite `IsForegroundFullScreen()` so the foreground-window-covers-monitor check
is the authoritative signal, and reliably hide the badge in all three scenarios
(F11 fullscreen, borderless-fullscreen, exclusive D3D) on both Win10-style and
Win11-style taskbars, including multi-monitor setups.

## Non-goals

- Event-driven detection (e.g., `IAppVisibility`, `SetWinEventHook`). The existing
  300 ms polling is acceptable; a brief one-tick badge flash on
  enter/leave fullscreen is fine.
- Other badge bugs (positioning, count parsing, theme handling).
- Settings / registry changes.
- New PDB symbols, platform branches, or external dependencies.

## Design

### Signature change

```c
// Old
static BOOL IsForegroundFullScreen(void);

// New
static BOOL IsForegroundFullScreen(HWND hTaskbarWnd);
```

The taskbar HWND lets the function decide *per-monitor* whether the foreground
fullscreen window is on the same monitor as the taskbar instance being updated.
This is required for multi-monitor: fullscreen on monitor A must not hide badges
on monitor B.

### Decision order

1. **Fast positive — exclusive D3D:**
   `SHQueryUserNotificationState(&quns) == S_OK && quns == QUNS_RUNNING_D3D_FULL_SCREEN`
   → return `TRUE`.

2. **Resolve target monitor:**
   `hMonTarget = MonitorFromWindow(hTaskbarWnd, MONITOR_DEFAULTTONULL)`.
   If `NULL`, fall back to `MonitorFromWindow(GetForegroundWindow(), MONITOR_DEFAULTTONULL)`.
   If still `NULL` → return `FALSE`.

3. **Get foreground window:** `hFg = GetForegroundWindow()`.
   If `NULL` → return `FALSE`.

4. **Filter shell / own windows.** Return `FALSE` if any of:
   - `hFg == GetDesktopWindow()` or `hFg == GetShellWindow()`
   - `GetClassNameW(hFg, ...)` matches one of:
     `Shell_TrayWnd`, `Shell_SecondaryTrayWnd`, `WorkerW`, `Progman`,
     `EP_TaskbarBadgeOverlay` (our overlay)
   - `GetWindowThreadProcessId(hFg, &pid); pid == GetCurrentProcessId()`
     (anything owned by explorer.exe — e.g., shell popups, jump lists)

5. **Same-monitor requirement:**
   `hMonFg = MonitorFromWindow(hFg, MONITOR_DEFAULTTONULL)`.
   If `hMonFg != hMonTarget` → return `FALSE`.

6. **Geometry check:**
   `MONITORINFO mi; GetMonitorInfoW(hMonTarget, &mi);`
   `GetWindowRect(hFg, &rcWnd);`
   Return `TRUE` iff
   `rcWnd.left   <= mi.rcMonitor.left   &&`
   `rcWnd.top    <= mi.rcMonitor.top    &&`
   `rcWnd.right  >= mi.rcMonitor.right  &&`
   `rcWnd.bottom >= mi.rcMonitor.bottom`.
   Use `<=` / `>=` (not `==`) to tolerate windows that extend slightly past the
   monitor bounds (some games do this by 1–2 px).

The "taskbar visible → not fullscreen" early-return is **removed entirely**.

### Call-site updates

Two callers in the same file:

| Call site | Line (current) | New call |
|---|---|---|
| `MSTaskListWClass_UpdateBadgeOverlay` | 1625 | `IsForegroundFullScreen(hTaskListWnd)` |
| `Win11Taskbar_UpdateBadgeOverlay` | 1935 | `IsForegroundFullScreen(hShellTrayWnd)` |

No other code references `IsForegroundFullScreen`.

## Edge cases

| Case | Result |
|---|---|
| Browser F11 fullscreen video | Hidden ✓ |
| Borderless-fullscreen game | Hidden ✓ |
| True D3D exclusive game | Hidden ✓ (fast path) |
| Maximized window (taskbar reserves space, so `rcWindow != rcMonitor`) | Visible |
| Multi-monitor: fullscreen on monitor A only | Hidden on A's taskbar; visible on B's |
| Auto-hide taskbar enabled, no fullscreen app | Visible (geometry-only check is independent of taskbar visibility) |
| Explorer popup / jump list in foreground | Visible (process-owner filter) |
| Our own overlay window in foreground | Visible (class-name filter) |
| Foreground window 1–2 px larger than monitor | Hidden (`<=` / `>=`) |
| `hTaskbarWnd` destroyed mid-call | Falls back to foreground monitor; still safe |

## Validation (manual — no test suite)

1. **Browser F11:** Open Edge/Chrome on YouTube, press F11 → badge disappears
   within ~300 ms. Press F11 again → reappears.
2. **D3D exclusive:** Launch a DXGI exclusive-fullscreen app → badge hidden.
3. **Borderless game:** Set a Steam title to borderless-fullscreen → badge hidden.
4. **Maximized non-fullscreen:** Maximize Notepad → badge stays visible.
5. **Multi-monitor:** Fullscreen video on monitor 1 → monitor 1's taskbar badges
   hidden; monitor 2's still visible.
6. **Auto-hide taskbar mode (no fullscreen app):** Enable Settings →
   Personalization → Taskbar → Automatically hide → badges remain visible.
7. **Win10 taskbar mode** (`bOldTaskbar = 1`): repeat test 1 against the
   `MSTaskListWClass` path → badge hidden.
8. **Re-show on exit:** After every "hidden" case above, exit fullscreen →
   badge reappears within ~300 ms.

## Risks

- **Latency:** Up to ~300 ms badge flash on enter/leave fullscreen, identical to
  the rest of the overlay-update cadence.
- **`SHQueryUserNotificationState`** is documented and stable since Vista; the
  geometry path doesn't depend on it.
- **No new platform branches** — works identically on amd64 / arm64 / IA-32
  subset builds (the badge code is already gated by `bShowCombinedWindowCount`
  and `bOldTaskbar`).

## Out of scope / explicit follow-ups

None planned. If event-driven detection is wanted later (to remove the
~300 ms flash), `IAppVisibility` would be the natural upgrade — but is not
included in this fix.
