# In-Game Mod Manager — Design & Feasibility

> Status: **proposal / design note** (not implemented). Investigation done
> 2026-06-16 against `libultraship` pinned SHA `be6415584d`.

## TL;DR

An in-game **Mod Manager** menu — list installed mods, show their metadata,
enable/disable each one, and reload — is **feasible entirely host-side** (in
`port/`) with **no libultraship changes required for the MVP**. Every primitive
it needs already exists in the engine. The only piece that would benefit from an
upstream change is *surgical* per-mod reload (reloading one mod without touching
its siblings); the MVP reloads all mods at once, which is acceptable.

This is the natural next step in the modding push already underway: the
[`docs/modding.md`](modding.md) guide and the `workspace/playertint` example
cover the **author** side; a Mod Manager covers the **end-user** side. Together
they make a complete modding story.

## Motivation

BattleShip already has a working mod *loader* (runtime TCC compilation,
`MountModsDir`, hot reload). What it does **not** have is a way to *manage* mods
from inside the game. Today the entire C-mod UI is a single button:

- `PortMenu::AddMenuMods` (`port/gui/PortMenu.cpp:1404`) draws one **Hot Reload**
  button plus a line of explanatory text. That's it.
- There is no list of installed mods, no per-mod enable/disable, and no surfaced
  metadata (name, version, author, description).

There is also a **naming overlap worth cleaning up**: the "Mods" sidebar section
historically also hosts the **Hi-Res texture pack** controls
(`PortMenu.cpp:1287`), which are a different feature from the C-mod system. A
Mod Manager is a chance to disambiguate the two.

## What exists today (grounded in code)

| Piece | Location | What it does |
|-------|----------|--------------|
| `DoHotReload()` | `PortMenu.cpp:1370` | `UnloadAll` → `MountModsDir` → `CompileAll` → `LoadAll`. Bulk reload of every mod. |
| `AddMenuMods()` | `PortMenu.cpp:1404` | The current Mods panel: one Hot Reload button. |
| `MountModsDir()` | `port.cpp:445` | Walks `mods/`, mounts folders containing `manifest.json` and `.o2r`/`.otr`/`.zip` files via `ArchiveManager::AddArchive`. Idempotent (skips already-mounted paths). |
| `ScriptLoader` | libultraship `ship/scripting/ScriptLoader.h` | Lifecycle + enumeration (see below). |
| `ArchiveManager` | libultraship `ship/resource/archive/ArchiveManager.h` | Mount/unmount + file access (see below). |
| CVar system | used throughout `PortMenu.cpp` | Persistence for any toggle/setting. |

### `ScriptLoader` public API (libultraship @ `be6415584d`)

- `CompileAll(...)`, `LoadAll(preInit, postInit)`, `UnloadAll(preExit, postExit)` — **bulk** lifecycle.
- `GetLoadersInDependencyOrder() -> std::vector<std::string>` — **enumeration of loaded mod names**, dependency-ordered. ← the key enabler.
- `GetFunction(name, function)` — exported-symbol lookup.
- `SetSafeLevel(SafeLevel)` — security policy (`DISABLE_SCRIPTS` … `ALLOW_ALL_SCRIPTS`).
- There is **no** `Unload(name)` / `Load(name)` — lifecycle is all-or-nothing.

### `ArchiveManager` relevant API

- `AddArchive(path)` / **`RemoveArchive(path) -> size_t`** — mount and **unmount**. ← lets disable be live, not restart-only.
- `GetArchives()`, `LoadFile(path)`, `ListFiles(mask)` — read manifests/assets from mounted mods.

### Manifest fields available per mod

From `manifest.json` (see `workspace/template/manifest.json`): `name`,
`version`, `author`, `description`, `website`, `license`, `code-version`, and an
optional `preview.png` / `.jpg` shown in the UI.

## Feasibility verdict, per capability

| Capability | Possible today? | How |
|------------|-----------------|-----|
| List installed + loaded mods | **Yes** | `GetLoadersInDependencyOrder()` for loaded; walk `mods/` for installed-but-disabled. |
| Show metadata (name/version/author/description) | **Yes** | `ArchiveManager::LoadFile("manifest.json")` per mod, parse JSON. |
| Show preview thumbnail | **Yes** (moderate) | Decode `preview.png` to an ImGui texture; reuse the PNG path `HiResPack` already uses. |
| Enable / disable a mod (persisted) | **Yes** | Disabled-set CVar + skip-on-mount in `MountModsDir` + `RemoveArchive`/`AddArchive` + reload. |
| Reload all mods | **Exists** | `DoHotReload()`. |
| Surgical per-mod reload | **No** | Bulk-only API. Stretch goal — needs `ScriptLoader::Unload(name)`/`Load(name)` upstream. |
| Set security level | **Yes** | `SetSafeLevel()` already exists. |

