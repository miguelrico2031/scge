# Scriptable Card Game Engine

A small C++ engine for building card games from data and scripts. The project starts with a single player versus a random CPU, while keeping the rules and runtime structured so that more game types and multiplayer can be added incrementally.

This README is the living design note for the project. It records current decisions and the first implementation target; it should evolve as implementation reveals better choices.

## Goals

- Support different card-game styles, including poker-like games, Uno-like games, and simple combat games with attack, defense, and abilities.
- Let game scripts define legal actions and card behavior without requiring a new C++ implementation for every game.
- Keep the engine core independent of any presentation library.
- Begin with a local human-versus-random-CPU game and leave a path to networked multiplayer.
- Keep each step small enough to produce a playable, understandable result.

## Initial technology direction

- **C++** for the runtime and applications.
- **Lua** for game rules, legal-action generation, and reactions to events.
- **JSON** for game configuration and card definitions.
- **raylib** for a later 2D front end.
- A console front end first, so the core can be exercised before rendering work begins.
- ECS may be used internally, but the public game and script APIs should not depend on a particular ECS implementation.

Exact library versions and build tooling remain implementation decisions.

## Architecture

The project is divided into three logical modules:

### Core / server

The server is authoritative and owns the game state. It handles turn progression, action execution, event dispatch, script execution, and the information each participant is allowed to observe. C++ code is organized under a server namespace (for example, `scge::server`).

Scripts decide which actions are legal. The server asks the game script for the legal actions available to a participant, then accepts a submitted action only if it matches a currently legal action. The server remains responsible for applying the action and updating state.

### Client

Client code is organized under a client namespace (for example, `scge::client`). A client presents the information it is allowed to see, gathers a player choice, and sends a request to the server. It does not mutate authoritative state or implement game rules.

The console front end and future raylib front end are clients of the same core/server interface. The random CPU is also a participant that chooses randomly from the server-provided legal actions.

### Transport

Client requests and server responses pass through a transport interface. The first implementation uses an in-process transport that forwards data between client and server in the same process. This adds an explicit boundary now and gives the project a seam for a network transport later; it is not network code yet.

Requests and responses should be designed as serializable messages from the beginning. The in-process transport must not give the client direct access to server state or bypass server validation.

## Script and event model

The game script provides a single legal-action function for a participant and the current game view. It returns the actions that participant may take at that point. The CPU chooses from its returned list. The human interface uses the legal choices to enable or constrain input without presenting an internal action list as game UI.

The server controls events. It emits events such as `game_started`, `turn_started`, `card_drawn`, `card_played`, `player_passed`, and `turn_ended`. Lua scripts can subscribe to these events and call exposed operations to make changes. Scripts cannot directly emit arbitrary events or communicate directly with other card scripts. If an operation causes another event, the server performs the operation and emits that event.

The script API should expose game state through a participant-appropriate view, so hidden information such as another player's hand or an undealt deck is not accidentally revealed. The exact event list, action representation, and script API will be refined through the first playable example.

## Data model

JSON describes game and card data. A card definition can contain fields such as an ID, display name, texture asset path, and Lua script path. Game configuration can describe the selected game scripts, card sets or decks, and turn-related settings.

Lua contains behavior; JSON contains configuration and data. Texture paths are consumed by a graphical client, not by the core server. The server should not depend on raylib or other rendering types.

## First vertical slice

Build a small console game that demonstrates the full path through the architecture:

1. Load game configuration and card definitions from JSON.
2. Start a match with one human participant and one CPU participant.
3. Ask Lua for each participant's legal actions.
4. Send the human's chosen action through the in-process transport to the authoritative server.
5. Have the CPU choose randomly from its legal actions.
6. Execute actions and advance turns on the server.
7. Dispatch at least one server event that a Lua script handles.

The sample rules should stay intentionally small. The purpose is to prove the boundaries and scripting loop, not to implement poker, Uno, or a full trading-card game in the first iteration. Add the raylib client after the console client can play the sample game through the same client/server interface.

## Design constraints and open questions

- Scripts determine legal actions, but only the server executes them and changes authoritative state.
- Clients receive only participant-appropriate information and choices.
- Events originate from the server; scripts react through registered handlers and exposed operations.
- The core must remain independent of console and graphics code.
- ECS choice, action serialization format, detailed event ordering, and Lua binding approach are open until needed by the first slice.

## Incremental roadmap

1. Define the smallest game state, action message, legal-action script contract, event contract, and in-process transport.
2. Implement the console vertical slice and one event-driven card behavior.
3. Add a raylib client using the same transport and server API.
4. Use additional sample games, such as Uno-like and poker-like rules, to find missing general-purpose features.
5. Add a real transport only when multiplayer is in scope.
