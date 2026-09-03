---
name: blender-python-addon
description: "Best practices for writing Blender Python add-ons using the bpy API, covering operators, panels, properties, registration, and API-safe scripting. Use when creating a Blender add-on, defining bpy.types.Operator or Panel classes, registering PropertyGroup settings, writing register()/unregister() functions, working with bmesh, or debugging add-ons in Blender's Python console."
---

# Blender Python Add-on Development

This skill covers building Blender add-ons with the `bpy` API, including add-on structure, operators, panels, properties, safe scene manipulation, and testing across Blender versions.

## Workflow for Building a Blender Add-on

1. **Scaffold the add-on** — Create a package with `__init__.py` as the entry point, and a `blender_manifest.toml` (Blender 4.2+ extensions) or `bl_info` dict (legacy add-ons) describing name, version, and supported Blender version.
2. **Define data** — Create `PropertyGroup` classes for grouped settings and register them on the appropriate ID type (e.g., `bpy.types.Scene.my_addon = PointerProperty(type=MyAddonSettings)`).
3. **Implement operators** — Subclass `bpy.types.Operator` for each user action; implement `poll()` for availability, `invoke()` for interactive setup, and `execute()` for the actual work.
4. **Build UI panels** — Subclass `bpy.types.Panel` to expose operators and properties in the appropriate editor (3D viewport sidebar, properties editor, etc.).
5. **Register everything** — List all classes in a `classes` tuple and register/unregister them in matching `register()`/`unregister()` functions; register property pointers alongside their classes.
6. **Test in a clean profile** — Launch Blender with `--factory-startup`, install the add-on, enable it, exercise each operator, then disable it and confirm nothing is left behind.
7. **Package and ship** — Zip the add-on folder (or build a `.zip` extension package) and verify it installs cleanly via Blender's Preferences > Add-ons (or Extensions) panel.

## Add-on Structure

- Keep the add-on's entry point in `__init__.py` with clear `register()` and `unregister()` functions that mirror each other exactly (everything registered must be unregistered, in reverse order).
- Group operators, panels, properties, preferences, and utility code into separate modules once the add-on grows past a trivial size; import and register them from `__init__.py`.
- Use `bl_info` (pre-4.2 legacy add-ons) or `blender_manifest.toml` (4.2+ extensions) matching the target Blender version and packaging model — don't mix conventions.
- Keep UI labels concise and use `bpy.app.translations` or the `"Category"`/label conventions so user-facing text can be localized where the project supports it.

## API Usage

- Use `bpy.types.Operator` for actions, `bpy.types.Panel` for UI layout, and `bpy.types.PropertyGroup` for grouped, related settings.
- Define `bl_idname` (lowercase, `category.action` format, e.g. `object.apply_custom_modifier`), `bl_label`, and `bl_options` (e.g., `{'REGISTER', 'UNDO'}`) explicitly on every operator.
- Validate context in `poll()` before allowing an operator to run — check for an active object, correct mode, or valid selection so the operator button greys out instead of erroring.
- Use `invoke()` for interactive setup (showing a dialog, reading mouse position) and `execute()` for the actual operation; `invoke()` should call `self.execute(context)` when it doesn't need extra interaction.
- Return `{'FINISHED'}` on success or `{'CANCELLED'}` on failure/user-abort consistently — never return a bare `None` or an unrecognized string.
- Use `bpy.context.evaluated_depsgraph_get()` and `object.evaluated_get(depsgraph)` when reading final scene state that includes modifiers, shape keys, or other dependency-graph-driven results.

### Example: Operator, Panel, and Registration

