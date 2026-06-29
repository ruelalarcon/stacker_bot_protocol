# Stacker Bot Protocol 1

Status: Stable

Version label: `SBP/1`

Default transport: newline-delimited JSON over standard input and standard
output

## 1. Purpose

Stacker Bot Protocol (SBP) connects a block-stacking game client to a bot
engine. The client sends authoritative game state to the bot. The bot responds
with suggested placements and optional runtime information.

SBP is designed around this ownership model:

- The client owns physical truth: board contents, active piece, queue, hold
  state, lock results, sequence numbers, and game lifecycle.
- The bot owns decision-making: search state, evaluation, derived strategy
  state, and preference ordering among legal or expected moves.
- A bot suggestion is advice. The client may reject a suggestion if it is
  illegal, unreachable, stale, or undesirable for the client environment.

The protocol is not tied to a particular runner, game implementation, or bot.
It can be implemented by local processes, web workers, sockets, or any other
message transport that can carry JSON objects.

## 2. Terminology

Client:

The program that owns the game state and asks for bot suggestions. A client may
be a game, replay tool, visualizer, local runner, training harness, or wrapper
around another engine.

Bot:

The program that receives state and returns suggested moves. A bot may be a
standalone executable, library worker, service, or network process.

Piece:

A controllable falling unit. Piece identifiers are strings such as `"T"`,
`"I"`, `"P"`, `"X"`, `"big_T"`, or `"custom:hook"`. SBP/1 does not reserve
piece identifiers. A client and bot may assign custom meaning to identifiers
outside the standard tetromino set.

Mino:

An occupied unit cell belonging to a piece or board block. The protocol name
uses "stacker" because the intended domain is grid-based block stacking, not
only a specific tetromino game.

Placement:

The final locked location requested or suggested for a piece.

Snapshot:

A full authoritative position sent by the client. A snapshot is the recovery
point for desyncs.

## 3. Conformance

A conforming SBP/1 bot must:

- Send `register` as its first message after startup.
- Correctly parse every required SBP/1 client-to-bot message.
- Ignore unknown message types.
- Ignore unknown object members on known messages.
- Either send `ready` after supported `rules`, or send `error` with
  `reason: "unsupported_rules"`.
- Respond to `suggest` with `suggestion` while a game is active.
- Honor every capability it advertises in `register`.
- Accept `stop` and `quit`.

A conforming SBP/1 client must:

- Wait for `register` before sending `rules`.
- Validate capabilities before relying on optional behavior.
- Send `rules` before `start`.
- Send `start` before `suggest` or `advance`.
- Use optional behavior only when the bot advertises compatible capabilities.
- Treat bot suggestions as advisory and validate them against authoritative
  state.
- Ignore unknown message types and unknown object members.

Implementations should be liberal in what they accept where doing so does not
hide protocol errors. Implementations should not crash on malformed, unknown,
or extension messages from the other side.

## 4. Transport

### 4.1 JSON Lines

The default SBP/1 transport is one JSON object per line. Each message is encoded
as UTF-8 JSON followed by `\n`.

Example:

```json
{"type":"suggest"}
```

Pretty-printed JSON is not valid on the JSON Lines transport because a message
must occupy exactly one line.

### 4.2 Other Transports

Other transports may carry the same JSON message objects. For example:

- a Web Worker may use `postMessage`
- a WebSocket may send one JSON object per text frame
- a library API may pass decoded objects directly

Transports may define additional framing, timeout, or process-lifecycle rules,
but they must not change the message object semantics.

## 5. Message Format

Every message is a JSON object with a string `type` member.

```json
{"type":"ready"}
```

Unknown message types must be ignored. Unknown members on known message types
must be ignored.

Unless otherwise specified:

- integers are signed JSON numbers without a fractional component
- booleans are JSON `true` or `false`
- strings are Unicode strings
- absent optional fields use the documented default
- `null` is only valid where explicitly documented

## 6. Coordinates and Board Model

SBP/1 uses integer grid coordinates.

