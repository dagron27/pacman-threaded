# Threaded Pac-Man Clone

![CI](https://github.com/dagron27/pacman-threaded/actions/workflows/ci.yml/badge.svg)

**Assignment:** `320-fa24-Final-Exam-main`.

## Overview

This is a local, single-player Pac-Man clone written in Python with Pygame, built as a vehicle for demonstrating concurrent programming (originally a final project for a college course, CS 320, Fall 2024). Each ghost runs as its own thread with an independent control loop, player input is handled on a dedicated thread, and the main thread runs the Pygame event loop and rendering. All shared mutable state (score, game-over/win flags, power-up status, move queue) lives in a single `SharedState` object guarded by one lock.

The entire live implementation is in [`logic/game_logic.py`](logic/game_logic.py):

- **One thread per ghost.** Each ghost is its own `Ghost(threading.Thread)` instance; movement decisions, collision checks, and speed timing happen inside that ghost's own thread, concurrently with the other ghost(s) and the player.
- **A dedicated input thread.** Player movement runs on `Game.handle_player_input`, separate from the ghost threads and from the main thread (which owns the Pygame event loop and `Game.render_frame`).
- **Shared state behind a lock.** `SharedState.lock` guards every read/write of score, game-over/win flags, and power-up status from `Ghost.calculate_move`, `Ghost.handle_collision`, and `Game.handle_player_input`, to prevent races between "ghost eats player," "player eats ghost," and concurrent score updates.
- **Producer/consumer via `Queue`.** Keyboard input is captured on the main thread and handed off through `SharedState.move_queue` (a thread-safe `queue.Queue`) so the input thread never touches Pygame's event system directly.
- **Daemon threads with bounded shutdown.** Ghost threads are `daemon = True` (the player thread is not); both are joined with a 1-second timeout in `Game.cleanup`, so the process exits cleanly even if a thread is mid-loop when the game ends.

**Gameplay:** Pac-Man collects dots and energizers (power pellets); two ghosts (`chase` and `random` personalities) pursue the player; eating an energizer lets Pac-Man eat ghosts temporarily; eaten ghosts walk back to their start position (`Ghost.calculate_move`, `logic/game_logic.py:48-59`) and respawn once they arrive, rather than on a fixed timer; the game ends on board-clear (win) or player-caught (game over).

## Dependencies

Listed in [`requirements.txt`](requirements.txt):

```
pygame
```

- Python 3.8+
- OS: cross-platform (Pygame supports Windows/macOS/Linux); developed/tested on Windows
- Requires a display/desktop environment — Pygame opens a window, there is no headless mode

## Environment Setup

```bash
pip install -r requirements.txt
python main.py
```

Entry point is [`main.py`](main.py), which imports and runs `logic.game_logic.Game`:

```python
from logic.game_logic import Game

if __name__ == "__main__":
    game = Game()
    game.start()
```

**Controls:** Arrow keys to move, `Esc` to quit.

## Project Structure

```
main.py                    Entry point - runs logic.game_logic.Game
logic/
  game_logic.py             Live implementation: Position, SharedState, Ghost (thread), Game
                             (board layout, movement, collisions, rendering, main loop)
  movement_logic.py         Dead code - see Known Issues
  __init__.py                Empty package marker
config.py                   Dead code - see Known Issues
entities.py                 Dead code - see Known Issues
display/
  game_display.py           Dead code - see Known Issues
```

## Known Issues

### Dead Code (verified against current imports)

Verification method: every `import`/`from` statement in the repository was enumerated. The only project-local import anywhere is `main.py:1` (`from logic.game_logic import Game`). Nothing imports `config.py`, `entities.py`, `display/game_display.py`, or `logic/movement_logic.py` — confirming these four modules are unreachable from the running program. They are leftovers from refactoring: at some point their logic was re-implemented inline inside `logic/game_logic.py`, and the original standalone modules were never deleted.

| File | Dead code | Duplicated by (live) | Fix-it plan |
|---|---|---|---|
| [`config.py:1-2`](config.py) | `BOARD_WIDTH = 28`, `BOARD_HEIGHT = 36` | `logic/game_logic.py:155-156` hardcodes its own `BOARD_WIDTH = 28`, `BOARD_HEIGHT = 31` inside `Game.__init__` instead of importing this file. Note the two disagree (36 vs. 31 rows). | Have `Game.__init__` import and use these constants instead of hardcoding its own; reconcile the row-count mismatch before wiring it up. |
| [`entities.py:1-68`](entities.py) | `Player`, `Ghost`, `Dot`, `Energizer`, `Fruit` classes | `logic/game_logic.py` defines its own `Position` dataclass (`:11-20`) and `Ghost` class (`:34-150`) inline instead of importing these. | Move the canonical entity definitions into `entities.py` and have `game_logic.py` import them, removing the duplicate inline classes. |
| [`display/game_display.py:3-21`](display/game_display.py) | `render_board()` | `Game.render_frame()` in `logic/game_logic.py:265-314` duplicates the same drawing logic inline instead of calling it. | Import `render_board` (or move `render_frame`'s logic into this module) so rendering has one source of truth. |
| [`logic/movement_logic.py:1-14`](logic/movement_logic.py) | `move_entity()` | `Ghost.calculate_move` (`logic/game_logic.py:48-113`) and `Game.handle_player_input` (`:220-263`) each re-implement the same bounds/collision check inline via `Game.is_valid_move` (`:215-218`). | Import and reuse `move_entity()` (or `is_valid_move`) from a single place instead of duplicating the check in three call sites. |

Note: `config.py`'s `BOARD_HEIGHT = 36` and `display/game_display.py`'s hardcoded screen size (`720px / 20px cells = 36 rows`, `game_display.py:5`) agree with each other — they appear to reflect an earlier board layout that was later shrunk to 31 rows in `logic/game_logic.py:156`, without the other files being updated or removed.

If this project is picked up again, the fix in each case is to consolidate — pick one implementation, delete the other — rather than maintaining two versions of the same logic side by side.

**Other TODOs for a future pass**, beyond the dead-code consolidation above:
- Add automated tests around collision/movement logic (`Game.is_valid_move`, `Position.collides_with`) — currently untested.
- Replace magic numbers (e.g., ghost-eaten score `200`, power-up duration `150`) with named constants.

### Security

No security findings. This is a local, single-player Pygame application with no networking, no file or database access beyond a hardcoded in-memory board layout, no subprocess/`eval`/`exec` usage, no credential handling, and no external data ingestion — there is essentially no attack surface. This was confirmed in an independent security review prior to this documentation pass.

## Status

Academic / archived. This was built as a final project for a college course and is a completed class demo — it is not actively maintained or intended for production use. It is kept here as a learning artifact and as a reference for the concurrency patterns described above. The dead-code items above are documented for anyone who forks or continues the project, not as active bugs being tracked.

## Contributors

- Ben Tran
- Daniel Leone
- Justin Halvorson

This was a three-person group project; see `LICENSE` for how that affects
licensing. Based on the commit history, each person's main areas of work
were roughly:

- **Ben Tran:** Wrote the initial working Pygame implementation in a single
  commit (`main.py`, `logic/game_logic.py`, `display/game_display.py`,
  `logic/movement_logic.py`, and edits to `entities.py`); authored and
  maintained `README.md` through most of its early history.
- **Justin Halvorson:** Authored the original `entities.py` entity classes
  and `config.py`; reworked the board layout to be more accurate; moved
  player input handling onto its own thread; fixed player/ghost collision
  handling that had regressed.
- **Daniel Leone:** Added `.gitignore` and `requirements.txt`; reworked
  `Game.cleanup`'s thread shutdown/joining logic in `logic/game_logic.py` to
  stop the game from freezing on close; made a small gameplay tweak (power-up
  pellet color); and wrote the current `README.md` documentation pass (dead
  code audit, project structure, known issues, and security sections) as
  well as this QA sweep and the `LICENSE` scoping.