```python
bl_info = {
    "name": "Random Vertex Color",
    "author": "Example",
    "version": (1, 0, 0),
    "blender": (4, 0, 0),
    "category": "Mesh",
}

import bpy
import random


class MESH_OT_random_vertex_color(bpy.types.Operator):
    """Assign a random color to the active vertex color layer"""

    bl_idname = "mesh.random_vertex_color"
    bl_label = "Randomize Vertex Colors"
    bl_options = {"REGISTER", "UNDO"}

    @classmethod
    def poll(cls, context):
        obj = context.active_object
        return (
            obj is not None
            and obj.type == "MESH"
            and obj.mode == "OBJECT"
            and obj.data.color_attributes.active_color is not None
        )

    def execute(self, context):
        obj = context.active_object
        color_layer = obj.data.color_attributes.active_color
        for data in color_layer.data:
            data.color = (random.random(), random.random(), random.random(), 1.0)
        obj.data.update()
        self.report({"INFO"}, f"Randomized colors on '{obj.name}'")
        return {"FINISHED"}


class VIEW3D_PT_random_vertex_color(bpy.types.Panel):
    bl_label = "Vertex Color Tools"
    bl_idname = "VIEW3D_PT_random_vertex_color"
    bl_space_type = "VIEW_3D"
    bl_region_type = "UI"
    bl_category = "Tool"

    def draw(self, context):
        layout = self.layout
        layout.operator(MESH_OT_random_vertex_color.bl_idname, icon="COLOR")


classes = (
    MESH_OT_random_vertex_color,
    VIEW3D_PT_random_vertex_color,
)


def register():
    for cls in classes:
        bpy.utils.register_class(cls)


def unregister():
    for cls in reversed(classes):
        bpy.utils.unregister_class(cls)


if __name__ == "__main__":
    register()
```

## Data and Properties

- Register custom properties through `PropertyGroup` classes instead of stuffing loose global state into module-level variables, which don't survive file reload and aren't undo-safe.
- Store add-on preferences (API keys, default paths, UI toggles that persist across files) in an `AddonPreferences` subclass registered with `bl_idname` matching the add-on's module name.
- Use `PointerProperty`, `CollectionProperty`, and fully-typed properties (`StringProperty`, `FloatProperty`, `BoolProperty`, etc.) with explicit `name=` and `description=` so tooltips and the Python API are self-documenting.
- Clean up custom properties, `bpy.app.handlers` entries, timers, and keymaps during `unregister()` — anything added to a `bpy.types.*` class or a handler list must be explicitly removed.

## Safety and Performance

- Never run destructive scene operations (deleting objects, overwriting files) without explicit user action — no silent auto-execution on load for anything destructive.
- Avoid blocking the UI thread in modal operators; use `context.window_manager.event_timer_add()` with a modal state machine for long-running operations instead of a tight loop.
- Batch mesh edits and use the `bmesh` module (`bmesh.from_edit_mesh`, `bmesh.new()`) when programmatically editing mesh data instead of looping over `mesh.vertices` one at a time for structural changes.
- Avoid repeatedly scanning large scenes or recomputing expensive data inside `draw()` methods — `draw()` runs on every UI redraw, so cache results and invalidate them only when the underlying data changes.
- Keep file paths configurable and resolve them with Blender path utilities (`bpy.path.abspath`, `bpy.utils.resource_path`) instead of hardcoding OS-specific paths.

## Testing and Debugging

- Test in both a clean Blender profile (`blender --factory-startup`) to catch hidden dependencies on other add-ons, and in a representative production scene to catch performance and data-shape issues.
- Add smoke tests that import the add-on module, call `register()`, run each core operator via `bpy.ops`, then call `unregister()` cleanly with no errors or leftover state.
- Use `self.report({'ERROR'}, "message")` or `{'WARNING'}`/`{'INFO'}` for user-facing operator feedback instead of printing to the console, which most users never see.
- Keep version-specific API differences (e.g., API changes between Blender 3.x and 4.x) isolated behind small helper functions so the rest of the add-on doesn't need version checks scattered throughout.

## Common Mistakes

- Forgetting to unregister classes, handlers, timers, and keymaps in `unregister()`, leaving Blender in a broken state after disabling the add-on.
- Mutating Blender data (adding objects, changing mesh data) from inside a `Panel.draw()` method — `draw()` must only read data and lay out UI.
- Assuming an active object, a non-empty selection, or a specific mode (Object/Edit/Sculpt) exists without checking `context` first.
- Hardcoding absolute asset paths that only exist on the developer's machine instead of using relative paths or `bpy.utils.resource_path`.