## Proposed design

### Components

1. **`ModRegistry`** (new, small, host-side — `port/mods/ModRegistry.{h,cpp}`).
   Walks `mods/` (reusing the `MountModsDir` traversal), parses each
   `manifest.json` into a `ModInfo { name, version, author, description,
   previewPath, archivePath, enabled, loaded }`. Cross-references
   `GetLoadersInDependencyOrder()` to mark which are currently `loaded`.

2. **Disabled-set persistence.** A CVar (e.g. `gMods.Disabled`, a
   comma-separated list of mod names, or per-mod `gMods.<name>.Enabled`). The
   single source of truth for which mods are turned off.

3. **`MountModsDir` honors the disabled set.** In the folder walk, read the
   manifest `name` *before* `try_mount`; if it's in the disabled set, skip it.
   Small, surgical change to `port.cpp`.

4. **`AddMenuMods` rewrite.** Replace the single button with a scrollable list of
   **mod cards**: name + version + author, expandable/tooltip description, an
   **enable checkbox**, a loaded/disabled badge, optional preview thumbnail.
   Keep the existing **Hot Reload** and **Open Mods Folder** controls.

### Toggle flow (live, no restart)

- **Enable** a disabled mod: remove from disabled-set CVar →
  `AddArchive(path)` → `DoHotReload()`.
- **Disable** a loaded mod: add to disabled-set CVar → `RemoveArchive(path)` →
  `DoHotReload()`. `DoHotReload` already runs `UnloadAll` first (fires `ModExit`
  and `UninstallHooksForOwner`), so the disabled mod tears down cleanly before
  recompile.

## Wireframe / mockup

The manager lives in the existing **Mods** sidebar section (the same
`PortMenu` window as Settings/Graphics/About). The single Hot Reload button
becomes an action bar plus a scrollable list of **mod cards**.

### Main panel

```
┌─ BattleShip ──────────────────────────────────────────────────────────────┐
│ ┌───────────┐  Mods                                      Safety: [Allow ▾] │
│ │ Settings  │  ───────────────────────────────────────────────────────────│
│ │ Graphics  │  [ ⟳ Hot Reload ]   [ 📁 Open Mods Folder ]   Filter: [____] │
│ │ Audio     │                                                              │
│ │ ▸ Mods    │  ┌──────────────────────────────────────────────────────┐   │
│ │ About     │  │ ┌────┐  Player Tint                 v1.0.0   ● LOADED │   │
│ └───────────┘  │ │ img│  by ssb64-port                      [x] Enabled│   │
│                │ └────┘  Gives each player a distinct fighter color.   │   │
│                │         ↳ FighterEnvColorQueryEvent · MIT             │   │
│                │  └──────────────────────────────────────────────────────┘ │
│                │  ┌──────────────────────────────────────────────────────┐ │
│                │  │ ┌────┐  HookTest                    v1.0.0   ● LOADED │ │
│                │  │ │ img│  by ssb64-port                      [x] Enabled│ │
│                │  │ └────┘  Exercises the funchook detour path.         │ │
│                │  └──────────────────────────────────────────────────────┘ │
│                │  ┌──────────────────────────────────────────────────────┐ │
│                │  │ ┌────┐  Big Head Mode               v0.3.1   ○ off    │ │
│                │  │ │ img│  by someone                         [ ] Enabled│ │
│                │  │ └────┘  Scales every fighter's head.   (disabled)    │ │
│                │  └──────────────────────────────────────────────────────┘ │
│                │  ┌──────────────────────────────────────────────────────┐ │
│                │  │ ┌────┐  Broken Mod              v0.1.0   ⚠ COMPILE ERR │ │
│                │  │ │ img│  by nobody                         [x] Enabled │ │
│                │  │ └────┘  dist/broken.c:42: undefined symbol 'foo'  [▾] │ │
│                │  └──────────────────────────────────────────────────────┘ │
│                │                                                            │
│                │  4 installed · 2 loaded · 1 disabled · 1 error             │
│                └────────────────────────────────────────────────────────── │
└────────────────────────────────────────────────────────────────────────────┘
```

### What each element maps to

