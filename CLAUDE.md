# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

C++23 reimplementation of the 1997 Pikachu Volleyball using SDL3. The game logic is a port of
[gorisanson's JavaScript reverse-engineering](https://github.com/gorisanson/pikachu-volleyball) of the
original binary. Comments across the physics and controller code reference the original Ghidra symbols
(`FUN_00402dc0`, `physicsEngine`, ...) — **when changing physics or AI behavior, preserve fidelity to the
original algorithm**; the odd-looking constants and integer arithmetic are intentional, not bugs.
Deviations from the original are called out in comments (see `physics_common.hpp:31` for the style).

## Build

Requires CMake 3.25+, Ninja, and a C++23 compiler (GCC 13+ / Clang 16+; MinGW-w64 on Windows).
SDL3, SDL_mixer, and SDL_ttf are fetched from source via `FetchContent` and linked **statically** — the
first configure of a build dir is slow.

```bash
cmake --preset release           # then: cmake --build --preset release
cmake --preset debug             # -Wall -Wpedantic, defines PIKA_DEBUG (verbose SDL logging)
cmake --preset release-embedded  # assets compiled into the binary
cmake --install build/release    # binary + assets/ dir into build/release/pikaball-release
```

Windows presets are `windows-release` / `windows-release-embedded` (presets are gated on host OS).
Binary lands at `build/<preset>/pikaball_revamped`.

There are no tests and no lint config — CI (`.github/workflows/cmake-multi-platform.yml`) only builds
the matrix of {Linux gcc, Linux clang, Windows gcc} × {`PIKA_EMBED_ASSETS` OFF, ON}. Any change must
compile in **both** asset modes.

## Asset loading (two modes)

`include/pikaball/resources.hpp` is the single point of asset access; everything goes through
`pika::load_resource(filename)` where `filename` is the repo-relative path (e.g. `"assets/sounds/pi.wav"`)
and doubles as the resource key.

- `PIKA_EMBED_ASSETS=ON`: `cmake/pika_embed.cmake` (a trimmed fork of `battery::embed`) generates, at
  *configure* time, one `.cpp` per asset holding the bytes plus a `battery/pika_embed.hpp` with
  `pika::b::embed<"path">()`. Adding a new asset requires editing **three** places: the `pika_embed(...)`
  list in `src/CMakeLists.txt`, the `resource_list` array in `resources.hpp`, and its `_filename` constant.
- `PIKA_EMBED_ASSETS=OFF` (default): loads from `SDL_GetBasePath()/assets/...`, or from
  `PIKA_ASSETS_RUNTIME_DIR` when set at configure time (used for Flatpak, where the `assets/` prefix is
  stripped and replaced by the sandbox path).

## Architecture

SDL3 callback app (`SDL_MAIN_USE_CALLBACKS`): `main.cpp` owns a single `pika::Game` and drives
`Game::step()` from `SDL_AppIterate`, sleeping to hit the target frame time. `Game::run()` also exists as
a self-driven loop but is not the path used by `main.cpp`.

Layering (each library built separately in `src/*/CMakeLists.txt`, public headers in `include/pikaball/`,
private ones next to the sources in `src/`):

- **`physics`** (`src/physics/`, `include/pikaball/physics/`) — `Physics` owns two `Player`s and one
  `Ball` and advances the world one frame from two `PlayerInput`s. Pure logic, no SDL rendering. It is
  the only mutable game-world state.
- **`controller`** (`src/controller/`) — `PlayerController` base with `on_update(const PhysicsView&) ->
  PlayerInput`. `PhysicsView` is a read-only *copy* of the ball and both players, handed to controllers
  so they cannot mutate the world. `KeyboardController` just forwards the input struct the `Game` filled
  in; `ComputerController` is the ported original AI. New input sources (joystick, network) belong here
  as new `PlayerController` subclasses.
- **`view`** (`src/view/*.hpp`, header-only) — `View` base owns the renderer/sprite-sheet non-owning
  pointers and the black fade texture (`fade_in`/`fade_out`). Each subclass (`IntroView`, `MenuView`,
  `VolleyView`, `OptionsView`, `FPSView`) has `start()` to reset and a `render(frame_counter, ...)` called
  once per frame. Views are stateful animation drivers keyed off the frame counter, not pure draw calls.
- **`SDLSystem`** (`src/sdl_system.*`) — RAII owner of `SDL_Init`/`SDL_Quit`, window, renderer, font,
  sprite-sheet texture, and `PikaSound` (SDL_mixer wrapper with 4 channels: general / left pika / right
  pika / ball). Everything else takes non-owning pointers from it.

Game flow is a state machine in `Game`: outer `GameState` (Intro / Menu / VolleyGame) with sub-states
(`MenuState`, `VolleyGameState`) — all enums in `include/pikaball/game_state.hpp`. Each state has a
`*_state()` method in `game.cpp` that advances logic and calls the matching view's `render`.
`menu_options_state()` runs every frame regardless of state (Esc-pause overlay).

Input is event-driven, not polled per-frame: `handle_event` pushes `SDL_EVENT_KEY_DOWN` (non-repeat) onto
a mutex-guarded queue; `compile_events()` drains it at the top of `step()` into `MenuInput` /
`PlayerInput`. This is deliberate — power-hit and menu keys must fire once per press. Key bindings are
`constexpr` scancodes in `namespace pika::keys` at the top of `game.cpp`.

## Conventions

- Fixed 432×304 internal resolution (`common.hpp`); all sprite rects are hardcoded pixel coords in
  `include/pikaball/sprites.hpp`.
- Game speed is frame rate: 20/25/30 FPS for slow/medium/fast, plus a fixed 5 FPS slow-motion mode. There
  is no delta-time; physics steps once per frame.
- 2-space indent, `snake_case` members with trailing underscore, `PascalCase` types, `namespace pika`
  (with `pika::view`, `pika::sprite`, `pika::txt`, `pika::keys` sub-namespaces), `#ifndef PIKA_*_HPP`
  include guards, Doxygen `/** */` on public methods, copy/move explicitly deleted on owning classes.
- UI strings are Spanish literals in `pika::txt` (`common.hpp`); no localization layer.
- `README.md` / `README_ES.md` are kept in sync — update both when changing user-facing behavior.