- `x = 0` is the leftmost board column.
- `y = 0` is the bottom board row.
- `x` increases to the right.
- `y` increases upward.
- Board row arrays are ordered from bottom to top.

The default board is 10 columns by 40 rows. Clients may send a `board_size`
rule to use another size when the bot advertises compatible board-size
capabilities.

Board cells are represented as:

- `null` for empty
- a string for an occupied cell

The string may be the piece identifier that produced the cell, `"G"` for
garbage, or another implementation-defined cell label. SBP/1 treats all
non-null cells as occupied. Clients and bots must not infer geometry from board
cell labels unless an extension explicitly defines that behavior.

Example bottom rows:

```json
[
  ["G", "G", "G", "G", "G", "G", "G", "G", "G", "G"],
  [null, null, null, null, null, null, null, null, null, null]
]
```

## 7. Piece Identifiers and Geometry

SBP/1 has a canonical piece geometry convention. Implementations may use any
internal coordinate system, rotation origin, or piece representation, but SBP
messages must be translated into the geometry convention defined here.

Piece identifiers are strings. SBP/1 does not restrict them to tetrominoes.
This allows clients and bots to use pentominoes, trominoes, custom pieces, or
variant piece sets without changing the core message shape.

Common tetromino identifiers are `"I"`, `"J"`, `"L"`, `"O"`, `"S"`, `"T"`,
and `"Z"`. They are examples, not the complete protocol vocabulary. Other
identifiers are valid when the client and bot use the same piece catalog.

A piece definition is a mapping from orientation strings to occupied mino cells
relative to a placement anchor. A placement location contains `piece`,
`orientation`, `x`, and `y`. For each relative cell `[dx, dy]` in that piece
orientation, the absolute board cell is:

```text
(x + dx, y + dy)
```

The standard orientation strings are `north`, `east`, `south`, and `west`.
For the standard tetromino examples below, the other orientations are produced
from the `north` cells by rotating each relative cell around the anchor:

| Orientation | Relative-cell transform from `north` |
|---|---|
| `north` | `(x, y)` |
| `east` | `(y, -x)` |
| `south` | `(-x, -y)` |
| `west` | `(-y, x)` |

Example SRS-like tetromino north-facing relative cells:

| Piece | `north` relative cells |
|---|---|
| `"I"` | `[[-1, 0], [0, 0], [1, 0], [2, 0]]` |
| `"J"` | `[[-1, 0], [0, 0], [1, 0], [-1, 1]]` |
| `"L"` | `[[-1, 0], [0, 0], [1, 0], [1, 1]]` |
| `"O"` | `[[0, 0], [1, 0], [0, 1], [1, 1]]` |
| `"S"` | `[[-1, 0], [0, 0], [0, 1], [1, 1]]` |
| `"T"` | `[[-1, 0], [0, 0], [1, 0], [0, 1]]` |
| `"Z"` | `[[-1, 1], [0, 1], [0, 0], [1, 0]]` |

These tetromino cells are examples of the SBP geometry convention. Piece
identifiers remain arbitrary strings, so a piece catalog may define additional
pieces with any number of minos. Those pieces must still use the same SBP
anchor, relative-cell, board-coordinate, and orientation-string conventions in
SBP messages.

SBP/1 core messages do not transmit the full piece catalog. Implementers are
responsible for ensuring that both sides use the same piece definitions and
that their communication follows this SBP geometry standard.

Pieces spawn at `(x=4, y=20, orientation="north")` unless the client sends a
custom `spawn_position` rule and the bot advertises spawn-position support.
Spawn coordinates are interpreted within the configured board size.

## 8. Lifecycle

A typical process lifecycle is:

1. Bot sends `register`.
2. Client sends `rules`.
3. Bot sends `ready`, or `error`.
4. Client sends `start`.
5. Client sends `suggest`.
6. Bot sends `suggestion`.
7. Client validates and accepts a placement for execution.
8. Client sends `advance`.
9. Client sends zero or more `new_piece` messages for newly visible generated
   pieces.
