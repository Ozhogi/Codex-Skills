---
name: code-style
description: Project C# code style and engineering rules for project-owned C# under Assets/Scripts only. Use when writing or reviewing C# code in Assets/Scripts, choosing type layout, SaveData/state/facade naming, runtime/editor boundaries, or general architecture there. For BehaviourInject DI/context/provider/injection work, use `$behaviourinject`.
---

# Code Style

## Purpose

Use this skill to keep project C# changes small, explicit, and aligned with the existing Unity architecture.

Scope: this skill applies only to project-owned C# source under `Assets/Scripts`. Use it for code audits and code changes in that directory, and do not use it to enforce style for packages, generated files, project settings, assets outside `Assets/Scripts`, or other non-script content unless a more specific skill explicitly routes there.

Mandatory gate: before creating any Unity C# `.cs` script under `Assets/Scripts`, load and apply this skill before choosing the path, filename, type layout, dependencies, or implementation. This also applies when another skill is the primary workflow, including `$apply-patch`, `$unity-mcp-orchestrator`, `$create-assets-menu-item`, or `$unity-recompile-menuitem`.

For BehaviourInject DI, context lifecycle, providers, injected views/controllers, commands, events, factories, or execution order, load `$behaviourinject`. For Unity test authoring and test helper `MonoBehaviour` placement, load `$test-writing`. For prefab asset creation, prefab edits, UI prefab hierarchy, RectTransform layout, TextMeshPro components, UI layers, Canvas/CanvasScaler/GraphicRaycaster, or prefab migration work, load `$prefab-creation` and `$ui-prefab-authoring` instead of keeping those rules here.

When changing or extending one of the project's large polymorphic systems, also read `references/ProjectPolymorphicAbstractions.md` before choosing the abstraction boundary.

When replacing serialized types, changing serialized data shape, converting lists to `[SerializeReference]`, or planning temporary migration/backfill/refactor work, also read `references/DataMigrationRefactoring.md` before choosing names, compatibility strategy, or migration tooling.

## Formatting Sources

Follow `.editorconfig` and `Outcasts.sln.DotSettings` before writing or heavily editing C#.

Compact formatting baseline:

- Use LF and valid UTF-8.
- Use tabs for C# indentation.
- Require braces for control statements.
- Use `var` only when the type is evident.
- Use explicit object creation when the type is evident.
- Allow expression-bodied methods, constructors, and local functions.
- Wrap near 150 columns.
- Preserve existing attribute, declaration, and block arrangement when Rider says to keep it.

Formatting rules never override stricter architecture rules below.

## Unity Script Assets And Packages

- Unity GameObject tags are forbidden for project gameplay and tooling. Do not add project tags, assign tags, search by tag, or use `CompareTag`, `GameObject.FindGameObjectWithTag`, `FindGameObjectsWithTag`, `gameObject.tag`, `transform.tag`, or serialized tag-name strings. Use layers, explicit serialized references, marker components, interfaces, `EntityTags`, physics materials, or domain data instead.
- Existing custom entries in `ProjectSettings/TagManager.asset` are legacy only. Do not add new ones; remove legacy assignments through Unity/AssetDatabase paths when touching the affected objects.
- Never hand-create Unity `.meta` files or invent GUIDs for scripts. Let Unity import generate `.meta` files.
- Never edit, replace, swap, regenerate, or migrate GUID values in `.cs.meta` files. A script GUID belongs to that script asset permanently.
- When renaming or moving a script, move the existing `.cs.meta` with it unchanged.
- Do not resolve missing scripts or class merges by rewriting script GUIDs in `.meta`, prefabs, scenes, or assets. Use Unity/AssetDatabase/component migration paths under the appropriate Unity/prefab skill instead.
- Do not edit package source, embedded packages, `Packages/`, or `Library/PackageCache/` files to solve project problems. Treat packages as read-only reference/API context, then implement fixes in project-owned `Assets/` code or configuration.