| UI element | Backed by | Phase |
|------------|-----------|-------|
| Card list | `ModRegistry` (walk `mods/` + parse manifests) | 1 |
| `● LOADED` / `○ off` badge | name ∈ `GetLoadersInDependencyOrder()` | 1 |
| Name / version / author / description | `manifest.json` fields | 1 |
| `img` thumbnail | `preview.png` decoded to an ImGui texture | 3 |
| `[x] Enabled` checkbox | disabled-set CVar + `Add`/`RemoveArchive` + `DoHotReload` | 2 |
| `⚠ COMPILE ERR` + `[▾]` expander | last libtcc error (today only in `ssb64.log`) | 3 |
| `Safety: [Allow ▾]` | `ScriptLoader::SetSafeLevel()` | 3 |
| `⟳ Hot Reload` / `📁 Open Mods Folder` | existing `DoHotReload` / `SDL_OpenURL` | exists |
| `Filter` box | client-side name filter over the card list | 1 |

### States a card can be in

- **Loaded** — enabled and running (`●`).
- **Disabled** — installed but skipped on mount (`○`); checkbox unticked.
- **Compile error** — enabled but libtcc rejected it (`⚠`); the expander shows
  the first error line. This is the highest-value Phase 3 add: today a typo in a
  mod fails silently to `ssb64.log` and the mod just doesn't appear.
- **Dependency-blocked** (future) — greyed checkbox with a tooltip naming the
  missing/disabled dependency, derived from the manifest `dependencies` field.

> **Note on layout fidelity:** this is a conceptual sketch, not pixel spec. The
> real panel is built from `PortMenu`'s widget framework (`AddWidget` /
> `WIDGET_CUSTOM` + raw ImGui inside the card loop), so exact spacing and the
> card chrome will follow whatever the surrounding menu already uses.

## Phased implementation plan

- **Phase 1 — MVP, read-only (lowest risk, purely additive).** `ModRegistry` +
  a mod list showing metadata and a loaded badge. Keep Hot Reload. No
  enable/disable yet. Nothing behaves differently; you just *see* your mods.
- **Phase 2 — enable/disable.** Disabled-set CVar + `MountModsDir` skip +
  `RemoveArchive`/`AddArchive` on toggle + reload.
- **Phase 3 — polish.** Preview thumbnails, `SafeLevel` control, dependency
  display, and surfacing compile errors (today they only land in `ssb64.log`).
- **Phase 4 — stretch (needs upstream).** Add `ScriptLoader::Unload(name)` /
  `Load(name)` / `Compile(archive)`-by-name to the libultraship fork for
  surgical per-mod reload.

## Files touched

- `port/mods/ModRegistry.{h,cpp}` — **new**.
- `port/port.cpp` — `MountModsDir` honors the disabled set.
- `port/gui/PortMenu.cpp` — `AddMenuMods` rewrite; reuse `DoHotReload`.
- CVar keys only — no new persistence infrastructure.
- `libultraship` — **only** in Phase 4.

All changes are host-side (`port/`, optionally `libultraship/`). **No
`decomp/src` changes, no game-behavior change** — this is purely additive engine
tooling.

## Open questions / risks

1. **Manifest read for folder-mods vs `.o2r`.** Confirm `ArchiveManager::LoadFile`
   resolves `manifest.json` through the VFS for *both* mounted folders and
   archives. `MountModsDir` mounts folders too, so it should — verify in Phase 1.
2. **`RemoveArchive` VFS eviction.** It returns a count; confirm it actually
   evicts the mod's files from the VFS so a disabled mod's sources are not
   re-compiled on the next `CompileAll`. **This is the main risk.** If eviction
   is incomplete, fall back to *enable is live, disable takes effect next
   launch* and say so in the UI.
3. **Preview decoding.** Need PNG → ImGui texture. `HiResPack` already loads
   PNGs; reuse that path rather than adding a decoder.
4. **Dependency safety.** Disabling a mod others depend on should warn — the
   manifest `dependencies` field plus `GetLoadersInDependencyOrder()` give
   enough to detect it.
5. **Security surface.** Mods are **native C** with full process access. A
   manager that makes loading arbitrary mods a one-click affair should at least
   surface `SafeLevel`; worth a visible note in the UI.

## Why this is the right next feature

It compounds with work already in flight. With the author guide ([#234](https://github.com/JRickey/BattleShip/pull/234)),
the `playertint` example, and a Mod Manager, BattleShip would have an end-to-end
modding story: **how to write a mod → a working mod to copy → a UI to install,
toggle, and reload them**. And it's feasible without waiting on anyone else's
repo for the MVP.