10. Client may send `board` when the physical board changes authoritatively
   without changing active piece, queue, hold, rules, or session lifecycle.
11. Client repeats from step 5.
12. Client sends `stop` when the game, analysis session, or current position is
   no longer active. This closes the current bot-side session.
13. Client may send `start` for another game after `stop`; the new session must
   be treated as fresh and independent from the stopped session.
14. Client sends `quit` before process shutdown.

The client may also recover from desync by sending `stop`, then `start` with a
fresh authoritative snapshot. A transport or implementation may instead restart
the bot process; these flows are semantically equivalent for session state. If
only the physical board changed and the bot supports mid-session board
replacement, the client should prefer `board` over `stop` plus `start`.

## 9. Client-to-Bot Messages

### 9.1 `rules`

Sent by the client after `register`. It describes the rule environment the bot
is expected to support.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"rules"` | | Message discriminator. |
| `randomizer` | no | string | | Randomizer name, for example `"seven_bag"`. |
| `kickset` | no | string | | Rotation kick profile name, for example `"srs"`. |
| `rot180` | no | boolean | `true` | Whether 180-degree rotation input is part of the movement rules. |
| `sonic_drop` | no | string | | Sonic-drop support mode. Common values are `"none"`, `"allow"`, and `"only"`. |
| `spin_detection` | no | string | `"t-spins"` | Spin detection mode. |
| `back_to_back_sources` | no | array of strings | `["quad","t-spin","t-spin-mini"]` | Clear classes that maintain or advance back-to-back. Order has no meaning. |
| `spawn_position` | no | object | `{ "x": 4, "y": 20 }` | Spawn coordinate. |
| `board_size` | no | object | `{ "width": 10, "height": 40 }` | Board dimensions. |

`spin_detection` values:

| Value | Meaning |
|---|---|
| `none` | No locks are classified as spins. |
| `t-spins` | T pieces use the T/corner rule. |
| `t-spins+` | T/corner rule, plus immobile T locks count as mini. |
| `all` | T/corner rule; immobile non-T locks count as full. |
| `all+` | T/corner rule, immobile T fallback as mini, immobile non-T locks as full. |
| `all-mini` | T/corner rule; immobile non-T locks count as mini. |
| `all-mini+` | T/corner rule, immobile T fallback as mini, immobile non-T locks as mini. |
| `mini-only` | T/corner spins and immobile all-piece spins count as mini. |

Spin detection uses these common primitives:

- T/corner rule: the piece must be grounded, the lock must come from a rotation,
  and at least three T corner cells must be occupied. Front-corner occupancy
  determines mini versus full, except rule-specific kicked-rotation upgrades
  may classify the spin as full. The kick details are implementation-internal
  and are not part of SBP messages.
- Immobility rule: the locked piece cannot move left, right, or up. Down is
  implicit because the piece is already locking at its lowest reachable row.

`back_to_back_sources` values:

| Value | Meaning |
|---|---|
| `quad` | Four-line clears. |
| `t-spin` | Full T-spin clears. |
| `t-spin-mini` | Mini T-spin clears. |
| `allspin` | Full non-T spin clears. T pieces are excluded. |
| `allspin-mini` | Mini non-T spin clears. T pieces are excluded. |
| `perfect-clear` | perfect-clears. |

`spin_detection` controls how a lock's `spin` is classified. `back_to_back_sources`
controls which clear classes maintain or advance back-to-back. Duplicate
`back_to_back_sources` values should be rejected or normalized by receivers.

Example:

```json
{
  "type": "rules",
  "randomizer": "seven_bag",
  "kickset": "srs",
  "rot180": true,
  "sonic_drop": "only",
  "spin_detection": "t-spins",
  "back_to_back_sources": ["quad", "t-spin", "t-spin-mini"],
  "spawn_position": { "x": 4, "y": 20 },
  "board_size": { "width": 10, "height": 40 }
}
```

If the bot supports these rules, it sends `ready`. If not, it sends `error`.

### 9.2 `start`

Starts analysis from an authoritative snapshot. This message must be sent before
`suggest` or `advance`. A `start` after `stop` begins a new session and must not
reuse hidden game, analysis, speculation, queue, or extension state from the
previous session.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"start"` | | Message discriminator. |
| `board` | yes | board array | | Authoritative board cells. |
| `active` | yes | piece identifier | | Active piece at its spawn location. |
| `queue` | yes | array of piece identifiers | | Upcoming queue, excluding `active`. |
| `hold` | yes | piece identifier or `null` | | Held piece, or `null` if hold is empty. |
| `combo` | yes | integer | | Number of consecutive line clears immediately before this snapshot. |
| `back_to_back` | yes | integer or boolean | | Current back-to-back count or status. |
| `piece_stream` | no | piece stream object | | Observed generated-piece history. |
| `incoming_garbage` | no | array of positive integers or `null` | `null` | Pending incoming garbage chunks. |
| `extensions` | no | object | | Extension-specific snapshot data. |

