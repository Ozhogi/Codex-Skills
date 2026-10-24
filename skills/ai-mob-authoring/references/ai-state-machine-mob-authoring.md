# AI State Machine Mob Authoring Reference

## Framework Shape

AI State Machine is the graph runtime used by scorpions and other mobs. A `StateMachineGraph` ScriptableObject stores `EntryNode`, `StateNode`, `EventNode`, `SubGraphNode`, and `Blackboard`. A `StateMachineMono` component on the mob root creates the runtime blackboard, clones graph states/conditions/events, injects blackboard variable references, and ticks the active state.

Runtime classes are serialized into graph assets but run against the mob GameObject through `RuntimeContext`. `VariablesContainer<T>.CreateInstance` clones the graph-authored object, injects `BlackboardVariable<T>` fields, then calls `Init(GameObject)` so the runtime copy can resolve components on the mob.

## Key Runtime Contracts

- `State`: implement `Enter()`, `Update()`, `Exit()`. Return `Running`, `Success`, or `Failure`.
- `Condition`: implement cheap `Check()` logic. It can run every frame on `Running` transitions.
- `Event`: subscribe in `Init`, call `ExecuteCallback()` when the event fires, unsubscribe in `OnDispose()`.
- `Transition`: target node plus required status, condition check mode (`All`/`Any`), and condition list.
- `BlackboardVariable<T>`: graph variable reference used by states, conditions, and events.

## Existing Building Blocks

States:

- `Patrol`: moves through `PathPoints` using `PawnMovementController`.
- `NavigateToTarget`: follows a `Transform` blackboard target.
- `NavigateToLocation`: moves to a `Vector3` blackboard value.
- `Warning`: turns toward target, triggers animation/marker, sets combat mode.
- `Charge`: waits while facing target, then writes a charge destination.
- `Attack`: calls `AttackPerformer.PerformAttack()`.
- `Wait`: timed wait state.

Conditions:

- `HasSpottedEnemies`: reads `CombatComponent.HasSpottedEnemies`.
- `IsInCombat`: reads `CombatComponent.IsInCombat`.
- `RangeCondition` / `RangeMinMaxCondition`: checks distance between two blackboard `Transform` values.
- `VariableComparison`: compares blackboard variables.

Events:

- `EnemyTargetChangedEvent`: listens to `CombatComponent.OnTargetChanged`, writes `Move Target`, then triggers transitions.
- `TakeDamageEvent`: listens to `Damageable.OnDamage` above a threshold.
- `DeathEvent`: listens to `Damageable.OnDeath`.
- `CollisionEvent`: handles collision-triggered graph flow.

## Typical Blackboard Variables

| Name | Type | Exposed | Use |
| --- | --- | --- | --- |
| `Self` | `Transform` | yes | Root transform for range checks. |
| `Move Target` | `Transform` | no | Current target written by `EnemyTargetChangedEvent`. |
| `Path Points` | `PathPoints` | yes | Patrol route. |
| `Attack Performer` | `AttackPerformer` | yes | Attack execution dependency. |
| `WorldMarkersUI` | `WorldMarkersUI` | yes when `Warning` is used | Marker UI dependency. |

## Prefab Root Checklist

A combat mob usually needs these on the root GameObject or the expected child hierarchy:

- `StatsRuntimeOwner` for Stat/Reserve initialization and save state.
- `Health` for `Reserve.Health` and death semantics.
- `Movement` for `Stat.MoveSpeed`.
- `Vision` for `Stat.SightRange` and `Stat.Visibility`.
- `Combat` for authored combat stats.
- `Pawn` with valid `FactionData`.
- `Agent` so generic sensors can discover the object.
- `Damageable` wired to the new `Health` component.
- `NavMeshAgent`.
- `PawnMovementController`.
- `StateMachineMono` with graph assigned.
- One active `IPawnSensorSource`: `VisionSensor` or `SensorsListPawnSensorSource`.
- `CombatComponent` on the same GameObject as the sensor source and `Pawn`.
- Attack component exposing `AttackPerformer`.
- Animator, colliders, visuals, ragdoll/drop components as needed for the mob.

Do not add or keep `ParametersComponent`, `VitalsComponent`, `VitalsCollection`, or vital resource assets on migrated/new mobs. During migration, copy needed values into Stat/Reserve config, then remove the old components/assets from the prefab and remove obsolete code after reference scan.

## Graph Creation Workflow

1. Copy `Assets/ScriptableObjects/AI/State Machine/Scorpion.asset` when creating a similar melee mob. Create a fresh `StateMachineGraph` only for substantially different behavior.
2. Rename the graph and update state node names so runtime logs are readable.
3. Add or prune blackboard variables. Mark per-prefab references as exposed.
4. Assign the graph in `StateMachineMono`.
5. Fill exposed overrides on the prefab or scene instance.
6. Verify `EntryNode` has a valid transition to the starting state.
7. Verify every transition target exists and required status matches the intended timing.