## File And Type Layout

- Keep one C# type per `.cs` file: one file equals one top-level class, struct, interface, enum, or record.
- Do not create nested C# types. Keep classes, structs, interfaces, enums, and records as separate top-level types in their own files.
- Do not put multiple top-level types in one file unless Unity serialization or source generation explicitly requires it.
- Before writing or editing any C# file, count the top-level types in the intended result. If there would be more than one, split them into separate `.cs` files named after each type.
- Avoid plural/helper container filenames such as `Lists.cs`, `Helpers.cs`, or `Types.cs`.
- Keep C# source as valid UTF-8. Russian/Cyrillic comments, string literals, and localization text must not become mojibake, `?`, replacement character `U+FFFD`, or unwanted `\uXXXX` escapes.

## Architecture Defaults

- Prefer the smallest direct solution. Do not add abstractions for future use.
- Keep systems simple. Do not create extra `Registry`, `Pipeline`, `Resolver`, `Context`, `Manager`, `Service`, `Provider`, `View`, or similar boundaries unless the current implementation has a real demonstrated need for that exact boundary.
- Prefer polymorphism over `switch`/type-code branching. Use interfaces, virtual methods, strategy objects, command dispatch, or event dispatch when behavior varies by type or mode.
- When a system already has a polymorphic extension point, use it instead of adding a parallel procedural API. Add a new implementation behind the existing interface, abstract base type, factory, or `[SerializeReference]` list rather than adding one public method, enum branch, or special case per concrete behavior.
- Do not pass broad runtime/save/service objects such as `GameRuntimeState`, full save roots, or facade services into polymorphic operation `Execute` methods. Follow container commands: pass the concrete dependencies the operation needs directly through its constructor, expose `Execute()` with no broad arguments, and do not wrap those dependencies in a context class unless the project already has a real shared runtime context contract for that subsystem.
- Before adding a new interface or side-channel for UI, interaction, or player feedback data, trace the existing owner chain from source component through event/presenter first. In Outcasts interaction prompts, prefer extending `InteractiveEntity -> Interactor -> InteractTips` with a fallback over adding trade-specific prompt plumbing.
- Do not use reflection in runtime code. Prefer properties with `{ get; private set; }`.
- Treat static state, static services, static helpers, singleton-style access, and the Singleton pattern as antipatterns by default. Do not add `Instance` accessors or private-constructor single-instance objects for runtime ownership; use explicit ownership, DI, serialized references, or per-call command objects instead.
- Allow `static` only for pure math, enum-specific utilities, constants, extension methods, or similarly stateless helpers that do not hide ownership, dependencies, lifecycle, or mutable state.
- Do not use `goto`.
- Use UniTask instead of Coroutines.
- Prefer event-driven or physics-event-driven flow over polling in `Update`. Use `Update` only when continuous per-frame sampling is the actual requirement.
- Write compact code and avoid duplicated forwarding layers. Do not create paired `public static` functions plus duplicated private instance functions for the same behavior; choose one clear API shape.
- Do not pass functions as `Action`; pass interfaces or direct references.
- For sequential calls, use `Debug.Log()` when a bool-gated log is needed.

## Runtime Ownership

- Require a single explicit runtime entry point for every workflow and subsystem.
- When a lifecycle already has an owner such as scene flow, entity initialization, save/load, DI composition, or a command execution path, fix that owner instead of adding a second local entry point.
- Do not pass long-lived services or dependencies as method parameters to objects that own behavior needing them. If a class, mode, or service uses a dependency as part of its responsibility, inject it through the constructor/DI and keep it as a field; reserve method parameters for per-call data, commands, DTOs, or event payloads.
- Do not add `Awake`, `Start`, `OnEnable`, lazy initialization, retry binding, idempotent rescue paths, or serialized boolean fallbacks to make a subsystem start itself when the intended owner failed to call it. Treat that as masking the ownership bug.
- Use fixed, linear `ICommand.Execute -> ExecuteAsync` or equivalent owner-driven flow instead of scattered `EnsureSomething` calls from multiple entry points.
- Treat `EnsureSomething` methods as an antipattern by default. Do not add them as lazy initialization, validation, repair, or workflow coordination wrappers; make initialization ownership explicit, validate at the boundary, and use a clearly named command/service method when a real workflow step is needed.
- Do not create multiple independent gameplay entry points through combinations of UnityEvents and Unity lifecycle methods. Choose one explicit owner for the workflow; keep UnityEvents as view/inspector hooks only when truly needed.

