---
name: ai-mob-authoring
description: Create or modify Unity enemy mobs that use the project's AI State Machine framework. Use when Codex needs to author mob prefabs, copy or configure StateMachineGraph assets, set StateMachineMono blackboard overrides, wire Pawn/CombatComponent/sensors/AttackPerformer/PathPoints, add states/conditions/events, or debug AI behavior for scorpions or other mobs in Outcasts.
---

# AI Mob Authoring

## Core Rule

Use this skill for AI State Machine mob work in Outcasts. Load and follow `$code-style`, `$unity-mcp-orchestrator`, `$prefab-creation`, and `$apply-patch` as applicable before editing scripts, graph assets, prefabs, or documentation.

Prefer Unity/MCP or Editor scripts for prefab, ScriptableObject, and asset changes. Do not hand-edit `.prefab`, `.asset`, `.unity`, `.meta`, controller, material, or graph YAML unless the edit is small, well understood, and Unity/MCP is unavailable or explicitly unsuitable.

## Reference

For framework details and authoring checklists, read `references/ai-state-machine-mob-authoring.md` before creating or materially changing a mob, graph, state, condition, event, blackboard variable, or AI prefab setup.

Also consult the project developer docs when useful:

- `Assets/Scripts/AI/State Machine/Docs/AI-State-Machine-Mob-Authoring.md`
- `Assets/Scripts/AI/State Machine/Docs/AI-State-Machine-Mob-Authoring.html`
- `Assets/Scripts/AI/State Machine/Docs/AI-State-Machine-Mob-Authoring.pdf`

## Standard Workflow

1. Inspect the existing example first: `Assets/Prefabs/Gameplay/Agents/AI/Enemy Scorpion.prefab` and `Assets/ScriptableObjects/AI/State Machine/Scorpion.asset`.
2. Identify the intended mob type: melee, ranged, passive, ambusher, patrol-only, or custom behavior.
3. Decide whether to copy an existing graph or create a new `StateMachineGraph`. Copy the closest existing graph unless the behavior is genuinely new.
4. Configure prefab root components: `StatsRuntimeOwner`, `Health`, `Movement`, `Vision`, `Combat`, `Pawn`, `Agent`, `Damageable`, `NavMeshAgent`, `PawnMovementController`, `StateMachineMono`, one `IPawnSensorSource`, `CombatComponent`, and an attack component exposing `AttackPerformer`.
5. Do not add or keep `ParametersComponent`, `VitalsComponent`, or vital resource assets on migrated/new mobs. During migration, copy needed values into Stat/Reserve config, then remove the old components/assets from the prefab and remove obsolete code after reference scan.
6. Configure blackboard variables and `StateMachineMono` overrides. Treat missing exposed overrides as the most likely cause of broken runtime behavior.
7. Wire behavior graph flow: `Entry -> Patrol`, detection event/condition -> combat branch, chase/navigation -> attack range transition, attack/cooldown loop, death/damage events when required.
8. Validate changed C# with `$mcp-unity-validate-script`; after C# or asmdef edits run `$unity-recompile-menuitem` and read Unity console errors.
9. For prefab/asset work, verify through Unity/MCP or generated asset files. Use Play Mode logs (`StateMachineMono.Logs`) when debugging runtime transitions.

## Authoring Defaults

Use `VisionSensor` for simple direct raycast vision. Use `SensorsListPawnSensorSource` when the mob needs modular `AI.Sensors.SensorsList` behavior such as vision, audibility, and memory.

Keep exactly one active `IPawnSensorSource` on the AI root unless the code is intentionally extended to support multiple sources. `CombatComponent` currently uses the first matching component returned by Unity.

Keep graph assets generic. Put per-prefab or per-scene references in exposed blackboard variables and assign them through `StateMachineMono` overrides rather than direct scene references inside graph assets.

## Ranged Combat Feel

For bow/ranged NPCs, build misses as last-moment release corrections: aim and body tracking should read as committed to the target during wind-up, then the shot may drift or be nudged off target on release. Avoid early obvious off-target aiming unless the design explicitly asks for telegraphed incompetence. Keep a configurable chance for guaranteed hits so ranged enemies remain threatening; later this can be tied to player/NPC bow skill or passive-skill tuning.

## Stat/Reserve Setup

- `StatsRuntimeOwner` is the save owner for reserves and active effects.
- `Health` declares `Stat.MaxHealth`, `Stat.HealthRegeneration`, and `Reserve.Health`.
- `Movement` declares `Stat.MoveSpeed` for `PawnMovementController`.
- `Vision` declares `Stat.SightRange` and `Stat.Visibility` for sensors.
- `Combat` declares combat tuning stats. Attack execution still goes through `AttackPerformer`.
- `Damageable` reads new `Health`/`StatsRuntimeOwner` state only. Do not add legacy health/vitals fallback or legacy vitals saving.

## New Runtime Classes

When adding states, conditions, or events:

- Put one top-level C# type per file.
- Derive states from `State`, conditions from `Condition`, events from `Event`.
- Use `[Serializable]`; add `[MenuPath(...)]` for event/condition discovery when matching existing patterns.
- Store graph-authored data in serialized fields.
- Use `[SerializeReference] BlackboardVariable<T>` for graph variables.
- Do not rename child variable/state/condition/event types already serialized through `[SerializeReference]` unless the user explicitly asks for a migration/backfill.
- Resolve mob components in `Init(GameObject gameObject)`.
- Return `Failure` when required dependencies or blackboard values are missing.
- Unsubscribe events in `OnDispose()`.

## Final Checks

Before finishing, report:

- Changed scripts, graph assets, prefabs, and documentation.
- Which mob prefab and graph asset were modified or created.
- How blackboard exposed variables should be assigned.
- Unity validation/recompile results and any remaining console errors.
- Any unrelated dirty files you observed and intentionally left untouched.