Example:

```json
{
  "type": "start",
  "board": [[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null]],
  "active": "T",
  "queue": ["I", "O", "L", "J", "S"],
  "hold": null,
  "combo": 0,
  "back_to_back": 0,
  "incoming_garbage": []
}
```

The `board` array must match the active board-size rules:
`board_size.height` rows and `board_size.width` cells per row.

`incoming_garbage` is optional. If omitted or `null`, incoming garbage
information was not provided or is unknown. If present, it must be an array of
positive integers ordered from earliest-resolving chunk to latest-resolving
chunk. An empty array means incoming garbage is known and currently empty.
Clients should preserve separate chunks when the selected mode's garbage rules
make chunk boundaries observable or strategically relevant. In modes where chunk
boundaries have no gameplay effect, clients may aggregate pending garbage into a
single chunk containing the total line count. Bots may also aggregate the array
when their strategy only needs the total pending garbage.

### 9.3 `board`

Replaces the bot's current physical board inside the active session. This
message is valid after `start` and before `stop`, and should only be sent when
the bot advertises support for mid-session board replacement.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"board"` | | Message discriminator. |
| `board` | yes | board array | | Authoritative physical board cells. |

Example (abbreviated; real messages must include `board_size.height` rows):

```text
{"type":"board","board":[[null,null,null,"G",null,null,null,null,null,null], "..."]}
```

The `board` message changes only physical board truth. It must not modify the
active piece, queue, hold state, combo, back-to-back, piece stream, rules,
extensions, or incoming garbage. It is intended for garbage rise, manual board
edits, and authoritative board correction after the client knows the physical
board has changed. Bots may discard or rebuild board-dependent search trees,
caches, and evaluation state when receiving it.

Clients should fall back to `stop` plus `start` with a fresh authoritative
snapshot when the bot does not support mid-session board replacement, or when
any non-board session state must also change.

`board` is distinct from `incoming_garbage`: `incoming_garbage` on `start` or
`suggest` communicates pending known garbage chunks that are not necessarily on
the board yet; `board` communicates physical board truth after garbage, edits,
or other corrections have actually changed board cells.

### 9.4 `suggest`

Requests suggestions for the current active snapshot.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"suggest"` | | Message discriminator. |
| `incoming_garbage` | no | array of positive integers or `null` | `null` | Pending incoming garbage chunks. |
| `extensions` | no | object | | Extension-specific request data. |

Example:

```json
{
  "type": "suggest",
  "incoming_garbage": [4, 2]
}
```

Bots should respond quickly, even if the result is incomplete. Bots may send
runtime `info` messages before or after `suggestion`.

### 9.5 `advance`

