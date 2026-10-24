---
name: uitoolkit-window-generation
description: Generate Unity UI Toolkit windows, UXML, USS, custom VisualElement widgets, and UIDocument controllers using the project structure demonstrated by Assets/Scripts/UI/UIToolkitTestWidgets. Use for new UI Toolkit screens or window prototypes.
---

# UI Toolkit Window Generation


## Reference Pattern

Use `Assets/Scripts/UI/UIToolkitTestWidgets` as a structural example only:

- `Widgets/`: reusable `[UxmlElement]` custom `VisualElement` or `Button` types.
- `Screens/`: screen UXML and a `MonoBehaviour` controller with `UIDocument`.
- `Styles/`: one USS file for shared window/widget styling.

The code style in that sample is not a recommendation. Follow `$code-style`, the target folder conventions, and current production UI code instead. The sample also contains mojibake in some string literals/UXML text; never copy that text or use it as localization source.

## Workflow

1. Decide whether the requested UI is a reusable widget, a screen/window, or both.
2. For C# files, load `$code-style` before choosing names, paths, type layout, or implementation.
3. Keep one top-level C# type per `.cs` file. Name files after the type.
4. Put reusable UI Toolkit elements in a `Widgets` folder and mark them with `[UxmlElement]` when they must be usable from UXML.
5. Put the screen UXML and controller in `Screens`; put shared style rules in `Styles`.
6. Use stable UXML `name` values for controller queries and callbacks.
7. In controllers, cache the `UIDocument.rootVisualElement`, subscribe in `Awake`/initialization, and unsubscribe in `OnDestroy`.
8. Write UXML/USS/C# through `$apply-patch`; after C# changes, run `$mcp-unity-validate-script` and `$unity-recompile-menuitem`.

## UI Rules

- Do not hard-code user-facing Cyrillic in UXML/C# for production windows; use localization or clear temporary placeholders until localization is added.
- Assign every created UI Toolkit GameObject to the Unity `UI` layer, including UIDocument hosts, prefab roots, and generated child objects. Set the layer explicitly in Editor scripts/MenuItems.
- Do not copy mojibake strings from `UIToolkitTestWidgets`.
- Use USS classes for visual styling; keep C# focused on structure, state, and behavior.
- Prefer stable dimensions and predictable layout constraints for windows, toolbars, buttons, counters, and repeated rows.
- Use UI Toolkit built-in elements and project custom widgets before adding new abstractions.
- Keep interaction state explicit through classes such as `selected`, `disabled`, `state-ok`, or similarly scoped names.

## Verification

After generation, verify that UXML references existing C# element types and USS paths, then let Unity compile/import. For visible UI work, use Unity/MCP or Browser screenshots when available to check that the window renders, fits, and callbacks do not throw console errors.
