# The Last Railway — project map

Read this when the task is in the current Unity project. If a path or type is missing, search the project. The map is a guide, not a guarantee that a later edit kept the same file.

Baseline recorded from the project: Unity `6000.5.8f1`, URP `17.5.0`. Confirm both in `ProjectSettings/ProjectVersion.txt` and `Packages/manifest.json` before using a version-specific API.

## Stack already chosen

| Need | Use |
| --- | --- |
| Replication | Netcode for GameObjects |
| Internet sessions | Unity Services Multiplayer, with LAN fallback in `SessionManager` |
| Transport | Facepunch transport package already in the manifest |
| Player input | Input System |
| Track geometry | Unity Splines. `SplineContainer` is the geometric track |
| Camera | Cinemachine 2.x, plus the existing train-seat camera |
| Enemies | AI Navigation. `ZombieAI` owns the NavMesh behaviour |
| Body animation constraints | Animation Rigging |
| Render pipeline | URP |
| Editor automation | `com.gamelovers.mcp-unity` when the editor is open |

Do not add a second package for a job this table already names.

## Folders

| Path | What lives there |
| --- | --- |
| `Assets/_Game/Scripts/Train` | Locomotive, consist, route, junctions, train audio, riders |
| `Assets/_Game/Scripts/Net` | Sessions, player setup, combat replication, menus, settings |
| `Assets/_Game/Scripts/Net/Campaign` | Campaign snapshot, capture, resume |
| `Assets/_Game/Scripts` | First-person glue, interactors, audio directors, menu juice |
| `Assets/_Game/Vendors/Invector-3rdPersonController_LITE` | On-foot body. Namespace `Motiong.vCharacterController` |
| `Assets/_Game/Vendors/DeadAuthorGames/ThirdPersonCombatKit` | Guns, health, inventory. Namespace `DeadAuthorGames.ThirdPersonCombat` |
| `Assets/Scripts/MountedGuns` | World-mounted guns. Namespace `MountedGuns` |
| `Assets/_ProjectArchitecture` | Contracts only. Namespace `TheLastRailway.Architecture` |
| `Assets/_ProjectArchitecture/Documentation` | `ARCHITECTURE.md` and `EXISTING_PROJECT_AUDIT.md` |
| `Assets/Editor` | Setup menus and build hooks |
| `Assets/_Game/Scenes` | `Bootstrap.unity`, `Level01.unity`, and experimental scenes |

Gameplay starts in `Bootstrap` and the host loads `Level01`. Experimental scenes (`trainGuns`, `forestDark`, `MyThirdperson`, recovered railway scenes) are not the shipping flow.

## Live types

### Train and railway

| Type | Role |
| --- | --- |
| `TrainSplineDriver` | Distance, signed speed, throttle, brake, wheel lock along a `RailRoute` |
| `TrainWheelRig` | Poses body and wheels on the rail centreline |
| `TrainConsist`, `TrainCarFollower` | Wagons coupled behind the locomotive |
| `RailSpan` | One continuous run on one spline |
| `RailRoute` | Ordered spans the locomotive is following |
| `RailRouteNet` | Netcode serialization of that route |
| `RailwayTopology` | Turnouts and plain joins derived from spline knot links |
| `RailwayNetwork` | Junction index. Extends the route when the train crosses a turnout |
| `RailwayJunction` | One turnout: lever, lamps, diverging flag |
| `RailwaySwitchboard` | Server-written lever positions |
| `RailwayLeverInteractor` | Player use of a lever |
| `NetworkTrainDriver` | Server simulates. One seated player sends throttle and brake |
| `TrainSeat`, `TrainSeatInteractor`, `TrainCamera` | Driver seat and seated camera |
| `TrainAudio` | Engine, track, brake, and horn from replicated speed and wheel lock |
| `MovingPlatform`, `PlatformRider`, `PlatformRiderSync` | Standing on the moving train |

`TrackSegmentId` is not a spline index. The mapping from an authored segment id onto a `RailSpan` is a future adapter, not something to invent inside a feature.

### Players, combat, enemies

| Type | Role |
| --- | --- |
| `vThirdPersonController`, `vThirdPersonInput` | On-foot locomotion and input |
| `PlayerGunController`, `PlayerHealth`, `WeaponInventory` | Combat kit |
| `ZombieAI` | Idle, patrol, alert, chase, crawl, attack, dead |
| `NetworkPlayerSetup`, `NetworkPlayerRegistry` | Spawned players and the local player |
| `NetworkHealth`, `NetworkCombat`, `NetworkWeaponState` | Replicated health and weapons |
| `PlayerRespawnController`, `PlayerSpawnArea`, `PlayerSpawnUtility` | Downed state and spawn placement |
| `MountedGun`, `MountedGunUser` | Guns fixed in the world |
| `ExplosiveDamageable` | Damageable prop replicated with world sync |

There is no shared interactable on the runtime objects. Seats, levers, and pickups are separate components. `IInteractable` in the architecture folder is only a permission seam.

### Session and world

| Type | Role |
| --- | --- |
| `SessionManager` | Host and join-by-code, plus LAN fallback |
| `NetworkGameBootstrap` | Connection approval, four-player cap, loads `Level01` |
| `LobbyMenuController`, `MainMenuUI`, `LoadingScreen`, `PauseMenuUI` | Menus |
| `NetworkWorldSync` | Server authority for dismemberment, explosives, and world pickups |
| `OfflineOnlyObject` | Exists only outside a session |
| `GameplayCampaignSaveService` | Adapts `ISaveService` onto the campaign snapshot |

## Namespaces already in use

`Game.Train`, `Game.Net`, `Game.Campaign`, `Game.FirstPerson`, `DeadAuthorGames.ThirdPersonCombat`, `Motiong.vCharacterController`, `MountedGuns`, `TheLastRailway.Architecture`, `TrueTrackSystem`, `BKPureNature`, `Sirenix.OdinInspector`.

New gameplay code joins one of the `Game.*` namespaces. New contracts join `TheLastRailway.Architecture`. Do not put game rules in vendor namespaces.

## Integration order when a contract needs a runtime

Do this only when the user allows the change to touch existing scripts.

1. Read-only adapters for session, train motion, and the railway graph. They read `SessionManager`, `TrainSplineDriver`, and `RailwayNetwork`. They write nothing back.
2. Authority checks from the domain table before any new replicated feature.
3. Interaction and enemy adapters only when a caller needs the shared contract. Leave the concrete components in place.
4. Objective and region ids when those features are designed, still without loading scenes.
5. Save writes through the existing slices after the read models are stable.
6. World streaming as its own change. It is the first contract that would load and unload scenes.

## Known gaps

Do not fill these with placeholder numbers or fake content:

- Train damage has bands and no hit points, parts, or repair rules.
- Stations, destinations, objectives, obstacles, and points of interest have definitions and no content.
- `RailwaySettingsDefinition`, `WorldSettingsDefinition`, and `SessionSettingsDefinition` are empty on purpose.
- No region is a scene yet.
- Service startup already lives on `NetworkGameBootstrap`. A second initializer is a design decision.