## Public API And Naming

- Do not create public properties or functions for future use. Public API must be used by the current implementation, required by Unity serialization/lifecycle, or required by an existing contract.
- Do not use `internal`; use `public` or `private` intentionally.
- Use explicit `private` modifiers for private fields, methods, properties, constructors, and nested types; do not rely on C# default private accessibility.
- Do not abbreviate variable names. Use complete words that describe the value clearly.
- Avoid names that collide with or read like reserved C# keywords, framework terms, Unity lifecycle methods, or broad CLR concepts; prefer `Begin` over `Start` and `Pattern` over `Type`.
- Do not name new or replacement types with temporary quality/version prefixes or suffixes such as `New`, `Fixed`, `Updated`, `Better`, `Improved`, `Final`, `V2`, or similar wording. Choose the domain role name directly. If replacing an old type that is not serialized through `[SerializeReference]`, temporarily rename the old type from `MyName` to `MyNameObsolete` and give the replacement the canonical `MyName` name.
- Name polymorphic operation/action classes by the concise action itself, following container command names such as `AddAmount`, `ClearSlot`, `Sell`, or `BuyBack`; do not add suffixes like `Operation`, `Action`, `Command`, `Handler`, or domain repetition unless an existing interface contract explicitly requires it. For operation logs, pass the operation object and use `operation.GetType().Name` at the logging boundary instead of passing operation names as strings.
- Use `OnUpdate` for frame-update hook names instead of `Tick`.
- Use `Factor` for multiplicative values instead of `Multiplier`.
- Prefer constructors over `SetValues()`/post-construction initialization methods. Constructors with many arguments are acceptable if they make dependencies clear.
- Prefer public fields over private fields plus trivial `Set...()` methods when the value is intentionally mutable/public state.
- For Unity-serialized mutable state, prefer `[field: SerializeField] public Type Name { get; private set; }` over `private` serialized fields plus public setter methods.
- Name `MonoBehaviour` components with concise domain nouns when the noun is clear; do not add a `Component` suffix by default. Prefer `Faction : MonoBehaviour` over `FactionComponent : MonoBehaviour`. Use a suffix only when it resolves a real ambiguity in existing code.
- Use explicit project suffixes for non-save state and facade types: `RuntimeDTO` for runtime data transfer state, `Service` for facade-backed services, `Provider` for providers that expose view facade instances, and `View` for the view-facing facade itself.
- Do not use a `SaveDTO` suffix for new persisted data unless the project already has that concrete contract and the change is extending it.
- Property getters must be cheap and unsurprising. Do not clone, allocate snapshots, mutate state, dispatch events, or perform load/save work from a property getter. Use an explicitly named method such as `Create...Snapshot()` when copying is required.

## MonoBehaviour And User Feedback

- Keep `MonoBehaviour` components small, self-contained, and reusable when possible.
- Prefer BehaviourInject and `RequireInjector`/`Injector` for `MonoBehaviour` dependencies by default. If a component needs services, player/session objects, event dispatchers, providers, or other long-lived runtime objects, load `$behaviourinject` and use `[Inject]` with verified injector coverage instead of manual dependency forwarding.
- Prefer composing entities from focused components over large entity-specific behaviours that mix unrelated responsibilities.
- Do not serialize references to components that live on the same GameObject. Resolve same-object component dependencies with `GetComponent`/`TryGetComponent` from the explicit owner initialization path instead of `[SerializeField]` fields.
- Test, debug, cheat, and development-only scripts follow the same runtime ownership and public API rules as gameplay scripts.
- Do not emit logs from `Update`, `FixedUpdate`, polling loops, repeated decision loops, or per-frame/per-tick callbacks unless the log is gated by a state change, explicit user action, or a serialized debug cooldown.
- When creating a new gameplay, UI, or other user-facing system, include player feedback in the first implementation: animation or visible state motion, using DOTween when it is the smallest suitable fit, and sound hooks or audio playback.
- If feedback is intentionally omitted, state why before implementing.

