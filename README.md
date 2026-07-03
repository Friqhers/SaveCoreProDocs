# SaveCore Pro

**A drop-in, automatic save system for Unreal Engine 5.**

Mark the variables you want to keep with `SaveGame`, then call **Save** or **Load**.
That's it. There's no manager actor to place in your level, no singleton to wire up,
and no per-actor registration to maintain — SaveCore Pro discovers your saveable
actors on its own and serializes the tagged fields for you.

> 🧩 Blueprint **and** C++ &nbsp;·&nbsp; 🎮 Single-player **and** multiplayer &nbsp;·&nbsp; 🧠 Zero setup &nbsp;·&nbsp; ⚡ Async by default

---

## Contents

- [Highlights](#highlights)
- [Quick start](#quick-start)
- [How it works](#how-it-works)
- [Installation](#installation)
- [Saving and loading](#saving-and-loading)
- [Save events](#save-events)
- [Players and levels](#players-and-levels)
- [Runtime-spawned actors](#runtime-spawned-actors)
- [Save slots and menus](#save-slots-and-menus)
- [Thumbnails](#thumbnails)
- [Per-actor save options](#per-actor-save-options)
- [Hooks](#hooks)
- [World Partition and streaming](#world-partition-and-streaming)
- [Multiplayer and authority](#multiplayer-and-authority)
- [Save versioning](#save-versioning)
- [Project settings](#project-settings)
- [Redirectors](#redirectors)
- [Troubleshooting](#troubleshooting)

---

## Highlights

| | Feature |
| :---: | :--- |
| 🔍 | **Automatic actor tracking** — the save manager finds saveable actors as they're placed or spawned. Nothing to register by hand. |
| 🏷️ | **Tag-to-save** — mark any field `UPROPERTY(SaveGame)` and it persists. No custom read/write code. |
| 🗺️ | **Level-aware** — every level keeps its own state, so leaving and returning restores it exactly. |
| 🌍 | **World Partition & streaming** — state for streamed-out areas is preserved and re-applied automatically. Nothing to set up. |
| 💥 | **Destruction that sticks — both ways** — a destroyed crate stays destroyed after loading; load an *older* save where it was alive and it comes back, identity and all. |
| 🏃 | **Momentum preserved** — a crate saved mid-roll keeps rolling; a player saved mid-jump keeps their arc. Automatic. |
| 👥 | **Multiplayer-ready** — per-player data is keyed by a stable ID that survives reconnects and restarts. |
| ⚡ | **Async by default** — saving and loading run off the game thread, with events to tell you when they finish. |
| 🖼️ | **Save-menu ready** — list slots, read their metadata without loading, grab a screenshot thumbnail, and delete — everything a Load screen needs. |
| 🗜️ | **Compressed saves** — Oodle compression on a background thread; loading auto-detects the format. |

---

## Quick start

Three steps to a working save.

### 1. Enable the plugin

**Edit → Plugins → SaveCore Pro**, tick it, and restart the editor if prompted.
(Full options in [Installation](#installation).) The save manager creates itself — there is
nothing to place in a level.

### 2. Make an actor saveable

Implement `ISCPSaveableInterface`, then tag the fields you want to keep:

```cpp
UCLASS()
class AMyChest : public AActor, public ISCPSaveableInterface
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)   // ← this value is saved and restored automatically
    bool bIsOpen = false;
};
```

> 🧩 **In Blueprint:** add **SCPSaveable Interface** under *Class Settings → Interfaces*,
> then tick **SaveGame** in the advanced details of any variable you want to keep.

### 3. Save and load

Drop a single node into any graph:

- **Save Game (Async)** — non-blocking, with **On Completed** / **On Failed** pins.
- **Load Game (Async)** — the same, for loading.

That's a working save system. Everything below is optional polish.

---

## How it works

A quick mental model before you dive in — three ideas cover almost everything.

- **One manager, created for you.** SaveCore Pro lives on a `GameInstanceSubsystem` —
  a single global object Unreal spawns automatically and keeps alive for the whole game.
  You never place or spawn it; you just ask it to save or load.
- **You tag, it serializes.** Any field marked `UPROPERTY(SaveGame)` is written and read
  back through Unreal's reflection system. No manual byte-packing, no versioned structs to
  maintain by hand.
- **Identity is automatic.** Placed actors are matched by their location in the level;
  runtime-spawned actors by their object name; players by a stable online/local ID. This is
  what lets a save reliably reconnect to the right actors after a reload — and you write none
  of it.

Everything else — levels, streaming, multiplayer, thumbnails — builds on those three ideas.

---

## Installation

Install **engine-wide** (available to every project on that engine version) or
**per-project** (travels with the project — best for source control and teams). Either works;
pick one.

**Option A — engine-wide**

1. In the **Epic Games Launcher → Fab Library**, find **SaveCore Pro** and click
   **Install to Engine**.
2. Enable it in **Edit → Plugins → SaveCore Pro** and restart when prompted.
   *(Manual alternative: copy the `SaveCorePro` folder into
   `<UnrealEngine>/Engine/Plugins/Marketplace/`, then enable it.)*

**Option B — per-project**

1. Copy the `SaveCorePro` folder into your project's `Plugins/` directory (create the
   `Plugins/` folder next to your `.uproject` if it doesn't exist yet).
2. Launch the project and let it compile (C++ projects), or enable it under
   **Edit → Plugins** (Blueprint-only projects).

> ✅ **Nothing else to set up.** The manager is a subsystem, so it's created automatically —
> there is nothing to add to a level or configure in Project Settings to get started.

---

## Saving and loading

**Blueprint** — place a node and wire the execution pins:

| Node | Use it for |
| :--- | :--- |
| **Save Game (Async)** | Normal saving. Non-blocking, with *On Completed* / *On Failed* pins. |
| **Load Game (Async)** | Normal loading. Same pins. |
| **Save Game (Sync)** / **Load Game (Sync)** | Blocking — use **only** at startup, before the world ticks. |

**C++** — call the subsystem directly; pass an optional callback for the result:

```cpp
auto* SaveManager = GetGameInstance()->GetSubsystem<USCPSaveManagerSubsystem>();

// Fire-and-forget:
SaveManager->SaveGameAsync(TEXT("Slot_0"));

// With a per-call result callback:
SaveManager->LoadGameAsync(TEXT("Slot_0"), 0, [](bool bSuccess)
{
    UE_LOG(LogTemp, Log, TEXT("Load %s"), bSuccess ? TEXT("OK") : TEXT("FAILED"));
});
```

The player's Pawn transform and any `UPROPERTY(SaveGame)` fields on the **Pawn**,
**PlayerState**, and **PlayerController** are captured automatically — **no interface
required on the Pawn**.

> 💡 **Every call is a single node.** All of SaveCore Pro's Blueprint functions
> (save/load, slots, backups, thumbnails, auto-save, versioning) are helpers with a
> hidden world-context pin, so you just drop them into any actor or widget graph —
> **no "Get Subsystem" or Target wiring**. (In C++ you call the
> `USCPSaveManagerSubsystem` methods directly, as shown throughout.)

---

## Save events

Prefer a single global handler — a HUD "Saving…" indicator, or a save menu that refreshes
itself? Bind these events on the manager. To reach them, drop the **Get Save Core Manager**
node once and bind on it.

| Event | Fires when… | Gives you |
| :--- | :--- | :--- |
| **On Save Started** | a save begins | `Slot Name`, `Is Auto Save` |
| **On Save Completed** | a save finishes | `Success`, `Slot Name`, `Is Auto Save` |
| **On Load Started** | a load begins | `Slot Name` |
| **On Load Completed** | a load finishes | `Success`, `Slot Name` |
| **On Save Deleted** | a slot is removed | `Slot Name` |

A few guarantees worth knowing:

- **Started always pairs with Completed.** Even a load of a missing slot fires
  *Started → Completed(false)*. A call that's rejected because a save/load is already
  running (or you're not the server) fires **neither**.
- **`Is Auto Save`** is `true` only for the periodic auto-save timer — so you can show
  "Auto-saving…" distinctly from a manual save.
- **On Save Deleted** fires once per slot actually removed (by *Delete Save*, or per slot in
  *Delete All Saves*), so an open menu can refresh its list without re-polling.
- All events fire on the game thread, for every save/load path — async, sync, and auto-save.

---

## Players and levels

A player's **state** and their **position** are handled differently, on purpose:

- **State is global.** Inventory, stats — any `SaveGame` field — follows the player between
  levels.
- **Position is per-level.** Return to a level and the player is put back where they left it.
  Enter a level they were never saved in and they start at that level's `PlayerStart`,
  instead of being teleported to the last level's coordinates.

This is automatic — nothing to configure.

---

## Runtime-spawned actors

Actors placed in your level are matched by their level path. Actors **spawned at runtime**
work the same way, with no extra setup and no identity code — SaveCore Pro identifies each by
its object name and re-spawns it under that exact name on load. Just implement the interface
and tag your fields:

```cpp
UCLASS()
class AMyPickup : public AActor, public ISCPSaveableInterface
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)
    int32 Quantity = 1;   // saved & restored automatically
};
```

A save → reload re-creates and re-hydrates these actors for you.

> ℹ️ **Streaming note:** runtime-spawned actors are re-spawned on load only for the
> *persistent* level. Spawn long-lived actors into the persistent level (the default) rather
> than a streaming sublevel. Placed actors inside sublevels and World Partition cells are
> restored normally.

---

## Save slots and menus

Everything a save/load screen needs, all on the manager:

```cpp
auto* SM = GetGameInstance()->GetSubsystem<USCPSaveManagerSubsystem>();

// List slots, newest first.
TArray<FString> Slots = SM->GetAllSaveSlots();

// Full metadata for each, in one call.
for (const FSCPSaveSlotInfo& Info : SM->GetAllSaveSlotInfos())
{
    // Info.SlotName, Info.SavedAt, Info.GameVersion, Info.LevelName, Info.bHasThumbnail
}

// The newest slot — back a "Continue" button.
FSCPSaveSlotInfo Latest;
if (SM->GetMostRecentSaveSlot(0, Latest)) { /* Latest.SlotName */ }

// Peek one slot WITHOUT loading it.
FSCPSaveSlotInfo Info;
if (SM->GetSaveSlotInfo(TEXT("Slot_0"), 0, Info)) { /* … */ }

// "Save As" / duplicate, or rename.
SM->CopySaveSlot(TEXT("Slot_0"), TEXT("Slot_0_Copy"));
SM->RenameSaveSlot(TEXT("Slot_0"), TEXT("MySave"));

// Delete a slot (and its thumbnail / info / backups) …
SM->DeleteSave(TEXT("Slot_0"));
// … or wipe every slot for a "clear all save data" button.
SM->DeleteAllSaves();

// Starting a NEW GAME — forget carried-over in-memory state first (deletes nothing on disk).
SM->ClearSaveState();
```

Every one of these is a **single Blueprint node**: *Get All Save Slots*, *Get All Save Slot
Infos*, *Get Most Recent Save Slot*, *Get Save Slot Info*, *Copy Save Slot*, *Rename Save
Slot*, *Delete Save*, *Delete All Saves*, *Clear Save State (New Game)*.

> 🆕 **New Game vs. Delete Save.** *Delete Save* removes a slot's file from disk.
> *Clear Save State* touches no files — it just resets the **in-memory** session so the new
> game's first save starts clean. Without it, world state from a previously loaded slot would
> be carried forward into the new save. Call it when the player picks "New Game".

---

## Thumbnails

Attach a screenshot to a slot for your save menu. Everything — capture, image encode, and
disk write — runs **off the game thread**, so it never hitches the frame.

**Blueprint** — three latent nodes, each with *On Completed* / *On Failed*:

| Node | What it does |
| :--- | :--- |
| **Save Thumbnail From Viewport (Async)** | One-click capture of what's on screen. No setup. |
| **Save Thumbnail (Async)** | Store a render target you drive yourself (a posed shot). |
| **Load Thumbnail (Async)** | Decode a slot's thumbnail into a `Texture2D` for a `UImage`. |

**C++:**

```cpp
// Capture the screen (downscaled to fit 480px) as the slot's thumbnail.
SM->SaveThumbnailFromViewport(TEXT("Slot_0"), 480, 0, /*bIncludeUI*/ false,
    [](bool bOk){ /* thumbnail written */ });

// In the menu — load it back as a UTexture2D for a UImage, off-thread.
SM->LoadThumbnailAsync(TEXT("Slot_0"), 0,
    [](UTexture2D* Thumb){ if (Thumb) { /* assign to UImage */ } });
```

**`SaveThumbnailFromViewport` has zero frame cost** — no game- or render-thread stall. It
grabs the frame with a non-blocking GPU readback, then resizes, encodes, and writes on a
worker thread. It works identically in PIE and packaged builds, and captures **only the game
viewport** (editor chrome and the PIE title bar are cropped out). `bIncludeUI` chooses whether
the HUD/UMG is included: `false` (default) captures the scene before the UI is drawn, `true`
captures the final composited frame. It's spam-safe too — a second call while one is in flight
fails fast (*On Failed*) rather than stomping the first.

Want a specific view instead (a posed hero shot, or a clean frame behind a loading screen)?
Drive a `SceneCapture2D` into a render target and store that:

```cpp
SM->SaveThumbnail(MyRenderTarget, TEXT("Slot_0"), 0, [](bool bOk){ /* done */ });
```

A few more details:

- **Portable storage.** Thumbnails go through the same platform save system as the save, so
  they travel with the slot and work on **all platforms** (including consoles).
- **Format.** Choose **PNG** (lossless) or **JPEG** (much smaller, with a quality slider) under
  *Project Settings → Plugins → SaveCore Pro → Thumbnail Format*. Loading auto-detects the
  format, so switching it never breaks existing thumbnails.
- **Housekeeping.** Removed by *Delete Save*, carried with backups, per-user (`UserIndex`).
  `HasThumbnail` and `FSCPSaveSlotInfo.bHasThumbnail` tell you whether a slot has one.
- A blocking `LoadThumbnail` also exists for editor tooling / loading screens; prefer the async
  path for in-game UI.

> ⚠️ **Thumbnails need rendering — not available on a dedicated server.** They're produced only
> where there's a GPU (standalone, listen server, or client). This is separate from save
> *authority*, which does include dedicated servers — the server owns the authoritative save;
> thumbnails are a client/standalone save-menu concern.

---

## Per-actor save options

Override `GetSaveFlags()` on a saveable actor to change how it's treated. Every flag defaults
to **off**, so actors that don't override it are saved normally.

| Flag | Effect |
| :--- | :--- |
| `bSkipSave` | Never written, restored, or pruned. For cosmetic actors that implement the interface only for `OnPreSave`/`OnPostLoad`. |
| `bSkipTransform` | Its `SaveGame` fields are saved/restored, but its **transform** is not — it keeps whatever position the level or spawn gives it. |
| `bSkipVelocity` | For a physics actor: restore everything *except* **physics velocity**, so the body starts at rest. Independent of `bSkipTransform`. |
| `bPersistent` | Stored in a global, **level-independent** channel and restored regardless of the current level. Runtime ones are re-spawned into the persistent level. |

**C++:**

```cpp
virtual FSCPSaveActorFlags GetSaveFlags_Implementation() const override
{
    FSCPSaveActorFlags Flags;
    Flags.bSkipTransform = true;   // save my data, but don't move me
    return Flags;
}
```

> 🧩 **In Blueprint:** implement the **Get Save Flags** interface event and return an
> `SCP Save Actor Flags` struct with the boxes you want ticked.

> ℹ️ `bPersistent` is for runtime-spawned actors that must outlive level transitions
> (companions, dropped loot that follows the player). Place persistent actors in the
> persistent level. Player pawns are saved through their player record and ignore these flags.

---

## Hooks

Override these `ISCPSaveableInterface` events to run logic around serialization — for example,
snapshot a procedural mesh before saving and rebuild it after loading:

- **`OnPreSave`** — called just before the object's fields are serialized.
- **`OnPostLoad`** — called just after the object's fields are restored.

---

## World Partition and streaming

Streaming works out of the box — nothing to enable. SaveCore Pro keeps an in-memory, per-level
snapshot of your world:

- When a sublevel or World Partition cell **streams out**, its actor state is captured *before*
  the actors are destroyed — so it's never lost.
- When it **streams back in**, the saved state is re-applied to the fresh actors automatically.
- A **save** writes every visited level, including ones currently streamed out.
- A **load** restores each level as it becomes available.

World Partition actors are identified by their stable, globally-unique IDs, so they match
correctly across sessions and between editor (PIE) and packaged builds.

---

## Multiplayer and authority

All save/load runs on the **server** (or in standalone) only — calls on a client are ignored.
For reconnecting players, call `ApplySinglePlayerData(PS)` from `GameMode::PostLogin`; it
restores that player from the in-memory cache with no disk I/O.

Each player is keyed by a stable identity that survives restarts:

1. **Online subsystem ID** (Steam / EOS / …) — every shipping multiplayer game has one.
2. **Local-player index** — for single-player and local co-op.
3. **Bot** — keyed by its pawn's stable ID.

> ℹ️ A remote human with **no** online subsystem (e.g. a bare LAN session) has no
> restart-stable identity, so they're skipped rather than risk restoring one player's data
> into another. For guaranteed multiplayer persistence, run an online subsystem.

---

## Save versioning

SaveCore Pro keeps its **own** file format backward-compatible automatically — old saves keep
loading after plugin updates, with nothing to do.

Versioning below is for when **your game's** data changes — you rename a field, change an
enum's meaning, restructure an inventory — and an old save would otherwise load wrong values.

### 1. Set a version for your data

**Project Settings → Plugins → SaveCore Pro → Game Save Version.** Start at `1`. Every save is
stamped with this number; bump it each time you make a breaking change to your saved data.

### 2. Migrate on load, where you have the data

After a load, `GetLoadedSaveVersion()` returns the version the *file* was written with, and
`GetCurrentSaveVersion()` your current setting. Compare them and patch the old data — the
natural place is the object's `OnPostLoad` hook, where its fields have just been restored.

Say v1 stored health as a 0–1 percentage in `HealthPercent`, and v2 switched to an absolute
`Health`. Bump **Game Save Version** to `2` and handle the old files:

```cpp
// MyCharacter.h
UPROPERTY(SaveGame) float Health = 100.f;
UPROPERTY(SaveGame) float HealthPercent = 1.f;   // legacy (v1) field, kept for migration

// MyCharacter.cpp
void AMyCharacter::OnPostLoad_Implementation()
{
    auto* SM = GetGameInstance()->GetSubsystem<USCPSaveManagerSubsystem>();

    if (SM->GetLoadedSaveVersion() < 2)   // this file was saved by v1
        Health = HealthPercent * MaxHealth;   // rebuild the new field from the old one
}
```

> 🧩 **In Blueprint:** in your actor's **On Post Load** event, **Branch** on
> `Get Loaded Save Version` `<` `2`, then set `Health` from the legacy field. Both getter nodes
> are pure (no execution pin).

> 💡 Keep old fields around (don't delete them) until you're sure no old saves remain, so the
> migration still has something to read from.

### 3. (Optional) Check a slot before loading

To warn the player or refuse an incompatible save *before* applying it, peek the slot's version
without loading it:

```cpp
int32 FileVersion = 0;
if (SM->GetSaveSlotVersion(TEXT("Slot_0"), 0, FileVersion))
{
    if (FileVersion > SM->GetCurrentSaveVersion())
        return;   // save is from a NEWER build than this one — don't load it
}
```

`GetLoadedSaveVersion()` returns `0` before anything is loaded. All three are Blueprint nodes:
*Get Loaded Save Version*, *Get Current Save Version*, *Get Save Slot Version*.

---

## Project settings

**Project Settings → Plugins → SaveCore Pro** (stored in `DefaultGame.ini`). Quick reference:

| Setting | Default | What it does |
| :--- | :--- | :--- |
| **Compress Save Files (Oodle)** | On | Compresses saves on a background thread. Loading auto-detects compressed vs. raw, so toggling never invalidates existing saves. |
| **Auto Backup Previous Save Data** | Off | Before overwriting a slot, copies it (and its sidecars) to a backup. See below. |
| **Backup Count** | 1 | How many previous saves to keep (max 16). See below. |
| **Enable Auto Save** | Off | Saves on a timer during gameplay. See below. |
| **Auto Save Interval (Seconds)** | 300 | Seconds of unpaused gameplay between auto-saves (min 5). |
| **Auto Save Slot Name** | `AutoSave` | Slot the auto-save writes to (falls back to *Default Slot Name*). |
| **Capture Auto Save Thumbnail** | On | Grabs a viewport thumbnail with each auto-save (no frame hitch). No effect without a viewport. |
| **Thumbnail Format** | PNG | PNG (lossless) or JPEG (smaller). **Thumbnail JPEG Quality** (1–100) applies to JPEG. |
| **Auto-Load Save (Startup & Map Travel)** | Off | Loads the default slot on launch and after each map travel. See below. |
| **Default Slot Name** | `Slot_0` | Slot used by auto-load; also the auto-save fallback. |
| **Game Save Version** | 1 | Your game's save-data version (see [Save versioning](#save-versioning)). |
| **Runtime Actor Redirects** / **Level Redirects** | *(empty)* | Keep old saves loading after you rename assets (see [Redirectors](#redirectors)). |

### Backups

When **Auto Backup Previous Save Data** is on, each save first copies the slot's current file
(and its thumbnail/info) to a backup before overwriting it. The first save of a slot makes
none, and backups never appear in slot enumeration. **Backup Count** rotates them newest-first
and drops the oldest once the limit is reached, so you can roll back more than one save.

```cpp
// Roll back to the immediately previous save.
if (SM->HasBackup(TEXT("Slot_0")))
    SM->RestoreBackup(TEXT("Slot_0"));                       // index 0 = most recent backup

// With Backup Count > 1, offer a list of recent versions:
const int32 N = SM->GetBackupCount(TEXT("Slot_0"));          // e.g. 3
SM->RestoreBackup(TEXT("Slot_0"), 0, /*BackupIndex=*/2);     // the oldest kept
```

### Auto-save

When **Enable Auto Save** is on, the game saves itself on a timer. The countdown only advances
during unpaused play on the server/standalone (never on clients), a save already in progress is
skipped (not queued), and saves use the async pipeline so there's no frame hitch. Auto-saves
**don't** rotate the backup history — that's reserved for your explicit saves. Toggle it at
runtime for menu-driven or multiplayer games:

```cpp
// Gate auto-save to gameplay (e.g. in GameMode BeginPlay or a pause menu):
SM->SetAutoSaveEnabled(true);
SM->SetAutoSaveEnabled(false);

// Drive a "saving soon" HUD indicator:
const float Secs = SM->GetSecondsUntilAutoSave();   // -1 when disabled
```

### Auto-load

When **Auto-Load Save (Startup & Map Travel)** is on, the default slot is loaded automatically
as the game's first world starts *and* again after every map travel — so a single-player game
restores with no Blueprint or C++ wiring. It fires once per world (never per streamed sublevel)
and does nothing if the slot has no save yet. Leave it off for menu-driven ("Continue" button)
or multiplayer games, which control load timing themselves.

<details>
<summary><b>Advanced: multi-threaded saving</b> (for scenes with many saveable actors)</summary>

<br>

These two are **off by default** and most games never need them — reach for them only if you
measure a real save hitch in a scene with a lot of saveable actors.

- **Multi-Threaded Saving** — an async save always writes to disk on a background thread, but by
  default it still *serializes* the world on the game thread, which can hitch in a scene with
  *many* saveable actors. Turn this on to move serialization onto a worker too. Your actor list
  and `OnPreSave` hooks still run on the game thread; only the heavy serialization is offloaded,
  so the frame never blocks (the save just finishes a few frames later).
  **Tradeoff:** because gameplay keeps ticking while the worker serializes, an actor whose state
  changes mid-serialization is captured a moment later — imperceptible for most games. Leave it
  off if you need a hard point-in-time snapshot. Affects `SaveGameAsync` only; `SaveGameSync` is
  always single-threaded.

- **Gather On Worker Thread** — requires *Multi-Threaded Saving*. By default the actor *list* is
  still walked on the game thread (classification, `OnPreSave`, component discovery). Turn this on
  to move that walk onto a worker too — the only thing that removes the remaining cost in scenes
  with *thousands* of saveable actors.
  **Tradeoff:** your `OnPreSave` / `GetSaveFlags` hooks then run on a worker thread, so they must
  be thread-safe — don't spawn/destroy actors, touch timers, or mutate the world from them. Actor
  state is read off the game thread (a value may be a frame stale), and components added to an
  actor *after* it registers may not be captured. Only enable it after confirming a real gather
  hitch and that your hooks are thread-safe.

</details>

---

## Redirectors

When you rename or move things during development, existing saves still reference the old names.
Two optional maps under **Project Settings → Plugins → SaveCore Pro → Redirectors** patch that up
on load. Both are empty by default and cost nothing until you add an entry.

### Runtime Actor Redirects

For a Blueprint that SaveCore Pro spawns at runtime (or resurrects). Map the **old class path**
to the **new class**, and old saves spawn the new one instead. The key accepts any form the
editor gives you — all of these match:

- `/Game/Blueprints/BP_OldEnemy.BP_OldEnemy_C` — the full generated path
- `/Game/Blueprints/BP_OldEnemy.BP_OldEnemy` — the object path without `_C`
- `/Game/Blueprints/BP_OldEnemy` — a bare **Copy Package Path**

It's only consulted when the saved class fails to load directly, so a stale entry is harmless.
(Map-placed actors are matched by level path and don't need this.)

> ℹ️ Renaming a Blueprint in-editor leaves an engine redirector behind, so old saves keep loading
> **without** an entry here. You only need a mapping once that redirector is gone — after
> *Fix Up Redirectors*, or if the asset was deleted/recreated, or a C++ class was renamed.

### Level Redirects

For a renamed or moved map. Per-level world data and per-level player positions are keyed by the
map's package path, so after a rename that data no longer matches. Map the **old level key** to
the **new** one, e.g. `/Game/Maps/OldMap` → `/Game/Maps/NewMap`, and the load re-buckets
everything (world records, player transforms, the slot's displayed level) onto the new map. Use
the package path without any PIE prefix.

---

## Troubleshooting

| Symptom | Likely cause |
| :--- | :--- |
| **A variable isn't saving.** | It's not tagged. Add `UPROPERTY(SaveGame)` (C++), or tick **SaveGame** in the variable's advanced details (Blueprint). |
| **An actor isn't saving at all.** | It doesn't implement `ISCPSaveableInterface`, or it returns `bSkipSave` from *Get Save Flags*. (Player pawns are the exception — they save automatically.) |
| **Nothing saves in multiplayer.** | Save/load only runs on the **server** or standalone. Calls on a client are ignored by design. |
| **A runtime actor doesn't come back after load.** | It was spawned into a streaming sublevel. Runtime actors are re-spawned only for the *persistent* level — spawn it there, or mark it `bPersistent`. |
| **The player teleports to the wrong place across levels.** | That's expected: position is per-level. A level you never saved in leaves the player at its `PlayerStart`. |
| **Old saves stopped loading after I renamed an asset/map.** | Add a [Runtime Actor Redirect or Level Redirect](#redirectors). |
| **A new game inherits the previous game's world.** | Call **Clear Save State (New Game)** before starting the new run. |

---

© Penguru Games. All Rights Reserved.