Advances the bot's active session by a placement the client has accepted as the
intended next lock. This allows the bot to preserve search state when no full
reconciliation is needed. The client remains authoritative: if later observed
state differs from this intended advancement, the client must correct with
`board` when only physical board cells changed, or restart from `start` when
non-board state changed.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"advance"` | | Message discriminator. |
| `move` | yes | placement object | | Accepted placement used to advance the session. |

Example:

```json
{
  "type": "advance",
  "move": {
    "location": {"piece": "T", "orientation": "south", "x": 4, "y": 1},
    "spin": "full"
  }
}
```

If the bot cannot reconcile `advance` with its internal state, it should continue
to tolerate later messages and may send an `info` warning. The client remains
authoritative and can send a fresh `start`.

### 9.6 `new_piece`

Informs the bot that a newly generated piece became known to the client.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"new_piece"` | | Message discriminator. |
| `piece` | yes | piece identifier | | Newly generated piece. |

Example:

```json
{"type":"new_piece","piece":"Z"}
```

Clients may send multiple `new_piece` messages after `advance` if more than one
piece entered the visible queue.

### 9.7 `stop`

Stops the current game or analysis session.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"stop"` | | Message discriminator. |
| `reason` | no | string | | Stop reason. |

Example:

```json
{"type":"stop","reason":"game_over"}
```

The bot should stop computation for the current session and discard session
state derived from `start`, `board`, `suggest`, `advance`, `new_piece`, and runtime
extension messages. The process may remain alive, but a later `start` must behave like a
fresh session rather than a continuation of the stopped one. Process-level
configuration from earlier `rules` messages may remain in effect.

### 9.8 `quit`

Requests process shutdown.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"quit"` | | Message discriminator. |

Example:

```json
{"type":"quit"}
```

Bots should exit promptly after receiving `quit`.

## 10. Bot-to-Client Messages

### 10.1 `register`

The first message sent by a bot after startup. It identifies the bot and
advertises capabilities.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"register"` | | Message discriminator. |
| `name` | yes | string | | Bot name. |
| `version` | no | string | | Bot version. |
| `author` | no | string | | Bot author. |
| `capabilities` | no | object | | Bot capability advertisement. |

Capability fields:

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `randomizers` | no | array of strings | | Supported randomizer names. |
| `kicksets` | no | array of strings | | Supported kickset names. |
| `rot180` | no | boolean | | Whether 180-degree rotation is supported. |
| `sonic_drop` | no | array of strings | | Supported sonic-drop modes. |
| `spin_detection` | no | array of strings | | Supported `spin_detection` values. |
| `back_to_back_sources` | no | array of strings | | Supported back-to-back source atoms. Any requested subset is supported. |
| `piece_stream` | no | boolean | | Whether `start.piece_stream` is supported. |
| `spawn_position` | no | boolean | | Whether custom `rules.spawn_position` coordinates are supported. |
| `board` | no | boolean | `false` | Whether mid-session `board` replacement is supported. |
| `board_size` | no | object | | Supported board dimensions. If omitted, only 10 by 40 is supported. |

`board_size` contains `width` and `height`. Each value may be either an integer
for one supported value, or an object with integer `min` and `max` fields for an
inclusive supported range:

```json
{
  "board_size": {
    "width": 10,
    "height": { "min": 1, "max": 64 }
  }
}
```

Example:

```json
{
  "type": "register",
  "name": "ExampleBot",
  "version": "1.0.0",
  "author": "Example Author",
  "capabilities": {
    "randomizers": ["seven_bag"],
    "kicksets": ["srs"],
    "rot180": true,
    "sonic_drop": ["none", "allow", "only"],
    "spin_detection": ["none", "t-spins", "t-spins+", "all", "all+", "all-mini", "all-mini+", "mini-only"],
    "back_to_back_sources": ["quad", "t-spin", "t-spin-mini", "allspin", "allspin-mini", "perfect-clear"],
    "piece_stream": true,
    "spawn_position": true,
    "board": true,
    "board_size": {
      "width": 10,
      "height": { "min": 1, "max": 64 }
    }
  }
}
```

Capabilities are advisory until rule negotiation completes. A client should use
them to avoid sending clearly unsupported rules, but the bot makes the final
decision by sending `ready` or `error`.
When provided, requested `spin_detection` must be one of
`capabilities.spin_detection`, and requested `back_to_back_sources` must be a subset of
`capabilities.back_to_back_sources`.
When `capabilities.board_size` is omitted, clients should only request the
legacy 10 by 40 board size.

### 10.2 `ready`

Indicates that the bot accepts the most recent `rules` message.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"ready"` | | Message discriminator. |

