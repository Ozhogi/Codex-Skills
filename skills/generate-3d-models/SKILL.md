---
name: generate-3d-models
description: Generate 3D model assets in Unity by assembling scene objects from primitives, materials, transforms, and hierarchy, then export the result to FBX with Unity FBX Exporter. Use when Codex needs to create kitbashed or blockout models, procedural primitive models, temporary model scenes, or exported `.fbx` assets; includes checking or installing the `com.unity.formats.fbx` package when needed.
---

# Generate 3D Models

## Core Rule

Use Unity Editor/MCP for all scene, object, material, package, and export operations. Do not hand-edit `.unity`, `.prefab`, `.asset`, `.mat`, `.fbx`, `.meta`, or `Packages/manifest.json` unless Unity MCP is unavailable, the user is told about the weaker fallback, and the user explicitly approves it.

Load and follow `$unity-mcp-orchestrator` before performing Editor work. If creating or changing C# Editor scripts, also load `$code-style`, `$mcp-unity-validate-script`, and `$unity-recompile-menuitem`.

## Workflow

1. Gather the model brief: object type, scale, intended output path, style constraints, required moving parts, pivot/origin expectations, and whether the source Unity scene/prefab should be kept after export.
2. Resolve the target Unity instance through `mcpforunity://instances` and pass its exact `Name@hash` as `unity_instance` on every project-scoped Unity MCP call that supports it.
3. Check for Unity FBX Exporter package `com.unity.formats.fbx` with Unity Package Manager/MCP or an Editor query. If missing, install it through Unity Package Manager via MCP/Editor APIs. Do not edit package source or `Library/PackageCache`.
4. Create or open a temporary authoring scene for the model. Use a clearly named root GameObject at world origin, with child primitives grouped by functional part.
5. Assemble the model from Unity primitives (`Cube`, `Sphere`, `Capsule`, `Cylinder`, `Plane`, `Quad`) using transforms, hierarchy, and materials. Prefer simple, inspectable geometry over hidden procedural mesh code unless the user asks for custom mesh generation.
6. Add materials through Unity APIs and assign readable names. Use colors and roughness/metallic values that support the requested model; avoid relying on scene lighting to communicate shape.
7. Normalize the result before export: root at `(0, 0, 0)`, intentional rotation, sane scale, no unrelated scene objects selected, clean child names, and obvious pivots for expected use.
8. Export only the root object or selected hierarchy to `.fbx` with Unity FBX Exporter APIs/menu integration. Prefer `ModelExporter.ExportObject` or the package's documented selection export path when available in the installed version.
9. Verify the FBX exists, has nonzero size, and Unity imports it without console errors. If visual correctness matters, instantiate or inspect the imported model in a bounded scene view screenshot.
10. Delete temporary generation scripts and temporary scenes unless the user explicitly asked to keep reusable tooling or source authoring assets.

## Package Notes

The package id is `com.unity.formats.fbx`. Install through the Editor Package Manager when needed. If package installation requires network access, registry credentials, or a Unity modal dialog, stop and report the specific blocker.

Before writing an Editor helper script for export, search existing project code for FBX exporter usage. If no helper exists, create a small editor-only script under `Assets/Editor/` or invoke an existing MCP custom tool. Remove one-off helper scripts after successful export and recompilation.

## Export Script Shape

When an Editor script is necessary, keep it narrow:

```csharp
using UnityEditor;
using UnityEngine;
#if UNITY_EDITOR
using UnityEditor.Formats.Fbx.Exporter;
#endif

public static class ExportSelectedModelToFbx
{
    private const string MenuPath = "MCP/Export Selected Model To FBX";

    [MenuItem(MenuPath)]
    public static void Export()
    {
        var root = Selection.activeGameObject;
        if (root == null)
        {
            Debug.LogError("Select a model root before exporting FBX.");
            return;
        }

        ModelExporter.ExportObject("Assets/Exports/GeneratedModel.fbx", root);
        AssetDatabase.SaveAssets();
        AssetDatabase.Refresh();
    }
}
```

Adapt paths and selection rules to the user request. Validate the actual API against the installed package version or nearby project code before committing to a script.

## Verification

Report:

- FBX path and any kept source scene/prefab path.
- Whether `com.unity.formats.fbx` was already installed or installed during the task.
- Console errors or warnings relevant to export/import.
- Any limitations of primitive-built geometry, such as approximate curves or non-deforming parts.
