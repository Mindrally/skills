---
name: gamemaker-gml
description: "Best practices for GameMaker Language (GML) development, covering scripts, object events, rooms, data structures, and performance-minded game code. Use when writing GML scripts or object event code in GameMaker Studio, structuring create/step/draw events, managing rooms and instances, building state machines for gameplay, or optimizing GML for performance."
---

# GameMaker Language (GML) Development

This skill covers writing maintainable, performant game code in GameMaker Language (GML), including code organization, style conventions, gameplay architecture, and performance patterns specific to GameMaker Studio.

## Workflow for Building GML Gameplay Features

1. **Design the object** — Decide what Create, Step, Draw, Collision, and Alarm events the object needs, and what data it owns.
2. **Initialize in Create** — Set default variable values, instantiate data structures (`ds_map`, `ds_list`, structs), and cache references (layer IDs, other instance IDs) in the Create event.
3. **Simulate in Step** — Put input handling, physics/movement, state machine transitions, and collision response in Step (or Step-adjacent events like Begin/End Step).
4. **Render in Draw** — Keep Draw events limited to rendering; never mutate gameplay state there.
5. **Extract reusable logic into scripts** — Move any behavior used by more than one object, or any event body that's grown complex, into a named script/function.
6. **Clean up on Destroy/Room End** — Destroy any manually created data structures (`ds_list_destroy`, etc.) and free surfaces to avoid memory leaks.
7. **Profile before optimizing** — Use GameMaker's built-in profiler or manual timing (`get_timer()`) to find actual hot paths before restructuring for performance.

## Code Organization

