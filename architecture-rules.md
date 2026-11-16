# Architecture rules

The contract folder describes the game. It does not run the game. Nothing in `Assets/_ProjectArchitecture` is attached to a scene, and nothing there references `Game.Train`, `Game.Net`, or the combat kits.

## What a contract is allowed to be

An interface, an enum, or a data shape.

A method named `Request…` stores intent for an adapter. Calling it must be safe even when no adapter is bound: it cannot move a transform, change a lever, load a scene, or write a file.

Summaries should say the limit in one or two sentences, the way `ITrainControl` says request methods do not integrate physics or networking.

## Dependency direction

Definitions do not reference gameplay interfaces. Session code may ask about authority. Authority does not open a relay. Audio and configuration are leaves: they do not call back into simulation.

Expected runtime direction, once an adapter exists:

1. Game flow publishes a phase change. Listeners pull what they need. They are not pushed during another system's startup.
2. Train control reads motion and submits throttle and brake. The railway graph says which segment that motion sits on. Route validation says whether a destination is reachable. Neither writes the live spline route.
3. Exploration boundaries and occupancy describe where players are. A streamer may request load or unload from that. Occupancy itself does not call `SceneManager`.
4. Objectives point at a region. Enemies expose activation only. Obstacles point at a segment when they block track.
5. Authority is consulted before a request is accepted.
6. Save capture reads train, railway, objective, and region slices. It does not read health, weapons, or network ids.

## Authority domains

`SessionRole` is `None`, `Host`, or `Client`. `PlayerIdentity` is a stable id plus a local flag. It is not a Netcode client id.

| Domain | The one decision | Who writes it in the live game |
| --- | --- | --- |
| `Session` | Who is in the session, and which scene the host starts | `SessionManager`, `NetworkGameBootstrap` |
| `PlayerBody` | Where that player stands and looks | Owning client through `NetworkPlayerSetup` |
| `TrainMotion` | Distance, speed, and wheel lock along the route | Server through `NetworkTrainDriver` |
| `RailwayJunctions` | Straight or diverging lever position | Server through `RailwaySwitchboard` |
| `WorldState` | Shared world facts such as explosive props | Server through `NetworkWorldSync` |

One player at a time may send throttle and brake, and the server simulates. A train-control request must pass both the `TrainMotion` check and that seat rule.

Add a new domain only when a fact has no row above. Name the single writer in the same change as the first replicated field.

## Railway identities

- `TrackSegmentId` is authored. It is not a spline index.
- `TrackEndpoint` is one end of a segment. A `TrackConnection` joins two ends.
- A plain join has no junction. A turnout names a `JunctionId`.
- A junction has three legs: stem in, stem out, and branch.
- `JunctionRouteChoice` is `Straight` or `Diverging`.
- `IRailwayGraph` looks up a segment, a junction, and the connections leaving an endpoint.
- `IRouteValidator` answers whether an origin segment can reach a destination. Failure carries a reason string. It does not change `RailRoute`.

`RailwayTopology` remains the code that reads knot links. An adapter may project `RailwayNetwork` into `IRailwayGraph`. That projection is not an excuse to rebuild the spline.

## Regions

Regions are not scenes.

```
Unloaded -> Loading -> Active -> Unloading -> Unloaded
```

`RequestLoad` and `RequestUnload` are intents. Refuse unload while `IRegionOccupancy` reports a player inside, once that rule is implemented. `PersistentRegionMetadata` stores lifecycle and whether the region was visited. It does not write a file.

## Save slices

`GameSaveData` is versioned. Schema starts at 1 so a later format change can be detected.

| Slice | Stores | Does not store |
| --- | --- | --- |
| Train | Route id, distance along the route, damage band, throttle notch, brake | Hit points, parts, repair |
| Railway | Junction id and route choice | Spline knots, scene references |
| Objectives | Objective id and status | Rewards, HUD state |
| Regions | Region id and visited flag | Loaded scene objects |

Left out on purpose: player health, weapons, dismemberment, ragdolls, network client ids, join codes, relay state, spline geometry, audio mixer parameters.

`ISaveService.TryCapture` and `TryRestore` are the only operations. The campaign adapter (`GameplayCampaignSaveService`) is the current implementation. A new persisted fact joins one of the four slices, or the user approves a new slice and a schema bump in the same change.

## Audio and configuration

`IAudioService` plays and stops an `AudioCueId`. A cue id is not a mixer channel. `TrainAudio` remains the locomotive's engine, track, brake, and horn.

`IReadOnlyConfiguration<TDefinition>` reads one definition object. Do not invent speeds, health, or player caps to make a definition look finished.

## Adapter checklist

- Reads an existing owner listed in [project-map.md](project-map.md)
- Writes back only through that owner's existing API
- Pulls when asked
- Does not run from `Awake` into the contract
- Does not construct a second `SessionManager`, train integrator, or relay client
- Lives outside `Assets/_ProjectArchitecture`

## Phase names

Coarse phases are Boot, Menu, Lobby, InSession, and Paused. Phase code may answer whether a phase may be entered. It does not load scenes and it does not replace `NetworkGameBootstrap`.
