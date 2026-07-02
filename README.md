# SaveCore Pro

A smart, automatic save system for Unreal Engine. Drop it in, tag the
variables you want to persist, and call Save / Load. No singletons to place,
no manager actor to spawn, no per-actor registration to maintain.

- **Automatic actor tracking** — a `GameInstanceSubsystem` discovers saveable
  actors as they are placed or spawned. Nothing to register by hand.
- **Reflection-based serialization** — tag any field with `UPROPERTY(SaveGame)`
  and it persists. No custom read/write code.
- **Level-aware** — each level keeps its own state, so leaving and returning to
  a level restores it.
- **World Partition & Level Streaming** — state for areas that are streamed out
  is preserved, written to disk, and re-applied automatically as cells and
  sublevels stream back in. Nothing extra to set up.
- **Destruction persists — both ways** — destroy a placed actor (a smashed
  crate, a picked-up item) and it stays destroyed after loading. Load an
  *older* save where it was still alive and it is resurrected on the spot,
  with its identity, transform, and saved fields intact.
- **Momentum is preserved** — a physics actor's linear and angular velocity, and
  a player/pawn's movement velocity, are captured and restored automatically. A
  crate saved mid-roll keeps rolling and a player saved mid-jump keeps their arc,
  instead of dropping to rest on load. Nothing to set up.
- **Multiplayer-ready** — per-player data is keyed by a stable platform/local ID
  that survives reconnects and restarts. GameState and GameMode `SaveGame`
  fields are captured automatically too.
- **Async by default** — saving and loading run off the game thread and report
  completion through Blueprint-assignable delegates.
- **Save-menu ready** — list every slot newest-first, read a slot's metadata
  (time, level, version, thumbnail) without loading it, capture a screenshot
  thumbnail, and delete slots — everything a Save / Load screen needs.
- **Per-actor control** — opt an actor out of saving, out of transform restore,
  or mark it as level-independent, straight from the saveable interface.
- **Optional auto-load** — flip one setting and a single-player game restores
  its slot automatically on launch *and* on every map travel, with no Blueprint
  or C++ wiring. Each map comes back as you saved it; maps you never saved start
  fresh, with the pawn left at its PlayerStart.
- **Compressed saves** — files are Oodle-compressed on a background thread
  (toggle in Project Settings). Loading auto-detects the format, so old saves
  keep working when you flip the setting either way.

## Installation

Install SaveCore Pro **engine-wide** (available to every project on that engine
version) or **per-project** (the plugin travels with the project — best for
version control, team projects, and source builds). Either works; pick one.

### 1. Install in Engine

1. In the **Epic Games Launcher**, open your **Fab Library**, find **SaveCore
   Pro**, and click **Install to Engine**.
2. Choose your engine version and confirm — the launcher downloads the plugin into
   `<UnrealEngine>/Engine/Plugins/Marketplace/`. Then enable it in your project
   under **Edit → Plugins → SaveCore Pro** and restart the editor when prompted.

   *Manual alternative:* copy the `SaveCorePro` folder into
   `<UnrealEngine>/Engine/Plugins/Marketplace/` yourself, then enable it as above.

### 2. Install in Project

1. Copy the `SaveCorePro` folder into your project's `Plugins/` directory (create
   the `Plugins/` folder next to your `.uproject` if it doesn't exist yet).
2. Launch the project and let it compile (C++ projects) or enable the plugin from
   **Edit → Plugins** (Blueprint projects).

That's it. The save manager is a `GameInstanceSubsystem`, so it is created
automatically — there is nothing to place in a level or set in project settings.

## Quick start

### 1. Mark an actor as saveable

Implement `ISCPSaveableInterface` on any actor (or actor component) you want
persisted, then tag the fields to save:

```cpp
UCLASS()
class AMyChest : public AActor, public ISCPSaveableInterface
{
    GENERATED_BODY()

    // This value is saved and restored automatically.
    UPROPERTY(SaveGame)
    bool bIsOpen = false;
};
```

In Blueprint: add `SCPSaveable Interface` under **Class Settings → Interfaces**,
then tick **SaveGame** in the advanced details of any variable you want to keep.

