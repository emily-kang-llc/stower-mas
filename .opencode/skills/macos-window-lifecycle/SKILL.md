---
name: macos-window-lifecycle
description: Decision skill for macOS window, scene, and app-lifecycle changes in the Stower repo — use whenever touching the Application Window scene, quit/termination paths, Window-menu commands, or window-close/minimize behavior, or when App Review's single-window requirements are mentioned.
---

# macOS window & app lifecycle in the Stower repo

Use this when you're changing how the Application Window, its Window-menu entry, or the
quit/termination path behaves. It is the judgment layer above the canonical contract —
`Docs/MacAppContract.md` §10 states the behavior; guards `6f`/`6g` in `Scripts/precheck.sh`
assert its shape. Read those first; this skill carries the *decisions behind* them, each dated
to a runtime observation.

## The Application Window is a single-instance `Window(_:id:)` scene, never a `WindowGroup`

Stower is a single-window app. The main scene is
`Window(ApplicationDefinition.applicationWindowTitle, id: ApplicationDefinition.applicationWindowSceneID)`
on `ApplicationDefinition`. A multi-window group scene caused an App Review rejection: closing
the window left a windowless Stower process running with no menu item to bring it back. The
rejection's HIG branch also demands a Window menu even for a one-window app. (Checklist item 2
✅ 2026-08-24.)

## THE trap: a `Window` scene does NOT auto-populate the Window menu

`CommandGroupPlacement` defines `.singleWindowList` as a *placement for commands the app adds*
("describe and reveal any windows that the app defines") — not an auto-populated list. Never
assume a `Window(_:id:)` scene puts its title in the Window menu; it did not (the menu ended at
Bring All to Front). The Window-menu "Stower" entry exists only because
`ApplicationWindowReopenCommand` adds it (`CommandGroup(after: .singleWindowList)`;
`openWindow(id:)` orders the window front). (A1 REFUTED 2026-08-31, reconfirmed 2026-09-10;
item 1 ✅ 2026-09-10.)

## Closing the window quits — via three deliberately redundant mechanisms

All routes converge on the one gate: `applicationShouldTerminate` →
`StowerTerminationDrain.drainPendingWork()` → `StowerBoardViewModel.drainPendingWork()`. No
route may bypass it; a new quit path that exits mid-write is a bug.

1. the single-instance `Window` scene itself (AppKit quits when its only window closes);
2. `ApplicationLifecycleDelegate.applicationShouldTerminateAfterLastWindowClosed → true` — the
   policy stated in source via `StowerAppLifecycle` (unit-tested per I-QuitPolicyTested), not
   left to scene-type implicit semantics;
3. the `NSWindow.willCloseNotification` observer + `NSApp.terminate(nil)` — the only route that
   covers the Settings-open case.

## Settings is an `NSWindow`, so it counts for last-window-close

Closing the board while Settings (⌘,) is open is NOT a last-window close, so mechanism #2 stays
silent — that is exactly why the close observer (#3) exists (JC3). Closing Settings alone never
quits; the observer keys on `stower.window.main` only (item 4b ✅ 2026-08-31). Subscribe to all
window closes, then filter by identifier — the 2026-08-31 run recorded a menu window closing as a
nil-identifier `willCloseNotification`, proving non-application windows post the same
notification. The identifier filter is load-bearing: without it, any other window's close could
terminate the app.

## The termination step is a drain, not a save

Drafts are written to SQLite per keystroke (`setDraft` → `enqueueDraftWrite` →
`draftStore.upsert`); `drainPendingWork()` writes nothing — it only awaits writes already in
flight. And: a `weak` capture anywhere on that path silently hollows the drain once the window
closes first — the bug this branch found and fixed (I-DrainOutlivesWindow; `6g` holds the
`registerDrain` capture on `StowerApplicationWindowContentView` strong).

## Window identity — the identifier match is the answer

SwiftUI sets `NSWindow.identifier` from the `Window(id:)` scene id (`stower.window.main`) —
verified at runtime 2026-08-24. The `NSViewRepresentable` fallback is NOT needed; don't re-derive it.

## Minimize never quits — and the restore-path map

A minimized window still exists, so it is never the rejected state. The Dock icon and Window ›
Stower both restore even with "Minimize windows into application icon" ON (A8/A9 verified
2026-09-10). But ⌘Tab activates the app while restoring NO window — a known SwiftUI platform gap,
accepted wart; Window › Stower is the way back. The close observer subscribes ONLY to
`willCloseNotification`, never `willMiniaturizeNotification` (I-MinimizeNeverQuits).

## The verification split: source grep vs. behavioral checklist

`6f`/`6g` in `Scripts/precheck.sh` grep the SOURCE; a grep can never see a menu. The manual
checklist in `Docs/MacAppContract.md` §10 proves the BEHAVIOR (re-run after any change to these
paths). Keep §10 canonical — this skill is the judgment layer, not a second source of truth; if
behavior and doc disagree, the observation wins and §10 must be corrected.