Example:

```json
{"type":"ready"}
```

### 10.3 `error`

Reports a protocol or rule error.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"error"` | | Message discriminator. |
| `reason` | yes | string | | Machine-readable error reason. |

Standard reasons:

- `unsupported_rules`
- `internal_error`

Example:

```json
{
  "type": "error",
  "reason": "unsupported_rules"
}
```

Errors do not automatically close the process.

### 10.4 `info`

Optional runtime information. May be sent at any time after `register`.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"info"` | | Message discriminator. |
| `topic` | yes | string | | Runtime information topic. |
| `data` | yes | any JSON value | | Topic-specific payload. |

Common topics:

- `status`: human-readable status
- `search`: search statistics
- `warning`: warning useful to print by default
- `debug`: verbose diagnostic data

Example:

```json
{
  "type": "info",
  "topic": "search",
  "data": {
    "nodes": 1312451,
    "nps": 568375,
    "depth": 8
  }
}
```

Clients may filter, display, log, or ignore `info` messages.

### 10.5 `suggestion`

Responds to `suggest`.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `type` | yes | `"suggestion"` | | Message discriminator. |
| `moves` | yes | array of suggestion move objects | | Suggested moves in descending preference order. |

Example:

```json
{
  "type": "suggestion",
  "moves": [
    {
      "location": {"piece": "T", "orientation": "south", "x": 4, "y": 1},
      "spin": "full"
    }
  ]
}
```

If `moves` is empty, the client should treat the response as no available
suggestion.

## 11. Shared Data Types

### 11.1 Piece Identifier

A piece identifier is a string. Empty strings should not be used.

Examples:

```json
"T"
```

```json
"P"
```

```json
"custom:wide_l"
```

Piece identifiers are looked up in the piece catalog shared by the client and
bot. The same identifier must refer to the same SBP-format piece definition
throughout a session.

### 11.2 Piece Location

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `piece` | yes | piece identifier | | Piece at this location. |
| `orientation` | yes | string | | Rotation/orientation name. |
| `x` | yes | integer | | X coordinate. |
| `y` | yes | integer | | Y coordinate. |

For compatibility with older Tetris-bot formats, consumers may accept `type` as
an alias for `piece`. New SBP/1 messages should emit `piece`.

Example:

```json
{"piece":"I","orientation":"east","x":7,"y":2}
```

`x` and `y` are always the SBP placement anchor coordinates. For every relative
cell `[dx, dy]` in the selected piece orientation, the occupied board cell is
`(x + dx, y + dy)`. Implementations with a different internal origin or
rotation system must translate to this representation before sending SBP
messages and translate from it after receiving SBP messages.

### 11.3 Placement

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `location` | yes | piece location | | Final locked piece location. |
| `spin` | yes | string | | Common values are `"none"`, `"mini"`, and `"full"`. |

Example:

```json
{
  "location": {"piece": "T", "orientation": "south", "x": 4, "y": 1},
  "spin": "full"
}
```

### 11.4 Piece Stream

Piece stream data provides observed generated-piece history without exposing
hidden randomizer state.

| Field | Required | Type | Default | Description |
|---|---:|---|---|---|
| `pieces` | yes | array of piece identifiers | | Observed generated pieces. |
| `offset` | yes | integer or `null` | | Generated-piece index of `pieces[0]`, or `null` when absolute alignment is unknown. |

`offset` is the generated-piece index of `pieces[0]`. If `offset` is `null`,
the client observed the order but does not assert absolute alignment.

The stream should end with the currently visible generated pieces: the active
piece followed by the visible queue.

Example:

```json
{
  "offset": 7,
  "pieces": ["J", "S", "T", "Z", "I", "O", "L", "T", "I", "O"]
}
```

