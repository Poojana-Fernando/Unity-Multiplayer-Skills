---
name: game-dev-journey
description: >-
  Guides Unity 6 game development for this user's journey, especially The Last
  Railway and later games built the same way. Use when adding gameplay, train
  or railway systems, third-person control, combat, enemies, multiplayer,
  save data, world streaming, audio, UI, scenes, or architecture. Use when the
  user asks how to extend the game, where code should live, or how to avoid a
  second system. Read ProjectSettings and Packages/manifest.json when versions
  matter; do not trust a remembered package version over the project files.
---

# Game development journey

Build the game that already exists. Add a seam when a job has no owner. Never add a second owner for a job that already has one.

This skill is the working method. Project facts live in [project-map.md](project-map.md). Contract, authority, and save rules live in [architecture-rules.md](architecture-rules.md). Read those files before changing train, network, save, combat, or world-streaming code.

## Before writing code

1. Name the player-visible outcome in one sentence.
2. Find the type that already owns that job. Search `Assets/_Game`, `Assets/Scripts`, and `Assets/_ProjectArchitecture` before creating a type.
3. If an owner exists, extend it or add a thin adapter. Do not replace it.
4. If no owner exists, add the smallest contract first, then one implementation.
5. Name the authority domain before any replicated field. See [architecture-rules.md](architecture-rules.md).
6. Read `ProjectSettings/ProjectVersion.txt` and `Packages/manifest.json` when a package API is involved.
7. Leave vendor folders alone unless the user explicitly asks to patch a vendor bug.

Stop and ask when the change would add an assembly definition, a second locomotive integrator, a second session client, a scene-streaming loader, or a save file format that is not already in the project.

## Where code goes

| Kind of change | Location | Namespace |
| --- | --- | --- |
| Train motion, consists, splines, levers, train audio | `Assets/_Game/Scripts/Train` | `Game.Train` |
| Sessions, replication, menus, settings | `Assets/_Game/Scripts/Net` | `Game.Net` |
| Campaign save and resume | `Assets/_Game/Scripts/Net/Campaign` | `Game.Campaign` |
| On-foot guns and combat glue that is ours | `Assets/_Game/Scripts` | Match the neighbouring script |
| Mounted guns | `Assets/Scripts/MountedGuns` | `MountedGuns` |
| Interfaces, ids, enums, save slices, events | `Assets/_ProjectArchitecture` | `TheLastRailway.Architecture.*` |
| Editor builders and one-shot setup | `Assets/Editor` | `*.EditorTools` when a namespace is used |
| Invector body, Dead Author combat kit | Vendor folders | Do not add gameplay rules here |

`Assets/_ProjectArchitecture` is a contract layer. Files there are interfaces, enums, or data shapes. No `MonoBehaviour`, no `ScriptableObject` asset, no scene load, no file write, no network message.

Every new asset under `Assets` gets a Unity `.meta` file so the guid stays stable. Do not add an `.asmdef` unless the user approves an assembly-boundary change. Gameplay scripts compile in the default game assembly.

## One owner

| Job | Owner | Do not also create |
| --- | --- | --- |
| Locomotive integration | `TrainSplineDriver` | A second movement integrator |
| Server train simulation and the driver seat | `NetworkTrainDriver` | A second throttle rule |
| Spline route and turnouts | `RailwayNetwork`, `RailwayTopology`, `RailRoute` | A rebuilt knot graph |
| Lever replication | `RailwaySwitchboard` | A second lever writer |
| Host, join, LAN fallback | `SessionManager` | A second relay or session client |
| Connection approval and `Level01` load | `NetworkGameBootstrap` | A renamed bootstrap flow |
| Player body pose | Owning client via `NetworkPlayerSetup` | Server-driven feet |
| Shared world props | `NetworkWorldSync` | A second world-state writer |
| On-foot locomotion | Invector `vThirdPersonController` | A parallel character motor |
| Guns, health, inventory | Dead Author combat kit plus `NetworkCombat` | A second health pool |
| Enemies | `ZombieAI` | A replacement brain unless asked |
| Train audio playback | `TrainAudio` | A second engine mixer |

A `Request…` method records intent. It does not move the train, throw a lever, load a region, or open a save file. An adapter performs the effect, and only after the authority check passes.

Adapters pull state when asked. They do not push into contracts from `Awake` or `Start`.

## Feature workflow

Copy this list and keep it current while implementing:

```
- [ ] Owner named, or a new contract justified
- [ ] Authority domain named for anything replicated
- [ ] Data shape separated from the MonoBehaviour
- [ ] Vendor code untouched
- [ ] No invented speeds, health, or player caps
- [ ] Save slice updated only if this fact must survive a resume
- [ ] Scene and prefab wiring described, not assumed
- [ ] Console is clean after scripts compile
```

Work in this order:

1. **Read.** Open the owner script and the matching contract. Note what the contract says it does not do.
2. **Data.** Add an id or a plain struct before adding a component. Stable ids are authored identities. A spline index, a `NetworkObjectId`, and a scene instance id are not stable ids.
3. **Rule.** Put the decision in one place. UI, animation, and audio read that decision. They do not decide it again.
4. **Adapter.** A runtime type implements the contract by reading the owner. The adapter writes back only through the owner's existing method.
5. **Replication.** One writer per domain. The owning client writes `PlayerBody`. The server writes `TrainMotion`, `RailwayJunctions`, `WorldState`, and session membership.
6. **Presentation.** Camera, animator, and audio follow state. They do not own it.
7. **Verify.** Recompile. Read the Console. If the Unity editor is connected, exercise the feature in play mode. Say what was not run.

## Design rules that keep later work cheap

- Dependencies point inward. Definitions do not reference gameplay interfaces. Gameplay does not reference vendor types from inside `TheLastRailway.Architecture`.
- A new interactable implements a permission check. It does not replace `TrainSeatInteractor`, `RailwayLeverInteractor`, or weapon pickups.
- Enemy work exposes activation and a behaviour seam. Pathing, sensing, and attacking stay on `ZombieAI` until a design replaces that brain on purpose.
- Regions are identifiers plus metadata. They become scenes only when a streaming change says so. Gameplay today enters through the Bootstrap scene into `Level01`.
- Save data is sliced. Train, railway, objectives, and regions store and restore independently. Health, weapons, ragdolls, join codes, spline geometry, and mixer parameters stay out of those slices.
- Configuration values live on the component or asset that already uses them. Empty settings definitions stay empty until a design pass fills them. Do not copy `NetworkGameBootstrap`'s player cap into a second field.
- Integer fields that store enums are storage. They are not tuning values.
- Comments state what the type will not do when that boundary is easy to violate. Match the tone of `ITrainControl` and `INetworkAuthority`.
- Prefer `sealed` on adapters. Keep public surface small: a state read, a request method, a `TryCapture` / `TryRestore`.
- Do not allocate in per-frame train or character loops. Do not add a new `Update` that polls what an event or a network variable already reports.

## Multiplayer check

Before adding a `NetworkVariable`, RPC, or scene object that other players see:

1. Which `AuthorityDomain` owns the write?
2. Does `INetworkAuthority` say the local peer may decide it?
3. Is there already a seat, pickup, or lever rule that further limits who may request it?
4. What does a late joiner need, and does `NetworkWorldSync` or the existing network variable already carry it?
5. What happens to this fact on disconnect? Session facts die with the session. Campaign facts go through the save slices.

Client prediction, lag compensation, and interest management are unspecified. Do not invent them inside a feature change.

## Combat and character check

- Locomotion stays on the Invector body. Camera rigs stay on Cinemachine unless the feature is the train seat camera or an existing first-person mode.
- Damage goes through the combat kit's existing damageable path. `IInteractable` is not `IDamageable`.
- A mounted gun is a world object with a user. It is not a weapon inventory entry unless the user asks to unify them.
- Spawning and respawn go through `PlayerSpawnUtility` and `PlayerRespawnController`.

## Scene and content check

- New gameplay content belongs in the scene the feature actually runs in. Do not create a parallel `Level01` to avoid editing the current one.
- Editor setup scripts under `Assets/Editor` may build objects. Runtime code must still work if that menu has not been clicked, or the setup must be documented as a required step.
- Prefab overrides beat duplicated prefabs. One player prefab, one locomotive, one wagon type, unless the user asks for a variant.

## Starting a later game with this method

Use the same shape even when the fiction changes:

1. Write a one-page map: live systems, their owners, and the jobs that have no owner yet.
2. Put contracts in a folder that cannot run the game.
3. Pick authority domains before the first replicated feature.
4. Split save data by facts that resume, and leave presentation and session glue out.
5. Keep third-party character, combat, and shader packs behind their own folders.
6. Add features in the order: read-only adapter, authority check, interaction, persistence, streaming last.

Streaming is last because it is the first system that loads and unloads scenes.

## How to answer implementation requests

Lead with the owner you will extend, or say that no owner exists and name the contract you will add. Then list the files. Then make the change.

When a request mixes a safe extension and a second system, do the extension and say which part needs a separate decision. Do not scaffold the second system "for later" in the same change.

After the edit, report:

- What now owns the behaviour
- Which authority domain writes it, if it is shared
- What you compiled or played
- What you could not verify

## Out of scope unless asked

- Rewriting the Invector controller or the combat kit
- DOTS or ECS replacements for the current `MonoBehaviour` game
- A new transport beside Netcode for GameObjects and the existing Facepunch transport
- Store certification, monetization, or analytics pipelines
- Deleting scenes, prefabs, or the recovered railway scene
- Git commits, pushes, or package upgrades done as a side effect of a feature
