# 01 — worldfactory: Active World Mods dialog click behavior fix

Branch: `mouse-worldfactory-click`
Warm-up feature: smallest real defect to validate the branch/build/test/commit loop.

## Symptom (before)
`worldfactory::show_active_world_mods()` (src/worldfactory.cpp:914) is a read-only
viewer dialog for a world's mod list. It registered `SELECT` but the unusual thing
is closing — any left-click anywhere, including inside the mod entries, executed
`break` (dialog closed, selection NOT applied). Hover already works (MOUSE_MOVE).

Entry point(s): Main menu → Load World → <world> → "Active World Mods"
(main_menu.cpp:1213); debug action ACTION_WORLD_MODS (handle_action.cpp:3046).

## Change
- SELECT now hits tests `ent_map` (built each redraw by `draw_mod_list`):
  - click inside a mod entry → moves cursor (selects) and stays open — same
    semantics as hover, plasma explicit.
  - click outside entries / empty list → dismiss (previous global behavior,
    kept so the dialog remains dismissable with the mouse alone).
- SELECT removed from the unconditional close list; right-click (SEC_SELECT)
  still dismisses, ESC (QUIT)/CONFIRM unchanged.

Pattern follows the established idiom (options.cpp:3929-3973; worldfactory
world-picker at worldfactory.cpp:645-660).

## Design notes
- `run_for_point_in` returns match count (0 = miss) — used as bool.
- No scroll (`recalc_start`) needed: clicked entry is tautologically onscreen.
- cursor index vocabulary unchanged (full-list index, consistent with
  navigate_ui_list and draw_mod_list).

## Verification
- [ ] Server-side compile: `devenv shell -- make RELEASE=1 TILES=1 SOUND=1` parse of
  worldfactory.cpp + relink; -Werror clean.
- [ ] In-game (user): 
  1. Main menu → Load World → choose world → Active World Mods.
  2. Click a mod row → highlight moves to it, dialog stays open.
  3. Click outside entries / on border → dialog closes.
  4. Right-click → closes. ESC → closes.
  5. Keyboard: up/down + CONFIRM still close the dialog; overmap of behavior intact.
- [ ] Regression sweep of worldgen tab-based mod screens (they share draw_mod_list,
  but no code path change): quick open + click a few rows.

## Files touched
- src/worldfactory.cpp (show_active_world_mods only)
