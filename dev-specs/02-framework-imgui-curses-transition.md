# 02 — Framework: stale mouse capture on ImGui→curses transitions

## Symptom
Many menus that switch from an ImGui screen (e.g. any uilist) to a plain
curses loop started with all mouse input dead (hover included) until a
registered keyboard action was pressed. Unregistered keys did not help,
because they are swallowed inside the blocking `input_context::handle_input()`
and never unblock the loop. Navigation keys unlock the mouse as a side effect
of causing a redraw.

## Root cause (verified with instrumented builds + log data)
1. The last ImGui frame drawn while the old window was alive left
   `WantCaptureMouse=true`.
2. The next screen's first `ui_manager::redraw()` runs ImGui NewFrame while
   the just-destroyed window is still in its one-frame `WasActive` grace, so
   hover still points at its ghost area → `WantCaptureMouse` stays true.
3. The curses loop then blocks in `handle_input()`. Gated mouse events are
   swallowed at the SDL layer and never insert input, the loop never
   iterates, and **no further ImGui frame ever runs** (frame counter frozen
   across hundreds of events in the logs). The stale capture is permanent
   until some keyboard action iterates the loop.

Instrumented evidence: `/tmp/opencode/mouse-diag4.log`, `mouse-diag5.log`,
`mouse-diag6.log`, Xvfb run `xvfb8/game.log` (autonomous repro; 5 hover moves
all swallowed at frozen `framecnt=40`; after one `Down` press the flag drops
at `frame=41`).

## Fix (commit 5cd9cb7, src/ui_manager.cpp)
When a `redraw_invalidated()` pass ends and the UI stack no longer contains
any ImGui adaptor but the previous pass did, run one extra empty ImGui frame
(`new_frame` + `end_frame`, the same prime-the-pump pattern already used at
startup in `select_initial_language`, main.cpp). At that point the dead
window's `WasActive` grace has expired, so hover recomputes to null and
`WantCaptureMouse` drops before the next loop starts blocking.

## Validation
- Reproduced the bug autonomously under Xvfb with xdotool (mouse-only path:
  main menu → world tab → world row → Manage world submenu → Show World
  Mods → the dialog), pre-fix: dead until `Down`; post-fix: instant response.
- Post-fix autonomous checks: hover immediately moves the list cursor; click
  on a row selects it (feature 01 behavior) and the dialog stays open;
  outside copy unaffected.

## Status
Merged into `mouse-worldfactory-click` (rebase) for combined in-game testing.
