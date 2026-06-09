# Fix kubeconfig syncs not registering until app restart

**Date:** 2026-06-09
**Related issues:** [#551](https://github.com/freelensapp/freelens/issues/551), [#1108](https://github.com/freelensapp/freelens/issues/1108)
**Author:** Yehuda Kahan (with Claude)

## Problem

When a user adds a kubeconfig file or folder via **Preferences → Kubernetes → Kubeconfig Syncs**, the watcher does not start and the clusters never appear in the catalog. Restarting the app (force-quit on macOS, kill via Task Manager on Windows) makes the same entries work because preferences are reloaded fresh on startup.

Confirmed on Windows 10/11 (issue #551, #1108) and macOS (issue #551 comment by `joshsmithers`, 2025-08-07). The root cause is in shared code, not platform-specific.

## Root-cause hypothesis

`KubeconfigSyncManager.startSync()` in `packages/core/src/main/catalog-sources/kubeconfig-sync/manager.ts` does two things:

1. Iterates the existing entries in `kubeconfigSyncs` (an `ObservableMap`) and starts a watcher for each.
2. Sets up `observe(kubeconfigSyncs, ...)` to react to runtime `add`/`delete` mutations.

On startup the entries are already in the map (loaded from preferences), so step 1 starts watchers correctly — this is why a restart "works." When the user adds a new entry from the renderer-side preferences UI, the new entry must propagate from the renderer process to the main process and end up as an `add` mutation on the same `ObservableMap` that step 2 is observing. The leading hypothesis is that the propagation either:

- doesn't deliver an observable `add` change to the main-side map (e.g., the entire map is being replaced rather than mutated), or
- delivers it on a different map instance than the one being observed.

Investigation during implementation will confirm which. A secondary hypothesis (chokidar v3→v4 watch regression) is in scope to verify but is now lower-priority given the cross-platform symptom pattern.

## Goals

1. **Primary:** adding a kubeconfig file or folder via the UI starts watching and surfaces clusters **immediately and automatically**, with no app restart and no extra user action, on Windows / macOS / Linux. This is the must-have outcome.
2. Removing an entry stops the watcher immediately.
3. **Secondary:** provide a manual **Refresh syncs** affordance as a user-visible fallback for edge cases (filesystem hiccups, sleep/wake on laptops, network mounts). This is a nice-to-have on top of goal 1, not a replacement for it.

## Non-goals

- Refactoring the wider kubeconfig-sync architecture.
- Replacing chokidar.
- Cosmetic changes to the preferences UI beyond adding the refresh control.

## Approach

### A. Fix the underlying propagation (primary, must-have)

This is the fix that delivers goal 1 — clicking "Sync file(s)" / "Sync folder(s)" causes the watcher to start automatically and clusters to appear without any further user action.

Identify why runtime mutations to `kubeconfigSyncs` are not triggering the main-process observer, and fix it. Concrete options, ranked by likelihood based on initial code reading:

1. **Replace the `observe()` hookup with a `reaction()` over the keyset.** A reaction that derives `Array.from(kubeconfigSyncs.keys())` and diffs added/removed paths against the previously-seen set is robust to whether the map is mutated in place or replaced wholesale. This is the simplest fix and is correct regardless of which propagation pattern the user-preferences IPC layer ends up using. Trade-off: marginally more work per change (set diff vs. direct delta) — negligible in practice.

2. **Fix the IPC layer to mutate the main-side map in place** instead of replacing it. More invasive; only chosen if option 1 turns out to be insufficient.

The fix lives in `packages/core/src/main/catalog-sources/kubeconfig-sync/manager.ts`. No `process.platform` branching — the bug is cross-platform and so is the fix.

### B. Manual refresh affordance (secondary, fallback)

This is a nice-to-have safety net, **not** the intended way to use kubeconfig syncs day-to-day. Goal 1 is delivered by fix A; this exists for edge cases.

Add a **Refresh syncs** button to the Kubeconfig Syncs preferences panel and bind a global keyboard shortcut (`Ctrl+Shift+R` / `Cmd+Shift+R`, chosen to avoid Chromium's `Ctrl+R` page reload which only reloads the renderer and cannot affect the main-process watcher).

The action sends an IPC message to the main process which calls `KubeconfigSyncManager.stopSync()` then `startSync()`. This is a user-facing fallback — it is independent of fix A and ships even if A fully resolves the bug, because:

- It gives users immediate recourse if the watcher ever drifts for unrelated reasons (filesystem hiccups, sleep/wake on laptops, network mounts).
- It costs almost nothing to implement.

### C. Tests

- Extend `packages/core/src/main/catalog-sources/__test__/kubeconfig-sync.test.ts` with a case: start sync with an empty map, add an entry to the map at runtime, assert that a new watcher is created and that entities appear in the source. This is the test that fails today and passes after fix A.
- Add a test for stop-then-start (covers the refresh flow).

## Architecture sketch

```
┌─ Renderer process ──────────────────────────────────────┐
│ KubeconfigSync UI                                       │
│   syncs: ObservableMap                                  │
│   reaction → state.syncKubeconfigEntries.replace(...)   │
│                                                         │
│ [Refresh syncs] button ──► IPC: refresh-kubeconfig-syncs│
└────────────────┬────────────────────────────────────────┘
                 │ user-preferences IPC
                 ▼
┌─ Main process ──────────────────────────────────────────┐
│ kubeconfigSyncs: ObservableMap (mirror)                 │
│   ▲                                                     │
│   │ reaction over keys()                                │
│   │   added paths   → startNewSync(path)   ◄── FIX A    │
│   │   removed paths → stopOldSync(path)                 │
│                                                         │
│ IPC handler: refresh-kubeconfig-syncs                   │
│   → manager.stopSync(); manager.startSync();   ◄── FIX B│
└─────────────────────────────────────────────────────────┘
```

## Validation plan

The fix has to be validated on a real Windows install matching the user's installation flow (MSI/exe). Two builds will be produced:

1. **Build #1 (baseline)** — built from current `main`, no fix applied. Confirms the dev-built `.exe` reproduces the same bug as the upstream `Freelens-1.9.0-windows-amd64.exe`. Without this step we can't rule out "the build pipeline itself happened to fix it."
2. **Build #2 (with fix)** — same pipeline, fix applied. Confirms the fix actually resolves the bug under real Windows installation conditions.

Both `.exe` builds will be unsigned (test-only); SmartScreen will show "Windows protected your PC → More info → Run anyway" on install. User data persists across uninstalls, so the user's existing sync configuration carries through both tests.

## Risks

- **Cross-build from WSL to Windows.** Electron-builder supports Linux→Windows cross-compilation but may need wine for installer creation. If the build pipeline doesn't cross-build cleanly, we'll fall back to building on a Windows runner (GitHub Actions or local).
- **The propagation bug might not be in `observe()` at all** but somewhere upstream in the user-preferences IPC. If so, fix A's `reaction()` rewrite still helps (more robust hookup), but a deeper fix may be needed. Investigation during Build #1 reproduction will tell us.
- **No Windows dev environment locally.** All Windows-specific validation depends on the user installing the test builds. Mitigation: detailed log instrumentation in dev builds to capture exactly where the chain breaks.

## Out of scope

- Changing chokidar versions.
- Reworking how kubeconfig clusters are discovered or displayed.
- Adding new kubeconfig source types.
