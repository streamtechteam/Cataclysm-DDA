# Mouse Support Work Catalog — CDDA (this fork, master d29db9c)

Status: enumeration deliverable. Evidence legend:
- [V] = claimed by audit agents AND spot-verified by reading code directly
- [A] = agent-audited with file:line evidence, not yet hand-verified (mus verification when its feature branch starts)
- file:line references all point at master d29db9c.

## 0. Architecture (what mouse support means here)

- Two UI planes coexist:
  1. **Legacy curses plane** — hand-drawn `catacurses::window` layouts driven by
     `input_context::handle_input()`. Mouse events already arrive normalized as
     actions (`"MOUSE_MOVE"`, `"SELECT"` on click) with coordinates via
     `input_context::get_coordinates(_text)` (input_context.h:380,432). Menus that
     support mouse here do: register `SELECT`/`MOUSE_MOVE`, build a
     `std::unordered_map<int, inclusive_rectangle<point>>` rect map during redraw
     (`run_for_point_in`-based hit-testing, e.g. main_menu.cpp:728-764).
  2. **ImGui plane** (`cataimgui::window`, cata_imgui.h) — mouse handled natively by
     Dear ImGui; game actions created via `action_button()`/`get_button_action()`
     (cata_imgui.h:217, cata_imgui.cpp:1129). Runtime toggle: `cata::options::mouse` +
     `ImGuiConfigFlags_NoMouse` (sdltiles.cpp:386).
- **`uilist` is ImGui-only** (uilist.h:553, uilist.cpp:890-906): every uilist-based
  menu already has click=select+activate (uilist.cpp:135-151, 1031-1044) and
  hover=preview. Terminal builds bridge mouse to ImGui via ImTui (cata_imgui.cpp:186-221).
  ⫦ Consequence: the "big shared layer" we hypothesized is ALREADY shared and done;
  our work is the curse-plane islands + ImGui dialogs with hand-drawn interaction.

## 1. Menus that already work (no action needed)

| System | Evidence |
|---|---|
| All uilist menus (crafting list, skills, traits lists, mission lists, etc.) | uilist.cpp:135-151,1031-1044 [A] |
| Main menu, options UI, worldgen wizard, inventory selector family (+drag), look/peek overlay, overmap screen, ranged targeting, veh_shape/editmap | main_menu.cpp:728ff; inventory_ui.cpp:2490-2581+; game.cpp:6050-6314; overmap_ui.cpp:2224-2254; ranged.cpp:3140-3180 [A] |
| NPC talk, query_popup, surroundings, scores, follower rules, study zones, MA details, char creator, end screen, debug console, item info window | dialogue_imgui.cpp:465-473; popup.cpp:70-74; surroundings_menu.cpp:866-880; scores_ui.cpp:237-273; npctalk_rules.cpp; study_zone_ui.cpp; martialarts.cpp:2479-2530; newcharacter.cpp:1090-1095; ui_iteminfo.cpp:93-95 [A] |
| Construction *placement preview* (hover) | construction.cpp:1405-1421 [V] |

## 2. Work targets — curse-plane menus with NO mouse [V=spot-verified, else A]

### SMALL — same idiom (register MOUSE actions + rect map during redraw)
| Menu | File:entry | Note |
|---|---|---|
| Messages log panel | messages.cpp message log window | read-only scroll; needs wheel+drag scroll |
| Faction camp worker/job assignment | faction_camp.cpp:2095-2270 [A] | two list loops |
| Vending machine UI | iexamine.cpp:1092-1150 [A] | cursor-on-list selection |
| Color picker | color.cpp [A] | grid hit-testing |

### MEDIUM
| Menu | File:entry | Note |
|---|---|---|
| Construction recipe menu (hand-rolled window; NOT uilist) | construction.cpp:522ff [V: uilist.h used only at 73/2625 for partial-con prompt] | two-pane category+entry list; high player impact |
| Bionics UI | bionics_ui.cpp [A] | installed/available panes |
| Zone manager + create-zone dlg | zone_manager_ui* [A] | |
| Auto-pickup / safemode rule editors | auto_pickup.cpp, safemode_ui.cpp [A] | shared pattern, do both in one feature |
| Armor layering UI | armor_layers.cpp [A] | priority list reordering by drag? |
| Mutation menu (verify: partially uilist-based already?) | mutation_ui.cpp / mutation.cpp:1618 [A⫦] | confirm uilist coverage first |
| Load-world/Active-mods click bug — click anywhere = dialog closes (SELECT breaks loop, cursor not applied) | worldfactory.cpp:940-987 [V] | also confirm show-load-world dialog at 947-980 is hover-only |
| Advanced inventory "Y" pane screen | advanced_inv.cpp:2008-2106 [A] | mechanical but two-pane layout |
| Computer terminal session menus | computer_session.cpp [A] | |
| Diary / proficiency / bodygraph / smart_controller | [A] | batch later, low player impact |

### LARGE
| Menu | File | Note |
|---|---|---|
| Vehicle designer (veh_interact) | veh_interact.cpp [A] | 11 cooperating sub-windows; roll finding stop |

## 3. Completed-ish ImGui dialogs — identified GAPS [A]

| Gap | File:line |
|---|---|
| faction_ui: talk/rename/radio keyboard-only (click selects list only) | faction_ui.cpp:177-238 |
| Medical UI: APPLY/TREAT keyboard-only | medical_ui.cpp:142-187 |
| Mission UI: DELETE_POI keyboard-only | mission_ui.cpp:126,256-259 |
| Crafting GUI: TOGGLE_FAVORITE / HIDE_SHOW / RELATED / COMPARE / PRIORITIZE not clickable | crafting_gui.cpp footer, process_action 2515-2541 |
| Keybindings: `show` mode rows not clickable | input_context.cpp:781-784,1147-1151 |
| Compare item menu: parallel-column wheel sync TODO | game_inventory.cpp:2689-2698 |
| Item-info pane in inventory action menu is tiles-rendered, keyboard-scroll only | game.cpp:2109+/2124-243 |
| Help window: click-empty-area = select-anything quirk | help.cpp:150-158 |

## 4. Proposed sequence (for approval)

1. **Warm-up bugfix branch** `mouse-factory-worlds-click`: fix Load-World & Active-mods
   dialogs in worldfactory.cpp so click selects (rect hit-test) instead of closing.
   Tiny; validates our branch→build→test→commit loop end-to-end.
2. **First menu feature**: `mouse-construction-menu` — recipe menu mouse
   (hover preview + click select + secondary-click activate), MEDIUM; idiomatic
   curse-plane work, high player impact.
3. Grupo subsequent per approved catalog order: bionics → ImGui action buttons
   (faction/medical/mission/craft GUI) → armor — then re-assess.

Regex: ich idioma repeats in N pubs...", No shared-code refactor until 3 curse-plane menus
converge on the identical idiom (per user rule 5); then propose a small helper
(probably extending existing `run_for_point_in` usage pattern), never unilateral.

## 5. Verification protocol (per feature)

- `devenv shell -- make -j$(nproc) RELEASE=1 TILES=1 SOUND=1` (release build, -
  Werror enforced by Makefile)
- User runs `./cataclysm-tiles`, clicks through the target menu on their save; I
  fix reported anomalies. Keyboard bindings regression-check in same pass.
- Update dev-specs/ content if implementation diverges; commit spec with code.
