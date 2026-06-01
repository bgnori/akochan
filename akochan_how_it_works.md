# Akochan: How it works

This document explains the runtime structure of Akochan from code.

## 1. High-level architecture

Akochan consists of two layers:

- **Mahjong engine layer** (game state, legal actions, progression)
  - Main files:   `mjai_manager.cpp`, `/tmp/workspace/bgnori/akochan/share/make_move.cpp`
- **Decision layer (AI)**
  - Main files:   `ai_src/selector.cpp`, `/tmp/workspace/bgnori/akochan/ai_src/tactics.cpp`

`system.exe` coordinates these layers via command modes in `/tmp/workspace/bgnori/akochan/main.cpp`.

## 2. Startup and command modes

`main.cpp` dispatches by `argv[1]`. Representative modes:

- `test`: run self-play games and output logs
- `mjai_client`: connect to mjai server via TCP and respond in real time
- `pipe` / `pipe_detailed`: stdin/stdout integration for external processes
- `mjai_log` / `full_analyze`: offline analysis for an existing game log
- `game_server`: progress one step with request validation and return new moves

All modes eventually call shared logic for:

1. reading or updating `game_record` (JSON move history)
2. deciding actions via AI (`ai` / `ai_review`)
3. applying legal progression rules

## 3. Core data model

The central data is **`Moves game_record`** (JSON sequence of actions).

- Each action contains `type`, `actor`, `pai`, etc.
- `game_record` is converted to `Game_State` when evaluation is needed.
- This design allows:
  - online mode (append events from server),
  - offline mode (load from file),
  - simulation mode (generate events internally)

## 4. Game progression flow

In `mjai_manager.cpp`:

- `game_loop(...)` repeats until `end_game`.
- `proceed_game(...)` advances one step depending on the last action type:
  - start game / next round setup
  - tsumo/dahai/kakan branch
  - reach/chi/pon follow-up discard handling
  - end of hand transition
- Legality is checked before applying requested actions (`is_legal_single_move` etc.).

So the engine is deterministic for a given `game_record` + request.

## 5. AI decision flow

Main entry points are in `ai_src/selector.cpp`:

- `ai(...)`: returns best move sequence
- `calc_moves_score(...)`: returns candidates with expected score
- `ai_review(...)`: returns candidates with detailed review info

`ai(...)` creates `Selector` and calls `Selector::set_selector(...)`.

Inside `set_selector(...)`, the rough flow is:

1. Build current `Game_State` from `game_record`
2. Analyze hand state (`Tehai_Analyzer`)
3. Estimate opponent tenpai probabilities (`Tenpai_Estimator_Simple`)
4. Compute risk/value tables (houjuu probability/value, tsumo/ron han-fu probabilities)
5. Compute expected placement-point outcomes (`kyoku_end_pt_exp`, `ryuukyoku_pt_exp`)
6. Generate candidate actions (discard/reach/fuuro/pass/hora)
7. Score candidates by expected value and sort descending

`ai(...)` returns the top candidate; if no action is needed, it returns `none`.

## 6. Tactics configuration

Tactics are loaded from JSON:

- multi-player setup: `set_tactics(...)`
- single profile for all players: `set_tactics_one(...)`

Default mjai configuration is `setup_mjai.json`.
It includes parameters like placement utility (`jun_pt`) and estimation model switches.

## 7. MJAI online integration

In `mjai_client` mode (`main.cpp` + `mjai_client.cpp`):

1. Connect to TCP server (`TcpClient`)
2. Receive mjai events line by line
3. Append events to `MJAI_Interface::game_record`
4. When actionable timing is reached (`tsumo` of self, or opponent discard with reactions),
   call `get_best_move(...)` -> `ai(...)`
5. Send chosen response JSON back to server

This is event-driven and stateful through the accumulated `game_record`.

## 8. Offline analysis

- `mjai_log`: loads a log and runs `ai_review(...)` at target state
- `full_analyze`: runs `calc_moves_score(...)` over all applicable points in a log
- `pipe_detailed`: emits review-enriched output for each decision timing

These modes are useful for debugging and strategy comparison.

## 9. Build artifacts and runtime binaries

On Linux:

1. Build AI shared library in `ai_src` -> `libai.so`
2. Build root executable -> `system.exe` (links against `libai.so`)

This separation reflects architecture: reusable AI core + command-mode host executable.