## UI And Input Code

- Before changing `Assets/Scripts/UI/General/UIManager.cs` or related window/input behavior, read `.codex/docs/UIManager.md`.
- Any player-visible text added from code must be localized through the project localization tables, except numeric-only or symbol-only content. Do not leave user-facing language as string literals, fallback-only TMP text, or enum labels.
- UI/input views must use the project Inputs service or an injected input abstraction.
- Do not read `Keyboard.current`, `Mouse.current`, or raw Input System globals directly from a view when an existing project input action/service can represent the command.

## Serialization And Editor Data

- Do not use `[FormerlySerializedAs]` unless the user explicitly requests that exact migration/backfill mechanism.
- For polymorphic Unity-serialized collections, prefer `[SerializeReference] IAbstract[]` over concrete-type arrays/lists or enum/switch dispatch. Odin Inspector provides a type picker for `[SerializeReference]`, so prefer this before introducing ScriptableObject definitions only to select implementation types.
- Do not rename concrete child types already used through `[SerializeReference]`; Unity serializes the concrete type name. Rename only with an explicit migration/backfill plan requested by the user.
- Definition: `SO` means `ScriptableObject`.

## Save And Runtime State

- For any gameplay system, design and implement in this order: `SaveState -> RuntimeState -> RuntimeExecution -> EditorDefinitions -> Editor Tools`.
- Runtime must be able to execute from save/runtime state before ScriptableObject/editor authoring layers are added.
- DTOs and save data contain no logic and are not used for dependencies.
- Persisted save data must use the existing `Saving.SaveData` hierarchy. Do not invent parallel `SaveDTO`, `SaveState`, wrapper, bridge, or migration types when an existing `SaveData`-derived contract can represent the data.
- For `EntityComponent` persistence, return a `SaveData`-derived component save type, usually an existing or new subclass of `EntityComponentSaveDTO`, and let `Entity` remain the `ISaveable` owner.
- Do not make an entity component implement independent `ISaveable` for data that belongs to the entity aggregate.
- SaveData fields must be initialized by field initializers or constructors.
- Do not add `EnsureInitialized()`, lazy `??=` initialization, migration/backfill normalization, or scattered null-repair paths for save data unless the user explicitly asks for migration or corrupted-save recovery.
- Treat required save data parameters, event payloads, and collection entries as present in runtime consumers.
- Do not add `null` fallback branches unless the API explicitly accepts missing or corrupted save data.

## Legacy And Migration Work

- Do not create migration code, migration assets, conversion tools, or backfill paths for a new system unless the user explicitly asks for migration. New systems should start clean by default.
- When replacing a legacy gameplay system, do not keep runtime fallbacks, compatibility branches, or dual-read paths to the old components.
- Migration may read old component values only as source data, then must write the new component configuration and delete or obsolete the old scripts/assets after a reference scan. For prefab/scene component migration, load `$prefab-creation` and use Unity/AssetDatabase/component migration paths.
- For Stat/Reserve migration specifically, treat `ParametersComponent`, `VitalsComponent`, `VitalResourceData`, `VitalResource`, `ParametricValue`, and old Vital/Vitals assets as legacy.
- Do not add or preserve fallback reads/writes to legacy Stat/Reserve types; copy required values into Stat/Reserve config, then remove legacy components and delete or obsolete legacy scripts/assets after references are gone.
