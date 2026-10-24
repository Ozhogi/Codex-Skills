---
name: unity-mcp-orchestrator
description: Orchestrate Unity Editor work through MCP tools for scene, prefab, asset, script, console, and Editor-state tasks in this Unity project. For Unity test authoring and test execution, use `$test-writing`.
---

# Unity MCP Orchestrator

## Generated Asset Cleanup

After generating Unity assets, delete temporary generation scripts once the assets are created and verified. Keep a generation script only when the user explicitly asked for a permanent reusable tool or the skill workflow requires a permanent MenuItem; in that case, state why it remains.

## Unity MCP Routing

When this skill uses Unity MCP directly or through another skill, target the exact Unity Editor instance for the current project. Read/resolve `mcpforunity://instances` before project-scoped MCP work, choose the exact `Name@hash` whose project matches the current workspace, and pass that exact value as `unity_instance` on every Unity MCP tool/resource call whose schema supports it, including custom-tool discovery/execution and every nested `batch_execute` command's `params`.

For multi-project work, do not use `set_active_instance` or any global active-instance routing as a substitute for per-call `unity_instance`. If the needed typed tool/resource schema does not accept `unity_instance`, stop and report that MCP routing is blocked for concurrent Unity work; do not execute project-scoped Editor operations through that unrouted call.

## Core Rule

Use Unity MCP for Editor-owned state: scenes, prefabs, materials, ScriptableObjects, Addressables, imports, console, play mode, screenshots, and MenuItems. Use `$test-writing` for Unity test authoring and test execution workflow. Use local file tools for plain text only when Editor state is not involved.

Use plain asset, folder, and resource-path names. Do not create or rename files/folders with square brackets, quotes, punctuation, or other special symbols; use letters, digits, spaces, hyphens, and underscores. Rename Unity assets through Unity/AssetDatabase so `.meta` GUIDs are preserved.

For ad-hoc MCP/Editor scans, do not probe, load, or generate code against Roslyn APIs such as `Microsoft.CodeAnalysis`, `CSharpCompilation`, or `SyntaxTree`. Use Unity-owned information surfaces first: typed MCP tools/resources, `AssetDatabase`, `MonoScript`, `TypeCache`, reflection, Unity docs/reflection tools, and local text search for plain source text. Do not create temporary `.cs` files in `Assets/` just to inspect code or types.

At the start of Unity-specific work, expose Unity MCP tools with one `tool_search` query for the needed tool family. If the exact required direct `mcp__unityMCP.*` tool is still missing or failing, check the Unity process before saying the Editor is closed.

If Unity Editor is running and the MCP HTTP server answers at the configured URL (for Outcasts this is usually `http://127.0.0.1:8080/mcp`) but typed Unity tools still do not appear in Codex, the required fix is a full Codex Desktop restart with Unity and the MCP HTTP server left running. After restart, run `tool_search` again. Do not treat this as a wrong Unity address until the HTTP endpoint itself fails.

Do not use a direct Streamable HTTP MCP client to perform Unity project work. Direct HTTP is allowed only for diagnostics such as confirming `initialize` or `tools/list`; asset, scene, prefab, script, console, and Editor mutations must use typed `mcp__unityMCP.*` tools. If typed tools remain unavailable after restart, report Unity MCP as unavailable and stop for reconnect/fix.

Generic MCP resources are optional. Empty or unrouted `list_mcp_resources` is not a Unity MCP outage when direct Unity MCP tools are available.

## Minimal Workflow

1. Before reading project-scoped Unity resources or running project-scoped Unity tools, resolve the target Editor ID from `mcpforunity://instances`. Use the exact `Name@hash` value whose project root matches the current workspace as the routing token.
2. Pass that token explicitly as `unity_instance` on every project-scoped Unity MCP tool/resource call whose schema supports it. This includes validation, console, MenuItem, asset, scene, prefab, custom-tool, and batch calls.
3. For `batch_execute`, add `unity_instance` inside each nested command's `params`; do not rely on a top-level batch route or active global instance.
4. If a needed resource or tool cannot accept `unity_instance`, stop and report that Unity MCP routing is blocked for concurrent Unity work. Do not use active-instance state as a substitute for per-call routing.
5. Never call `mcp__unityMCP.set_active_instance` or `set_active_instance` as a workaround for missing `unity_instance` support. The correct fix is exposing/routing the typed tool schema with per-call instance support.
6. Inspect only the target area: scene/object/asset/script/package state needed for the task.
7. Modify through Unity MCP tools for Editor state; use `$apply-patch` for project text files.
8. After C# or `.asmdef` edits, run `$mcp-unity-validate-script`, then `$unity-recompile-menuitem`, then read console errors.
9. For visual scene/UI changes, verify with a bounded screenshot only when visual correctness matters.

## Modal Dialogs

Scene reloads or scene switches can open a Unity modal confirmation dialog that locks the Editor and blocks MCP progress. When MCP calls stop making progress after a scene reload/switch, assume a user-only dialog may be open; report that the user must click it manually and wait. Do not try to bypass it with shell automation, repeated MCP calls, or global instance switching.

## Script Rules

Before creating or substantially editing Unity C# scripts, load `$code-style`. Keep exactly one top-level C# type per `.cs` file. Do not add migration scripts, conversion tools, package-cache edits, or future-facing public API unless the user asks or existing code requires it.

Do not edit package source, embedded packages, `Packages/`, or `Library/PackageCache/`. Read packages or docs only when project code and reflected APIs are insufficient.

## Direct-Only MCP Tools

Call these directly, never through `batch_execute`:

- `mcp__unityMCP.find_in_file`
- `mcp__unityMCP.validate_script`

Use `batch_execute` only for tools known to route correctly, such as grouped scene/object/component operations.

## Payload Discipline

Keep discovery bounded:

- Prefer narrow paths, names, components, layers, and page sizes. Do not search, create, assign, or filter by Unity GameObject tags.
- Read the first page or summary first; follow `next_cursor` only when the task needs the complete set.
- Keep console reads focused on recent errors unless warnings are part of the task.
- Use screenshots at 256-512 px unless higher detail is necessary.
- Use `include_image=True` only when image analysis is needed.

## API Verification

Prefer this order when API details are uncertain:

1. Existing project code/assets.
2. Live reflection through `unity_reflect` when available.
3. Official Unity/package docs through `unity_docs` or package docs.
4. Large reference files in this skill, read by relevant section only.

Do not run docs/reflection lookups for routine Unity APIs already clear from nearby project code.

## References

Read only the relevant reference index, then open the one topic file it points to:

- `references/tools-reference.md`: index for exact MCP tool parameters and uncommon tool actions.
- `references/workflows.md`: index for extended workflow examples. For testing workflow, use `$test-writing` first and consult MCP references only for exact tool parameters.