- Keep object event code short; move any reusable behavior into scripts or named functions instead of duplicating logic across events or objects.
- Use clear, consistent prefixes or naming conventions for scripts (`scr_`), objects (`obj_`), sprites (`spr_`), rooms (`rm_`), and global variables/macros, matching whatever convention the project has already established.
- Prefer functions over copy-pasted event blocks — if the same five lines appear in two objects' Step events, it belongs in a script.
- Keep Create/Step/Draw responsibilities separate: initialization goes in Create, simulation and logic in Step, and rendering-only work in Draw (and Draw GUI for HUD/UI elements that shouldn't scale/move with the room camera).

## GML Style

- Use descriptive variable names (`player_speed`, `enemy_target`) and avoid single-letter names outside small, obviously-scoped loops (`for (var i = 0; i < count; i++)`).
- Prefer local variables declared with `var` (function/event-scoped) over unnecessary instance variables — instance variables should hold state that genuinely needs to persist or be accessed elsewhere.
- Use constants, enums (`enum PlayerState { IDLE, RUN, JUMP }`), and macros (`#macro`) for repeated identifiers, layer names, state IDs, and collision groups — never scatter raw numeric or string literals for meaningful values.
- Guard optional or possibly-stale instance references with `instance_exists(other_id)` before accessing their variables, since instance IDs can become invalid after the referenced instance is destroyed.
- Keep global state (`global.*`) minimal and documented; excessive globals make an object's actual dependencies invisible.

### Example: A State Machine with a Script and an Object

```gml
// scr_player_states.gml
function player_state_enter(_inst, _state) {
    _inst.state = _state;
    switch (_state) {
        case PlayerState.IDLE:
            _inst.sprite_index = spr_player_idle;
            break;
        case PlayerState.RUN:
            _inst.sprite_index = spr_player_run;
            break;
        case PlayerState.JUMP:
            _inst.sprite_index = spr_player_jump;
            _inst.vspeed = -_inst.jump_force;
            break;
    }
}

function player_state_step(_inst) {
    switch (_inst.state) {
        case PlayerState.IDLE:
            if (_inst.input_x != 0) {
                player_state_enter(_inst, PlayerState.RUN);
            } else if (!_inst.on_ground) {
                player_state_enter(_inst, PlayerState.JUMP);
            }
            break;

        case PlayerState.RUN:
            _inst.hspeed = _inst.input_x * _inst.move_speed;
            if (_inst.input_x == 0) {
                player_state_enter(_inst, PlayerState.IDLE);
            } else if (!_inst.on_ground) {
                player_state_enter(_inst, PlayerState.JUMP);
            }
            break;

        case PlayerState.JUMP:
            _inst.hspeed = _inst.input_x * _inst.move_speed;
            if (_inst.on_ground && _inst.vspeed >= 0) {
                player_state_enter(_inst, PlayerState.IDLE);
            }
            break;
    }
}
```

```gml
// obj_player - Create event
enum PlayerState { IDLE, RUN, JUMP }

move_speed   = 4;
jump_force   = 8;
input_x      = 0;
on_ground    = false;
state        = PlayerState.IDLE;

player_state_enter(id, PlayerState.IDLE);
```

```gml
// obj_player - Step event
input_x = (keyboard_check(vk_right) - keyboard_check(vk_left));
on_ground = place_meeting(x, y + 1, obj_ground);

player_state_step(id);

move_and_collide(hspeed, vspeed, obj_ground);
```

## Gameplay Architecture

- Use finite state machines (enums + switch, or a struct-based FSM) for player, enemy, UI, and game-flow states instead of a tangle of boolean flags.
- Keep collision logic explicit and deterministic — use `place_meeting`, `instance_place`, and collision events consistently rather than mixing multiple ad hoc distance checks for the same interaction.
- Separate input collection (reading `keyboard_check`/`gamepad_*` into intent variables like `input_x`) from action execution (applying that intent to movement/attacks), so input remapping or AI-driven control doesn't require touching gameplay logic.
- Use alarms, timelines, or explicit timer variables consistently within a project; mixing multiple timing patterns for the same kind of behavior makes debugging harder.
- Store save data through structured maps/structs (`{ level: 3, hp: 80, inventory: [...] }`) and version the save format (a `save_version` field) so future format changes can migrate old saves instead of breaking them.

## Performance

- Avoid expensive searches such as broad `instance_find`/`instance_number` loops or repeated full-room collision scans inside every Step event; scope searches with object type filters and spatial partitioning (grids, layers) where possible.
- Cache frequently used asset IDs, layer IDs, and instance references in Create rather than re-resolving them (e.g., `layer_get_id("Instances")`) every Step.
- Destroy data structures (`ds_list_destroy`, `ds_map_destroy`, `ds_grid_destroy`) when no longer needed — GML data structures are not garbage collected and leak until manually freed or the game restarts.
- Use object pooling (deactivate/reuse instances via `instance_deactivate_object`/reactivate, or a custom pool) for frequently spawned projectiles, particles, or short-lived effects once `instance_create_layer`/`instance_destroy` churn becomes a measurable cost.
- Profile before optimizing (GameMaker's built-in profiler, or `get_timer()` deltas around suspect code) and keep hot-path code simple — premature micro-optimization usually costs more in the code review than it saves in frame time.

## Debugging and Testing

- Add debug overlays (drawn conditionally behind a debug flag) for collision boxes, current state, velocity vectors, and AI decisions when tracking down gameplay bugs.
- Use assertions or explicit guard clauses (`if (!instance_exists(target)) return;`) for states that should be impossible, so bugs fail loudly during development instead of silently misbehaving.
- Test room transitions, pause/resume, save/load, and controller/keyboard input as separate concerns — each has its own failure modes (state not reset on room change, input still processed while paused, etc.).
- Keep reproducible test rooms for complex mechanics (a "test_boss_fight" room, a "test_platforming" room) so a specific interaction can be re-verified quickly after a change.

## Common Mistakes

- Putting game logic (state changes, physics, spawning) inside a Draw event, which can run at a different rate than Step and shouldn't mutate gameplay state.
- Creating data structures (`ds_list_create`, `ds_map_create`, surfaces) without ever destroying them, leaking memory over a play session.
- Relying on room-editor instance creation order for critical behavior — order isn't guaranteed to match visual/editor order at runtime; use explicit Create-event initialization or a controller object instead.
- Hardcoding magic numeric state IDs (`state = 2`) instead of named enum values (`state = PlayerState.JUMP`), which makes the code unreadable and error-prone to change.