### 2. Save and load

**Blueprint** — place a node and wire the execution pins:

- `Save Game (Async)` — non-blocking node with **On Completed** / **On Failed**
  execution outputs. Just drag off Self.
- `Load Game (Async)` — same, for loading.
- `Save Game (Sync)` / `Load Game (Sync)` — blocking; use only at startup.

The async nodes also broadcast the subsystem-wide `On Save Completed` /
`On Load Completed` events if you prefer a single global handler. To bind those
events, drop the `Get Save Core Manager` node (also single-node) and bind on it.

> **Blueprint surface — every call is a single node.** All of SaveCore Pro's
> Blueprint functions (save/load, slot management, backups, slot info, thumbnails,
> auto-save, versioning) are static helpers with a hidden world-context pin, so you
> just drop the node into any actor/widget graph — **no "Get Subsystem" or Target
> wiring**, exactly like the async nodes. (In C++ you still call the
> `USCPSaveManagerSubsystem` methods directly, as shown below.)

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

The player's Pawn transform and any `UPROPERTY(SaveGame)` fields on the Pawn,
PlayerState, and PlayerController are captured automatically — **no interface
required on the Pawn.**

### Players across levels

A player's **state** (inventory, stats — any `SaveGame` field) is global and
follows them between levels. Their **position** is saved per level: returning to
a level puts them back where they left it, and entering a level they were never
saved in leaves them at that level's `PlayerStart` instead of teleporting them to
the previous level's coordinates. This is automatic — nothing to configure.

## Runtime-spawned actors

Actors placed in the level are matched by their level path automatically.
Actors spawned at runtime work the same way with **no extra setup and no identity
code** — SaveCore Pro identifies each one by its object name and re-spawns it under
that exact name on load, so a save → reload re-creates and re-hydrates them
automatically. Just implement `ISCPSaveableInterface` and tag the fields you want
persisted with `UPROPERTY(SaveGame)`:

```cpp
UCLASS()
class AMyPickup : public AActor, public ISCPSaveableInterface
{
    GENERATED_BODY()

    UPROPERTY(SaveGame)
    int32 Quantity = 1;   // saved & restored automatically
};
```

SaveCore Pro re-spawns and re-hydrates these actors on load.

> **Streaming note:** runtime-spawned actors are re-spawned on load only for the
> *persistent* level. Spawn long-lived persistent actors into the persistent
> level (the default) rather than into a streaming sublevel. Placed actors inside
> streaming sublevels and World Partition cells are restored normally.

## World Partition & Level Streaming

Streaming works out of the box — there is nothing to enable. SaveCore Pro keeps
an in-memory, per-level snapshot of your world:

- When a sublevel or World Partition cell **streams out**, its actor state is
  captured before the actors are destroyed, so it is never lost.
- When it **streams back in**, the saved state is re-applied to the fresh actors
  automatically.
- A **save** writes every visited level — including ones currently streamed out.
- A **load** restores each level as it becomes available.

World Partition actors are identified by their stable, globally-unique actor IDs,
so they match correctly across sessions and between editor (PIE) and packaged
builds.

## Optional hooks

Override these `ISCPSaveableInterface` events to run logic around
serialization (e.g. snapshot a procedural mesh before save, rebuild it after
load):

- `OnPreSave` — just before the object's fields are serialized.
- `OnPostLoad` — just after the object's fields are restored.

## Per-actor save options

Override `GetSaveFlags()` on a saveable actor to change how it is treated. All
flags default to off, so actors that don't override it are saved normally — and
existing actors need no change.

| Flag | Effect |
| --- | --- |
| `bSkipSave` | The actor is never written, restored, or pruned. Use for purely cosmetic actors that implement the interface only for `OnPreSave`/`OnPostLoad`. |
| `bSkipTransform` | The actor's `SaveGame` fields are saved/restored, but its **transform** is not — it keeps whatever position the level or spawn gives it. |
| `bSkipVelocity` | For a physics-simulating actor: save/restore everything *except* its **physics velocity**, so the restored body starts at rest instead of resuming its saved motion. Independent of `bSkipTransform` — position and velocity are separate. |
| `bPersistent` | The actor is stored in a global, **level-independent** channel and restored regardless of which level is current. Runtime persistent actors are re-spawned into the persistent level on load. |

