---
name: behaviourinject
description: Unity BehaviourInject (BInject) dependency injection workflows. Use when adding or refactoring DI with Contexts, registration patterns, Injector order, [Inject]/[InjectEvent], events, commands, factories, or hierarchy contexts in this Unity project.
---

# BehaviourInject


## Workflow

Use this skill for BInject-specific design and refactors. Read `references/behaviourinject.md` only when exact API signatures, examples, or uncommon patterns are needed.

- Decide context scope: global named context or local hierarchy context.
- Context names are code constants. Do not expose context names as serialized fields, editable strings, config, or scene data.
- Application and Session context owners live across scene loads and must call `DontDestroyOnLoad(gameObject)` after duplicate-context checks. Scene context owners are scene-bound and must not be marked DontDestroyOnLoad.
- Use `Context` classes only to create, own, parent, and destroy context hierarchy.
- Keep context lifecycle separate from application/gameplay initialization. Context bootstrap creates scopes and registers dependencies; initialization/startup code runs separately from that scope creation.
- Do not make ordinary services own, create, reset, or dispose contexts. For example, a scene loading service stays a scene loading wrapper and must not create `SceneContext`.
- Treat static DI bootstrap, static context wrappers, singleton context access, and `[RuntimeInitializeOnLoadMethod]` context creation as anti-patterns. Create BInject contexts from explicit `MonoBehaviour` context components in `Awake`, placed in the scene or bootstrap prefab.
- Do not register or inject `Context` classes or instances as dependencies.
- Do not create static wrappers or facades around DI resolution, registered services, or `IEventDispatcher.DispatchEvent`.
- Keep statics limited to application bootstrap code that creates or ensures the root `ApplicationContext`; after bootstrap, use injected dependencies, command/event payloads, or the owning context setup flow.
- Create a Builder for every context; the Builder owns dependency, type, factory, and command registration.
- Keep registration code out of Context classes except constructing/running their Builder.
- Register services with `RegisterType<TService>()` or `RegisterTypeAs<TService, TInterface>()` for autocomposition.
- Fix cyclic dependency errors by changing ownership and dependency direction in game code. Move the responsibility to the narrower service that actually owns the workflow, split overly broad orchestration services, or make existing callers delegate to the narrower service. Do not break cycles by adding a second/manual `Init`, passing registered services after `CreateAll`, resolving dependencies from services, adding command/event indirection only to avoid the constructor graph, or editing BInject internals. Dependencies still enter only through registered constructor dependencies or `[Inject] Init` during `Context.CreateAll`/autocomposition.
- `[Inject]` methods are for receiving dependencies only: assign injected dependencies to fields/properties and do not start gameplay, dispatch startup commands, generate levels, spawn objects, load scenes, or run the next workflow step from an `[Inject]` callback.
- `[Inject] void Init(...)` is only for passing DI dependencies. Do not use an `[Inject] Init` method for per-call data, view args, state payloads, configuration values, or manual setup parameters.
- DI dependencies must never be optional in constructors or `[Inject] void Init(...dependencies)` methods: do not use default parameter values such as `= null`, nullable dependency parameters, `null` guards, fallback branches, or optional behavior. Register required dependencies in the Builder and treat them as present after construction/injection.
- Do not clear or null out injected service fields in `OnDisable`, `OnDestroy`, or UI close/hide callbacks. Lifecycle methods may unsubscribe listeners or stop tweens, but injected dependencies remain required dependencies owned by the context lifecycle.
- Inject dependencies directly into the view/controller that uses them; do not pass injected services from one view to another view through manual `Init(...)` forwarding.
- Prefer BehaviourInject for almost every `MonoBehaviour` that needs services, player/session objects, event dispatchers, providers, or other long-lived dependencies. The default implementation is `[Inject] void Init(...)` plus `RequireInjector` on the component type or an `Injector` component on the same prefab/scene object.
- If a component has any `[Inject]` or `[InjectEvent]` method, verify its prefab/scene object has injector coverage before finishing: inherit from `RequireInjector`, add an `Injector` component, or document the existing parent/context mechanism that injects it. Do not leave a plain `MonoBehaviour` with `[Inject]` on an object that BInject will not scan.
- Use manual dependency forwarding only as a deliberate exception for short-lived per-call data or simple value arguments, not for registered services or long-lived runtime objects. State the exception before implementing it.
- For any view/controller `MonoBehaviour`, receive services and long-lived dependencies through `[Inject] void Init(...)`. Do not pass services through `SetArgs`, request DTOs, bind payloads, or manual setup methods; those payloads carry only per-call view data. Runtime request/flow state belongs in the service or coordinator, with the view rendering the current request and reporting user commands back to the service.
- For prefab or scene UI components that trigger a service action, add `RequireInjector`/`Injector` coverage and inject the service into that component with `[Inject]`. Do not have a service discover its child UI with `GetComponentInChildren(...)` and call `Initialize(service)` on it. Examples: a cutscene skip button must inject `GameSceneFlow`, `FirstVisitCutsceneService`, or a narrow skip service directly; the cutscene service should not wire itself into the button manually.
- For scene objects needed by DI, register a plain provider object in the Builder; the `MonoBehaviour` registers itself into that provider in `Awake`.
- Keep provider objects as narrow as possible and do not wrap every scene object in ViewFacade/ViewProvider/FacadeService layers by default. Add those layers only when they remove real duplicated coordination, isolate a true view boundary, or support multiple implementations.
- Do not start DI-driven gameplay workflows from arbitrary `MonoBehaviour.Start`. `Start` does not guarantee that every required provider registration and injection across the scene happened in the intended order. After dependencies are injected, the next runtime step should be called from `Awake` or from an explicit bootstrap/context owner whose execution order is deliberately accounted for, not from an `[Inject]` callback.
- Do not dispatch BInject events merely to wrap a direct service call. If code needs to load a scene, call the scene loading service directly; reserve BInject events/commands for real decoupled workflows, not for replacing a straightforward method call. Plain C# events on a service are valid for UI/status notifications such as scene load begin/end.
- Register commands with `context.RegisterCommand<TEvent, TCommand>()`; do not register command implementations as normal dependencies.
- Pass per-call command parameters through event payloads and receive them with `[InjectEvent]`.
- Ensure the context `MonoBehaviour` runs before the `Injector`, and the `Injector` runs before dependent scripts. Prefer explicit execution order on context behaviours instead of static initialization.
- Prefer method injection with `[Inject]` / `[InjectEvent]` where appropriate. For dependency injection, use `[Inject] void Init(...dependencies)` and assign fields inside the method; do not use `[Inject] SomeType fieldName` field/member injection.

Reference: `references/behaviourinject.md`.