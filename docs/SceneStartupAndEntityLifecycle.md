# Scene Startup And Entity Lifecycle

Read this before creating or changing gameplay prefabs that contain `Entities.Entity`, before writing new `EntityComponent` subclasses, and before touching scene startup code.

## Runtime Entry Points

Application startup is owner-driven. Do not add parallel startup paths to new components.

1. `ApplicationContext.Initialize()` runs before scene load, instantiates the `ApplicationContext` resource, creates the Application DI context, and registers application-wide services through `ApplicationBuilder`.
2. `Menu` chooses the session mode. `NewGameClick()` clears slots, starts a session with `eSceneStart.NewGame`, then loads build index `1`. `LoadLastClick()` loads shared data and opens the saved scene build index.
3. Every gameplay scene has `GameSceneContext`. Its `Awake()` creates the Game context, starts or joins the Session context, resolves scene dependencies, builds `GameSceneBuilder`, then calls `Context.CreateAll()`.
4. `GameSceneContext.Start()` is the single scene runtime entry point. It calls `GameSceneFlow.StartScene()`.
5. `GameSceneFlow` selects one path by `GameSessionOptions.SceneStart`:
   - `Loaded`: verify saved scene, load current slot, select scene variant, initialize runtime, resume quests, initialize HUD, then play first-visit flow if needed.
   - `NewGame`: select scene variant, initialize runtime without loading a slot, load quest runtime state, initialize HUD, start the initial quest, then run first-visit flow.
   - `TravelFromLocation`: select scene variant, initialize loaded runtime, move player to spawn point, save the slot.
6. Base scene runtime initialization is `GameSceneFlow.InitializeBaseSceneRuntime() -> EntityRuntimeInitializer.InitializeRuntime()`.

`EntityRuntimeInitializer` finds every `Entity` with `FindObjectsByType<Entity>(FindObjectsInactive.Include)` and calls `Entity.Initialize()` once. This happens after save data is applied for loaded scenes and before quest runtime/HUD work.

## Entity Initialization Contract

`Entity` is the aggregate owner for its child `EntityComponent` instances and for component save data.

`Entity.Initialize()` does this exact sequence:

1. Guard against a second initialization. A repeated call throws.
2. Collect all `EntityComponent` instances in children, including inactive objects.
3. Validate that every component has a non-empty, unique `Id` inside this entity.
4. Build saved component data by `EntityComponentSaveDTO.ComponentId`.
5. Call `Initialize()` on every component.
6. Call `Load(saveDTO)` on every component, passing `null` when no saved data exists.
7. Call `Activate()` on every component.

Use the phases this way:

- `Initialize()`: build runtime-only objects and discover sibling/child components. Do not read persisted values here except serialized prefab config.
- `Load(saveDTO)`: apply saved state. Accept `null` as clean/default state for new games and newly added components.
- `Activate()`: start behavior that depends on all entity components already being initialized and loaded.
- `Save()`: return an `EntityComponentSaveDTO` subclass only when the component owns persisted state. Return `null` for stateless components.

Do not start Entity runtime behavior from `Awake`, `Start`, `OnEnable`, lazy `Ensure...` methods, retry binding, or UnityEvents. If a component needs startup work, put it in the correct Entity phase and let `GameSceneFlow -> EntityRuntimeInitializer -> Entity.Initialize()` own the call.

## Prefab Authoring Rules

For a gameplay prefab that represents an entity:

- Put one `Entity` on the aggregate root that should be saved/loaded as a single object.
- Put `EntityComponent` subclasses on the root or children under that `Entity`. They are discovered with inactive children included.
- Keep each `EntityComponent.Id` stable. The default is the component type full name, which is fine only when there is at most one component of that type under the same entity. If there can be duplicates, set explicit unique ids in the prefab.
- Do not duplicate ids. The entity will throw during scene startup.
- Configure `Entity` save file/origin/resource path intentionally when the object is shared-state, baseline scene object, or runtime-spawned resource-backed object.
- For runtime-spawnable entities, `EntitySaveDTO.SpawnSceneObject()` loads the prefab from `Resources` using `SpawnResourcePath`; author the prefab and path so save/load can restore it without scene-only references.
- When editing prefab, scene, or asset state, use Unity Editor/MCP workflows. Do not raw-edit `.prefab`, `.unity`, `.asset`, `.meta`, controller, material, or animation YAML.

## New EntityComponent Checklist

When adding a new component for an entity:

- Inherit from `EntityComponent` unless the object is intentionally outside the Entity aggregate.
- Keep the component focused. Use other components/interfaces for separate concerns instead of one large entity-specific script.
- Put dependencies from DI in `[Inject] Init(...)` only when they are context services. Put entity runtime wiring in `Initialize()`.
- If the component consumes another runtime built during `Initialize()` (for example stats), follow the existing owner/consumer pattern such as `StatsRuntimeOwner` and `IStatsRuntimeConsumer` instead of adding lazy lookup from `Update`.
- If the component persists data, create one `EntityComponentSaveDTO` subclass and return it from `Save()`. Do not implement independent `ISaveable` for data that belongs to the entity aggregate.
- Store runtime-only fields as non-serialized/private state. Save DTOs are data only and should not contain logic.
- Do not add public API for future use. Add only the methods and properties used by the current prefab/component wiring.

## Files To Check First

- `Assets/Scripts/GameLoop/ApplicationContext.cs`
- `Assets/Scripts/GameLoop/GameSceneContext.cs`
- `Assets/Scripts/GameLoop/GameSceneFlow.cs`
- `Assets/Scripts/GameLoop/EntityRuntimeInitializer.cs`
- `Assets/Scripts/Entities/Entity.cs`
- `Assets/Scripts/Entities/EntityComponent.cs`
- `Assets/Scripts/Entities/EntitySaveDTO.cs`
- `Assets/Scripts/Saving/Saver.cs`