For strict seven-bag randomizers, an aligned stream allows a bot to infer bag
boundaries. Generated piece index `i` belongs to bag `floor(i / 7)`.

### 11.5 Board

A board is an array of row arrays ordered from bottom to top.

The board must contain `board_size.height` rows. Each row must contain exactly
`board_size.width` cells. When no board-size rules are sent, this means 40 rows
with 10 cells each.

Example empty 10-wide bottom row:

```json
[[null, null, null, null, null, null, null, null, null, null]]
```

Example occupied cells:

```json
[
  ["G", "G", "G", "G", "G", "G", "G", "G", null, null],
  ["I", "I", "I", "I", null, null, null, null, null, null]
]
```

## 12. Rule Profiles and Extensions

SBP/1 core intentionally keeps game-specific mechanics small. Complex or
nonstandard games should define extensions.

Extensions may define additional state that is meaningful to a specific client,
mode, or bot family, for example:

- opponent board snapshots or versus attack state
- known garbage RNG seeds, hole forecasts, or garbage-source metadata
- mode-specific meters and state such as Zone charge, combo tables, or garbage blocking
- timing and physics details such as gravity, lock delay, or handling constraints
- client-side reachability, input, or pathing constraints
- replay, analysis, training, or debugging metadata

### 12.1 Runtime Extensions

Use `extensions` for namespaced data with protocol or gameplay meaning.
Runtime messages such as `start` and `suggest` may include an `extensions`
object for extension-scoped state or request data.

### 12.2 Extension Names

Extension names should be stable strings. Reverse-DNS or URL-like prefixes are
reasonable for private experiments, for example:

```json
{
  "type": "suggest",
  "extensions": {
    "org.example.mode_state.v1": {
      "zone_meter": 42
    }
  }
}
```

Bots that do not understand a runtime extension may ignore it.

## 13. Desync and Recovery

Because the client is authoritative, desync recovery is straightforward:

1. The client detects that a bot placement is stale, illegal, or based on old
   state.
2. The client ignores that suggestion.
3. If only the physical board changed and the bot supports mid-session board
   replacement, the client sends `board`; otherwise the client sends `stop`,
   then `start` with the current snapshot, or restarts the bot process.

Bots should not assume that `advance` and `new_piece` are the only ways state can
change. `board` replaces physical board contents inside the current session,
while `stop` closes the current bot-side session and a new `start` always
replaces any previous bot-side session state.

## 14. Compatibility Notes

SBP is conceptually related to earlier Tetris bot protocols, but it is a
standalone protocol. The major design differences are:

- The client sends authoritative snapshots.
- Suggestions are advisory.
- Piece identifiers are arbitrary strings.
- Rule support is negotiated through capabilities and `rules`.
- Runtime information is sent through independent `info` messages.

## 15. Complete Example

Bot:

```json
{"type":"register","name":"ExampleBot","version":"1.0.0","capabilities":{"randomizers":["seven_bag"],"kicksets":["srs"],"rot180":true,"sonic_drop":["only"],"spin_detection":["t-spins"],"back_to_back_sources":["quad","t-spin","t-spin-mini"],"piece_stream":true,"spawn_position":true,"board":true,"board_size":{"width":10,"height":{"min":1,"max":64}}}}
```

Client:

```json
{"type":"rules","randomizer":"seven_bag","kickset":"srs","rot180":true,"sonic_drop":"only","spin_detection":"t-spins","back_to_back_sources":["quad","t-spin","t-spin-mini"],"spawn_position":{"x":4,"y":20},"board_size":{"width":10,"height":40}}
```

Bot:

```json
{"type":"ready"}
```

Client:

```json
{"type":"start","board":[[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null],[null,null,null,null,null,null,null,null,null,null]],"active":"T","queue":["I","O","L","J","S"],"hold":null,"combo":0,"back_to_back":0}
```

Client:

```json
{"type":"suggest"}
```

Bot:

```json
{"type":"suggestion","moves":[{"location":{"piece":"T","orientation":"south","x":4,"y":1},"spin":"none"}]}
```
