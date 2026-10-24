---
name: quest-dialogue-authoring
description: Edit Outcasts Unity quest and dialogue story graphs. Use when Codex needs to add or modify quest objectives, dialogue lines, dialogue choices, graph flow, story actions/conditions, or localization for StoryGraph `.qstgraph`/`.storydlggraph` assets in this project.
---

# Quest Dialogue Authoring


## Core Rules

- Read `AGENTS.md`, `$apply-patch`, `$unity-mcp-orchestrator`, and `$code-style` before changing project files.
- Treat `.qstgraph`, `.storydlggraph`, localization tables, scenes, prefabs, and ScriptableObjects as Unity Editor-owned state. Prefer Unity MCP, Unity APIs, or a focused Editor MenuItem for asset changes.
- Focused Editor MenuItem scripts used only to generate or patch assets must be deleted after the assets are generated, imported, saved, and verified. Keep permanent generator scripts only when the user explicitly asks for repeatable tooling.
- Dialogue authoring must always produce Graph Toolkit-backed `.storydlggraph` assets. Do not create, save, or serialize dialogue content directly as `DialogueGraphAsset` assets except when reading existing runtime data for migration or diagnostics.
- For Unity 6.5+ Graph Toolkit authoring, use the public/editor API from `Unity.GraphToolkit.Editor` through a temporary or reusable Editor script instead of raw YAML. Prefer patterns from the project compiler/importer: create or open the graph with `GraphDatabase`, mutate it with graph/node APIs such as `AddNode` and `Connect`, save/import through Unity, then verify the generated runtime asset. Unity's Graph Toolkit update notes confirm code-created graphs as a supported workflow: `https://discussions.unity.com/t/graph-toolkit-update-in-unity-6-5-alpha/1712169`. API reference for `GraphDatabase`: `https://docs.unity3d.com/Packages/com.unity.graphtoolkit@0.1/api/Unity.GraphToolkit.Editor.GraphDatabase.html`.
- Edit existing story graphs in place. Do not regenerate a graph from scratch, replace all node IDs, or rewrite unrelated serialized graph data.
- Keep graph diffs minimal: add or modify only the required nodes, wires, actions, conditions, positions, and localization references.
- Preserve existing `.meta` GUIDs and graph asset GUID references.

## ScriptableObject Naming

Name graph ScriptableObject/source assets as `Type Domain Descriptor Number`, with each segment in PascalCase words separated by spaces and the trailing number optional. Examples: `DialogueGraph Old Man Bark Water Ask 1`, `DialogueGraph Old Man Bark Water Ask`, `QuestGraph Repair Truck Find Fan`.

Use the graph type as the first segment (`DialogueGraph`, `QuestGraph`, etc.), then the domain/owner (`Old Man`, `Repair Truck`, `Sailboat`), then specific descriptors from broad to narrow. Avoid suffix-first names such as `Old Man Dialogue`, `Old Man World Bark`, or `Repair Truck Find Fan Quest` for new or renamed graph assets.

## Localization Requirement

Every newly added dialogue line, dialogue choice, quest title, and quest objective must have localization filled in both English and Russian before finishing the task.

Use the `Dialogue` string table collection unless the existing nearby graph clearly uses another collection:

- Shared data: `Assets/Localization/Dialogue/Dialogue Shared Data.asset`
- English: `Assets/Localization/Dialogue/Dialogue_en.asset`
- Russian: `Assets/Localization/Dialogue/Dialogue_ru.asset`

Do not invent `m_Id` values by hand. Add or update entries through Unity Localization APIs, MCP, or a small `Assets/Editor` MenuItem, then save assets. Raw YAML edits to localization tables are only acceptable for tiny, well-understood corrections to entries that already exist.

Prefer stable semantic keys for new quest text, for example:

```text
quest.<quest_id>.title
quest.<quest_id>.objective.<objective_id>
<dialogue_id>.<line_or_choice_id>
```

Avoid using English sentence text as a key for new content.

## Quest Graph Workflow

1. Identify the existing quest graph and current node sequence. For the initial quest, the graph id is `repair_truck.find_fan` and the source asset is `Assets/Resources/StoryGraph/Repair Truck/QuestGraph Repair Truck Find Fan.qstgraph`.
2. Treat `QuestStageNode` as the quest state. If the requested design needs several objectives visible and completable in the same state, attach those objective keys to the stage instead of inventing nested/sub-objective concepts under `QuestObjectiveNode`.
3. Add intermediate objectives by inserting nodes and rewiring only the affected edge when the quest state should advance after that objective. Preserve downstream objective keys unless the requested behavior requires a key change.
4. When objective text is new, add `quest.<graph_id>.objective.<node_key>` to `Dialogue_en` and `Dialogue_ru`.
5. If a dialogue event should advance the quest, attach a `CompleteObjectiveAction` to the exact dialogue line or choice block where the state change should happen. Do not put this on `DialogueSituationNode` unless runtime code confirms situation actions execute.
6. For dialogue situations selected by a current objective, use `QuestObjectiveActiveCondition` against the quest graph and objective key. It checks the active quest node and ignores already completed objectives.
7. When only the editor-facing name of a reference/flag/objective should change, update the database display name only. Do not change `ElementId`; serialized scene, prefab, graph, save, and condition references depend on it.

## Dialogue Graph Workflow

1. Find the parent dialogue selector graph and the target dialogue subgraph. For old-man water flow, `DialogueGraph Old Man.storydlggraph` selects `DialogueGraph Old Man Water Needs.storydlggraph` while the water flag is false.
2. Before adding or generating any dialogue asset, confirm the target is a Graph Toolkit `.storydlggraph` source. If tooling would emit a raw `DialogueGraphAsset`, change the tooling first or stop and report the mismatch.
3. Use `DialogueSituationNode` for selector/root dialogue graphs with conditional situations. For a minimal one-shot dialogue that is opened directly from an object and contains only a simple line/choice chain, a direct entry graph without a situation node is valid; `DialogueRunner` falls back to the graph entry node when no matching situation exists.
4. Add actions to existing line or choice nodes when possible. Add new dialogue line/choice nodes only when requested.
5. Any newly added line or choice must reference a `LocalizedString` table entry with both `en` and `ru` values.
6. To jump immediately from one dialogue situation into another, end the current dialogue subgraph with a `StoryEndGraphNode` that has a `StartDialogueAction` targeting the parent dialogue graph. `DialogueRunner` supports this by closing the active frame before starting the next dialogue.
7. After changing dialogue or quest graph sources, force Unity to import the touched `.qstgraph`/`.storydlggraph` assets and check console errors.

## Verification

- Run `$apply-patch` inspect/verify on edited text files, especially files containing Cyrillic.
- If C# Editor tooling was created or modified, validate with `mcp__unityMCP.validate_script`, force recompilation through `MCP/Force Recompile Scripts`, and read console errors, passing `unity_instance` on every MCP call whose schema supports it.
- Execute any temporary MenuItem through `mcp__unityMCP.execute_menu_item` with explicit `unity_instance="<target Name@hash>"` routing. If the exposed schema does not accept `unity_instance`, stop and report that routing is blocked for concurrent Unity work; do not claim asset changes succeeded from file edits alone when Unity import/API work was required.
- Verify the final diff does not contain broad whitespace churn, regenerated graph content, mojibake, replacement characters, or missing ru/en localization entries.