## Basic Melee Graph

Recommended starting graph:

1. `Entry -> Patrol`.
2. `Patrol` runs `Patrol` state with `Path Points`.
3. `Patrol` has a `Running` transition with `HasSpottedEnemies == true` to `Warning` or `Chase`.
4. `EnemyTargetChangedEvent` writes `Move Target` and transitions to the combat branch.
5. `Warning` faces `Move Target`, triggers marker/animation, and sets `CombatComponent.IsInCombat = true`.
6. `Chase` runs `NavigateToTarget` using `Move Target`.
7. `Chase` has a `Running` transition with `RangeCondition(Self, Move Target, attackRange, inside=true)` to `Attack`.
8. `Attack` runs `AttackPerformer.PerformAttack()` and returns to `Chase`, `Wait`, or a cooldown state.
9. `DeathEvent` transitions to a dead/deactivation branch.

For ranged mobs, keep the same detection/combat spine, but use larger attack range/stopping distance and a ranged attack performer.

## Sensor Choices

Use `VisionSensor` when direct raycast vision is enough. Configure bone, authored distance, angles, cast count, and layer mask. With `StatsRuntimeOwner`, `VisionSensor` receives sight range from `Stat.SightRange`; target visibility comes from `Stat.Visibility`. Do not keep legacy sight/visibility `ParameterData` fallback in migrated/new mobs.

Use `SensorsListPawnSensorSource` when the mob needs modular sensors. Configure `SensorsList` entries (`Vision`, `Audibility`, `Memory`) and keep `Memory` last. Ensure targets have both `Agent` and `Pawn` in the same hierarchy. Set the practical sensor range high enough because `CombatComponent` uses it to expire stale enemies.

## Debugging

1. Turn on `StateMachineMono.Logs` and watch `STATE:` / `EVENT:` messages.
2. If the graph does not start, check graph assignment, `EntryNode`, and first transition.
3. If patrol fails, check `Path Points` override and `PawnMovementController` / `NavMeshAgent`.
4. If detection fails, check sensor source, target `Agent`/`Pawn`, faction relation, and layer masks.
5. If target is not written, check `EnemyTargetChangedEvent` blackboard variable binding.
6. If attack never starts, check `RangeCondition` variables, attack range, and target transform.
7. If attack starts but does nothing, check `Attack Performer` override and animation/damage timing.
8. If health does not save/load, check `StatsRuntimeOwner` and `StatsRuntimeSaveDTO`; do not reintroduce legacy vitals runtime fallback or saving.
9. Always read Unity console after C# or prefab/asset changes.

## Adding A New State

Use this shape and keep one type per file:

```csharp
using System;
using UnityEngine;

namespace StateMachine.States
{
    [Serializable]
    public class MyState : State
    {
        [SerializeReference] private BlackboardVariable<Transform> _target;

        protected override void Init(GameObject gameObject)
        {
        }

        public override Status Enter()
        {
            return _target.Value ? Status.Running : Status.Failure;
        }

        public override void Exit()
        {
        }

        public override Status Update()
        {
            return _target.Value ? Status.Running : Status.Failure;
        }
    }
}
```

## Adding A New Condition

Conditions should be cheap and dependency-light:

```csharp
using System;
using UnityEngine;

namespace StateMachine.Conditions
{
    [Serializable]
    public class MyCondition : Condition
    {
        protected override void Init(GameObject gameObject)
        {
        }

        public override bool Check()
        {
            return true;
        }
    }
}
```

## Adding A New Event

Events must unsubscribe in `OnDispose()`:

```csharp
using System;
using UnityEngine;

namespace StateMachine.Events
{
    [Serializable, MenuPath("My Event")]
    public class MyEvent : Event
    {
        protected override void Init(GameObject gameObject)
        {
        }

        protected override void OnDispose()
        {
        }
    }
}
```

## Common Failure Modes

- Direct scene object reference stored in graph asset instead of exposed blackboard override.
- Empty `StateMachineMono` overrides.
- `EntryNode` lacks a valid transition.
- Transition intended for live checking uses `Success` instead of `Running`.
- `EnemyTargetChangedEvent` writes a different `Move Target` variable than states/conditions read.
- `CombatComponent` cannot find an active `IPawnSensorSource` on the same GameObject.
- Target lacks `Pawn`, `Agent`, or an enemy faction relation.
- `StatsRuntimeOwner` is missing, so stats/reserves are not initialized or saved.
- `Memory` is not last in `SensorsList`.
