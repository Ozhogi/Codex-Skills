---
name: reference-database-list-authoring
description: Create or refactor Outcasts systems to use ReferenceDatabase-backed collector references and persisted lists instead of raw string keys. Use when adding a new ReferenceCollectorBase subclass, replacing serialized string IDs such as trader keys with typed references, wiring scene collectors into ReferenceDatabaseAsset, or storing save-state lists keyed by reference identity.
---

# Reference Database List Authoring

## Baseline

Use this together with `$apply-patch`, `$code-style`, `$unity-mcp-orchestrator`, `$saving`, `$mcp-unity-validate-script`, and `$unity-recompile-menuitem` when project files change.

Keep the change small: define a typed reference identity, store that identity in save data, and adapt callers. Do not introduce a manager/service layer unless an existing owner cannot perform the lookup.

## Existing Pattern

Reference-backed identities live under `Assets/Scripts/References/` and derive from `ReferenceCollectorBase`.

Typical shape:

```csharp
[Serializable]
public sealed class NpcReference : ReferenceCollectorBase
{
	public override string CollectorTypeId => typeof(NpcReference).FullName;
}
```

Scene-collected references usually implement:

```csharp
public static List<ReferenceCollectorBase> Collect(object context)
```

and are merged from `ReferenceDatabaseAsset.CollectReferencesOnSceneSaving`. If the reference is configured in a component, collect it from loaded scene roots, set its container id to the scene GUID, ensure `ElementId` is non-empty and unique, then return it.

## Workflow

1. Read the target component/service/save classes first. Identify the current raw key and every call site.
2. Create one new sealed reference class under `Assets/Scripts/References/` named after the domain, for example `TraderReference`.
3. Add a typed field to the authoring component, for example `public TraderReference Reference`, instead of `string _traderKey`.
4. Convert runtime APIs to accept the typed reference, not a raw string. If save data needs a compact key, create a separate serializable key value object/struct that stores `CollectorTypeId`, `ContainerId`, and `ElementId`.
5. Store save-state lists by the typed key. Prefer a `List<...SaveState>` with `Find/GetOrCreate` methods over dictionaries when Unity serialization and existing code already use lists.
6. Update tests to construct references directly through public fields/properties. Do not use reflection for ordinary setup when fields can be public authoring data.
7. If scene or prefab component fields must be migrated, use Unity MCP/prefab tools. Do not raw-edit `.unity`, `.prefab`, `.asset`, or `.meta` files unless the user explicitly approves that fallback.
8. Validate changed scripts, force Unity recompile, and run focused EditMode tests for lookup, save round-trip, and caller behavior.

## Reference Key Rules

Use a separate key type when the identity must be serialized into save state. The key should be plain serializable data with no Unity object dependencies.

Recommended shape:

```csharp
[Serializable]
public struct TraderReferenceKey : IEquatable<TraderReferenceKey>
{
	[SerializeField] private string _collectorTypeId;
	[SerializeField] private string _containerId;
	[SerializeField] private string _elementId;

	public string CollectorTypeId => _collectorTypeId;
	public string ContainerId => _containerId;
	public string ElementId => _elementId;
	public bool IsEmpty => string.IsNullOrWhiteSpace(_elementId);

	public static TraderReferenceKey From(TraderReference reference) => reference == null
		? default
		: new TraderReferenceKey(reference.CollectorTypeId, reference.ContainerId, reference.ElementId);
}
```

Equality must compare all identity fields with `StringComparison.Ordinal`. Do not use display name as identity.

## Collector Rules

For scene-owned component references:

- Use the scene asset GUID as `ContainerId` in editor, falling back to `scene.path` outside editor-only code.
- Track duplicate `ElementId` values per scene and assign `Guid.NewGuid().ToString("N")` only when missing or duplicated.
- Preserve user-authored `ElementId` when it is unique.
- Register the collector in `ReferenceDatabaseAsset.CollectReferencesOnSceneSaving`; keep the hook bounded to the new collector plus existing collectors.

For global lists not tied to a scene, override `UsesContainerId` in the collector only if the existing `ReferenceDatabase` dropdown pattern supports that domain. Otherwise keep container id explicit.

## Save-State Rules

Save data must remain DTO-like:

- Put persisted entries in the existing owner save state, usually `GameSaveState` or a nested save state such as `TradingSaveState`.
- Do not store `ReferenceCollectorBase` instances in runtime save state when a compact key is enough.
- Do not add migration/backfill from old raw string keys unless the user asks.
- Initialize list fields at declaration and clone every new persisted field.

Example list owner:

```csharp
public List<TraderStockSaveState> TraderStocks = new();

public TraderStockSaveState GetOrCreateStockState(TraderReference reference, int capacity)
{
	TraderReferenceKey key = TraderReferenceKey.From(reference);
	for (int i = 0; i < TraderStocks.Count; i++)
	{
		if (TraderStocks[i].ReferenceKey == key)
			return TraderStocks[i];
	}

	TraderStockSaveState created = new(key, capacity);
	TraderStocks.Add(created);
	return created;
}
```

## Tests

Add or update focused EditMode tests for:

- Two different references create two different saved entries.
- JSON round-trip preserves the reference key and list contents.
- Runtime service methods use the reference key and no longer require raw strings.
- Collector generation assigns missing duplicate ids when applicable.

Use public fields/properties for setup. Reflection is acceptable only for legacy private data that the task is not changing.