**C++**

```cpp
virtual FSCPSaveActorFlags GetSaveFlags_Implementation() const override
{
    FSCPSaveActorFlags Flags;
    Flags.bSkipTransform = true;   // save my data, but don't move me
    return Flags;
}
```

**Blueprint** — implement the **Get Save Flags** interface event and return a
`SCP Save Actor Flags` struct with the boxes you want ticked.

> `bPersistent` is intended for runtime-spawned actors that must outlive level
> transitions (companions, dropped loot that follows the player). Place
> persistent actors in the persistent level. Player pawns are saved through their
> player record and ignore these flags.

## Managing save slots

Everything a save/load menu needs, all on the subsystem:

```cpp
auto* SM = GetGameInstance()->GetSubsystem<USCPSaveManagerSubsystem>();

// List slots, newest first.
TArray<FString> Slots = SM->GetAllSaveSlots();

// Or get full metadata for each in one call.
for (const FSCPSaveSlotInfo& Info : SM->GetAllSaveSlotInfos())
{
    // Info.SlotName, Info.SavedAt, Info.GameVersion, Info.LevelName, Info.bHasThumbnail
}

// Peek a single slot WITHOUT loading it.
FSCPSaveSlotInfo Info;
if (SM->GetSaveSlotInfo(TEXT("Slot_0"), 0, Info)) { /* … */ }

// Delete a slot (and its thumbnail).
SM->DeleteSave(TEXT("Slot_0"));
```

All of these are exposed to Blueprint as **single-node** helpers (`Get All Save
Slots`, `Get All Save Slot Infos`, `Get Save Slot Info`, `Delete Save`) — just drop
the node into any graph, no "Get Subsystem" wiring needed (see *Blueprint surface*
below).

## Thumbnails

Attach a screenshot to a slot for your save menu. Everything is **asynchronous** —
the screenshot capture, the image encode, and the disk I/O all stay off the game
thread, so saving or loading a thumbnail never hitches the frame.

**Blueprint** — three latent nodes, each with `On Completed` / `On Failed` (the
load node hands back the decoded `Texture2D`):

- **Save Thumbnail From Viewport (Async)** — the one-click option, no setup.
- **Save Thumbnail (Async)** — store a render target you drive yourself.
- **Load Thumbnail (Async)** — decode a slot's thumbnail for a `UImage`.

**C++** — the subsystem methods take an optional completion callback:

```cpp
// Capture the screen (downscaled to fit 480px) as the slot's thumbnail.
SM->SaveThumbnailFromViewport(TEXT("Slot_0"), 480, 0, /*bIncludeUI*/ false,
    [](bool bOk){ /* thumbnail written */ });

// In the menu — load it back as a UTexture2D for a UImage, off-thread.
SM->LoadThumbnailAsync(TEXT("Slot_0"), 0,
    [](UTexture2D* Thumb){ if (Thumb) { /* assign to UImage */ } });
```

`SaveThumbnailFromViewport` has **zero frame cost** — it neither stalls the game thread
nor the render thread. It grabs the frame with a non-blocking GPU readback (no blocking
backbuffer read, which would also return black on a flip-model swapchain), then resizes,
encodes and writes entirely on a worker thread. It works the same in PIE and in a
packaged/standalone build, and captures **only the game viewport** — editor chrome and the
PIE window's title bar are cropped out. `bIncludeUI` controls the HUD/UI: `false` (the
default) captures the scene *before* the UI is drawn, `true` captures the final frame with
HUD/UMG included. It is also spam-safe: if a capture is already in flight, a second call
fails fast (`On Failed`) instead of stomping the first.

If you'd rather capture a specific view (a posed hero shot, an offscreen camera,
or a clean frame while a pause/loading screen is covering the real viewport),
drive a `SceneCapture2D` into a render target and store that instead:

```cpp
SM->SaveThumbnail(MyRenderTarget, TEXT("Slot_0"), 0, [](bool bOk){ /* done */ });
```

A blocking `LoadThumbnail` (returns the `UTexture2D` directly) also exists for
editor tooling / loading screens where a stall is acceptable — prefer the async
path for in-game UI.

Thumbnails are stored through the same platform save system as the save (so they
travel with the slot and work on **all platforms**, including consoles where saves
live in the platform's per-user storage). Choose the encoding under **Project
Settings → Plugins → SaveCore Pro → Thumbnail Format** — PNG (lossless) or JPEG
(much smaller, with a JPEG quality slider) — which controls the stored size.
Loading auto-detects the format, so switching it never breaks existing thumbnails;
the next save re-encodes the slot's thumbnail. They are removed by `DeleteSave`,
travel with backups, and are per-user (`UserIndex`). `HasThumbnail` and the
`bHasThumbnail` field on `FSCPSaveSlotInfo` tell you whether a slot has one.

> **Thumbnails need rendering — they are not available on a dedicated server.**
> Both methods produce an image only where there is a GPU/RHI: standalone,
> listen server, or a client. `SaveThumbnailFromViewport` needs a game viewport
> (a headless dedicated server has none and returns `false`); `SaveThumbnail`
> needs the scene capture to actually render into the target. This is independent
> of save *authority*, which does include dedicated servers — the server owns the
> authoritative save; thumbnails are a client/standalone save-menu concern.

## Plugin settings

**Project Settings → Plugins → SaveCore Pro** (stored in `DefaultGame.ini`):

- **Compress Save Files (Oodle)** — on by default. Compression runs on a
  background thread; loading auto-detects compressed vs. raw files, so
  changing this setting never invalidates existing saves.
- **Auto Backup Previous Save Data** — off by default. When on, each save first
  copies the slot's current file (and its thumbnail / info sidecars) to a backup
  before overwriting it, so a previous version can be recovered. The first save of
  a slot makes none. Backups never show up in slot enumeration.
- **Backup Count** — how many previous saves to keep (default 1, max 16; only
  used when *Auto Backup Previous Save Data* is on). Each save rotates the backups
  newest-first and drops the oldest once this many exist, so you can roll back more
  than one save. Recover a backup with `RestoreBackup` — pass a backup index to pick
  which one (0 = most recent, the default). `HasBackup`/`GetBackupCount` tell you
  what's available (all single-node in Blueprint via the function library):

  ```cpp
  // Roll back to the immediately previous save.
  if (SM->HasBackup(TEXT("Slot_0")))
      SM->RestoreBackup(TEXT("Slot_0"));        // index 0 (most recent backup)

  // Or, with Backup Count > 1, offer the player a list of recent versions:
  const int32 N = SM->GetBackupCount(TEXT("Slot_0"));  // e.g. 3
  SM->RestoreBackup(TEXT("Slot_0"), 0, /*BackupIndex=*/2);  // the oldest kept
  ```
- **Multi-Threaded Saving** — off by default (Advanced). An async save (`SaveGameAsync`)
  always does its disk write on a background thread, but by default it still serializes
  the world on the game thread, which can cause a brief hitch in a scene with *many*
  saveable actors. Turn this on to move the serialization onto a worker thread too: the
  actor list and your `OnPreSave` hooks still run on the game thread, only the heavy
  serialization work is offloaded, so the frame never blocks (the save just finishes a
  few frames later). Tradeoff (the same one EasyMultiSave makes): because gameplay keeps
  ticking while the worker serializes, an actor whose state changes mid-serialization is
  captured a moment later — imperceptible for most games. Leave it off if you need a hard
  point-in-time snapshot, or if you have few saveable actors (there's no hitch to fix).
  Only affects `SaveGameAsync`; `SaveGameSync` is always single-threaded.
- **Gather On Worker Thread (Advanced)** — off by default; requires Multi-Threaded Saving.
  By default the actor *list* is still walked on the game thread (classification, your
  `OnPreSave` hooks, component discovery). Turn this on to move that walk onto a worker too —
  the only thing that removes the remaining frame cost in scenes with *thousands* of saveable
  actors. **Tradeoff (the EasyMultiSave model):** your `OnPreSave` / `GetSaveFlags`
  hooks then run on a worker thread, so they must be thread-safe — don't
  spawn/destroy actors, touch timers, or mutate the world from them. Actor state is read off
  the game thread while gameplay ticks (a value may be a frame stale), and components added to
  an actor *after* it registers may not be captured in this mode. Set it in Project Settings
  (it is read when actors register, i.e. at level load), and only after confirming a real
  gather hitch and that your save hooks are thread-safe. Most games never need it.
- **Enable Auto Save** — off by default. When on, the game saves itself on a timer
  during gameplay. The countdown only advances during unpaused play on the server/
  standalone (never on clients), a save already in progress is skipped (not queued),
  and saves run through the async pipeline so there is no frame hitch. Auto-saves do
  *not* rotate the backup history — *Auto Backup Previous Save Data* applies to your
  explicit saves only, so the frequent timer never churns through it.
  Leave it off for menu-driven or multiplayer games — or toggle it at runtime:

  ```cpp
  // Gate auto-save to gameplay (e.g. in your GameMode's BeginPlay / a pause menu):
  SM->SetAutoSaveEnabled(true);
  SM->SetAutoSaveEnabled(false);

  // Drive a "saving soon" HUD indicator:
  const float Secs = SM->GetSecondsUntilAutoSave();   // -1 when disabled
  ```
- **Auto Save Interval (Seconds)** — seconds of unpaused gameplay between auto-saves
  (default 300, minimum 5).
- **Auto Save Slot Name** — the slot the auto-save writes to (default `AutoSave`,
  kept separate from manual saves; falls back to *Default Slot Name* if empty).
- **Capture Auto Save Thumbnail** — on by default. Grabs a viewport thumbnail
  alongside each auto-save (the same async screenshot path as
  `SaveThumbnailFromViewport`, so no frame hitch), so the auto-save slot shows a
  preview in a load menu just like a manual save. No effect where there is no
  viewport (e.g. a dedicated server).
- **Thumbnail Format** — PNG (default, lossless) or JPEG (smaller). Controls how
  slot thumbnails are encoded (and thus their stored size); they're saved portably
  through the platform save system either way. **Thumbnail JPEG Quality** (1–100,
  default 90) applies when JPEG is selected. Loading auto-detects the format, so
  changing this never breaks existing thumbnails.
- **Auto-Load Save (Startup & Map Travel)** — off by default. When on, the
  configured slot is loaded automatically as the game's first world starts *and*
  again after every map travel, so a single-player game restores with no Blueprint
  or C++ setup. It fires once per world (never per streamed sublevel) and does
  nothing if the slot has no save yet. Leave it off for menu-driven ("Continue"
  button) or multiplayer games, which control load timing themselves.
- **Default Slot Name** — the slot used by *Auto-Load Save* (default
  `Slot_0`; also the auto-save fallback slot).
- **Game Save Version** — your game's save-data version (see *Versioning* above).
- **Runtime Actor Redirects** / **Level Redirects** — keep old saves loading after
  you rename assets (see *Redirectors* below).

## Redirectors

When you rename or move things during development, existing saves still reference
the old names. Two optional maps under **Project Settings → Plugins → SaveCore Pro
→ Redirectors** patch that up on load — both are empty by default and cost nothing
until you add an entry.

- **Runtime Actor Redirects** — for a Blueprint that SaveCorePro spawns at runtime
  (or resurrects). Map the **old class path** to the **new class** and old saves
  spawn the new one instead. The key accepts any form the editor gives you — the
  full generated path `/Game/Blueprints/BP_OldEnemy.BP_OldEnemy_C`, the object path
  without `_C` (`…/BP_OldEnemy.BP_OldEnemy`), or a bare **Copy Package Path**
  (`/Game/Blueprints/BP_OldEnemy`) all match; the value is picked from the class
  dropdown. Only consulted when the saved class fails to load directly, so a stale
  entry is harmless. (Map-placed actors are matched by their level path, so they
  don't need this — it's for runtime/persistent/resurrected actors.)

  > Note: renaming a Blueprint in-editor leaves a redirector behind, so old saves
  > keep loading **without** an entry here. You only need a mapping once that
  > redirector is gone — after **Fix Up Redirectors**, or if the asset was deleted,
  > recreated, or its C++ class renamed.

- **Level Redirects** — for a renamed or moved map. Per-level world data and
  per-level player positions are keyed by the map's package path, so after a rename
  that data no longer matches. Map the **old level key** to the **new** one, e.g.
  `/Game/Maps/OldMap` → `/Game/Maps/NewMap`, and the load re-buckets everything
  (world records, player transforms, the slot's displayed level) onto the new map.
  Use the package path without any PIE prefix.

## Multiplayer & authority

All save/load operations run on the server (or in standalone) only — calls on a
client are ignored. For reconnecting players, call `ApplySinglePlayerData(PS)`
from `GameMode::PostLogin`; it restores that player from the in-memory cache with
no disk I/O.

Each player is keyed by a stable identity that survives restarts: the online
subsystem ID (Steam/EOS/…), the local-player index for single-player and local
co-op, or — for AI bots — their pawn's stable ID. A remote human with no online
subsystem (e.g. a bare LAN session) has no restart-stable identity and is skipped
rather than risk restoring one player's data into another. For guaranteed
multiplayer persistence, run an online subsystem.

## Save versioning

SaveCore Pro keeps its *own* file format backward-compatible automatically, so
old saves keep loading after plugin updates — nothing to do for that.

Versioning is for when **your game's** data changes — you rename a field, change
an enum's meaning, or restructure an inventory — and an old save would otherwise
load wrong values. The flow has three parts.

### 1. Set a version for your data

**Project Settings → Plugins → SaveCore Pro → Game Save Version.** Start at `1`.
Every save you write is stamped with this number. Bump it each time you make a
breaking change to your saved data.

### 2. Migrate on load, where you have the data

After a load, `GetLoadedSaveVersion()` returns the version the file was written
with, and `GetCurrentSaveVersion()` returns your current setting. Compare them and
patch the old data. The natural place is the object's `OnPostLoad` hook — its
fields have just been restored, so you can fix them up in place.

Say version 1 stored health as a 0–1 percentage in `HealthPercent`, and in
version 2 you switched to an absolute `Health`. Bump **Game Save Version** to `2`
and handle the old files:

**C++**

```cpp
// MyCharacter.h
UPROPERTY(SaveGame) float Health = 100.f;
UPROPERTY(SaveGame) float HealthPercent = 1.f;   // legacy (v1) field, kept for migration

// MyCharacter.cpp
void AMyCharacter::OnPostLoad_Implementation()
{
    auto* SM = GetGameInstance()->GetSubsystem<USCPSaveManagerSubsystem>();

    // GetLoadedSaveVersion() is the version THIS file was saved at.
    if (SM->GetLoadedSaveVersion() < 2)
    {
        // v1 save: rebuild the new field from the old one.
        Health = HealthPercent * MaxHealth;
    }
}
```

**Blueprint** — in your actor's **On Post Load** event: **Branch** on
`Get Loaded Save Version` `<` `2` → set `Health` from the legacy field. Both
nodes are pure (no execution pin).

> Keep old fields around (don't delete them) until you're confident no old saves
> remain, so the migration code still has something to read from.

### 3. (Optional) Check a slot before loading

To warn the player or refuse an incompatible save *before* applying it, peek the
slot's version without loading it:

```cpp
int32 FileVersion = 0;
if (SM->GetSaveSlotVersion(TEXT("Slot_0"), 0, FileVersion))
{
    if (FileVersion > SM->GetCurrentSaveVersion())
    {
        // Save is from a NEWER build than this one — don't load it.
        return;
    }
}
```

`GetLoadedSaveVersion()` returns `0` before anything is loaded. All of these are
exposed to Blueprint (`Get Loaded Save Version`, `Get Current Save Version`,
`Get Save Slot Version`).

---

© Penguru Games. All Rights Reserved.
