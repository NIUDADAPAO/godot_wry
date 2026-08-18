# Godot WRY → Godot 4.7.1 Migration Notes

> Migration notes for upstream (doceazedo/godot_wry). Explains what changed and the pitfalls hit during adaptation.

## Overview

- **gdext**: `0.2.4 (api-4-1)` → `0.5.5 (api-4-7)`, pinned to commit `637cef7`
- **Goal**: run natively on Godot 4.7.1 (no API-version warnings)
- **Trade-off**: `api-4-7` builds only run on Godot 4.7+ (GDExtension rule: higher-version bindings cannot load on lower-version engines)

## 1. Dependency & gdextension config

`rust/Cargo.toml` — bump gdext to api-4-7, pinned to a fixed commit:

```toml
godot = { git = "https://github.com/godot-rust/gdext", rev = "637cef73172bba23850131acd8583b0c72ebf0c7", features = ["api-4-7"] }
```

`godot/addons/godot_wry/WRY.gdextension` — only `compatibility_minimum` changes (`4.1` → `4.7`); `entry_symbol`, `reloadable`, `[libraries]`, `[icons]`, and `[dependencies]` all stay unchanged:

```ini
[configuration]
entry_symbol = "gdext_rust_init"
compatibility_minimum = 4.7   # was 4.1
reloadable = true

[libraries]
windows.release.x86_64 = "bin/x86_64-pc-windows-msvc/godot_wry.dll"
# ... linux / macos entries unchanged
```

## 2. API migrations (gdext 0.2.x → 0.5.x, 6 mechanical changes)

| Old | New | Note |
|---|---|---|
| `Dictionary` | `VarDictionary` | 0.5.x generics |
| `get_tree().expect(...)` | `get_tree_or_null().expect(...)` | returns `Option` |
| `get_root().expect(...)` | `get_root()` | returns non-null `Gd` |
| `"headless".into()` | `GString::from("headless")` | disambiguate |
| `enum::from_ord(n)` | `enum::try_from_ord(n)` | returns `Option` |
| — | `use godot::obj::Singleton;` | for `DisplayServer` / `Input` singletons |

All covered by gdext's official migration guide; no logic changes.

## 3. Two hidden runtime pitfalls (compile fine, break at runtime)

### Pitfall 1: calling `self.base*()` inside `&mut self` methods panics

- **Symptom**: debugger prints `bind_mut() failed, already bound; 1 = <Class>`
- **Cause**: gdext 0.5.5 has strict storage borrow checking (0.2.4 doesn't). Calling `self.base()` multiple times inside `ready()` / `input()` / `process()` / `enter_tree()` panics.
- **Fix**: cache `let base_gd = self.base().clone();` at the top and use `base_gd` everywhere. Note: `Callable::from_object_method` takes `&Gd<T>` → pass `&base_gd` (not `&*base_gd`, which derefs to `&T` → E0308); `connect` consumes the `Gd` → use `base_gd.clone().connect()`.

### Pitfall 2: `Viewport.push_input` does not update global `Input` state

- **Symptom**: scripts relying on `Input.is_mouse_button_pressed()` (e.g. mouse-drag to rotate/pan) stop working, while scripts using the event param (e.g. wheel via `event.button_index`) keep working. Both go through `push_input` with correct field values — easy to misdiagnose.
- **Cause**: `Viewport.push_input()` only dispatches the event; it does NOT update the global `Input` singleton state.
- **Fix**: use `Input::singleton().parse_input_event(&event)` (updates global state AND dispatches). Wheel branches can keep `push_input`.
- **Note**: gdext 0.2.4's `push_input` (fptr_by_index) can "accidentally work" on newer Godot due to method-index drift; 0.5.5 fixes the index, exposing this bug — suspect this first when "old dll works, new dll doesn't".

## 4. Verification checklist

1. **headless**: class registration + instantiation
2. **GUI**: left / right / middle mouse buttons, wheel, keyboard
3. **CI**: Windows / Linux / macOS all pass (Linux needs `libgtk-3-dev` etc., already in upstream CI)
