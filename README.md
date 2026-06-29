# Stacker Bot Protocol

Stacker Bot Protocol (SBP) is a JSON message protocol for connecting
block-stacking game clients to bot engines.

The client owns the authoritative game state. The bot receives rules and
snapshots, then returns suggested placements. Rule negotiation and capability
advertisement let clients and bots agree on supported randomizers, movement
rules, board sizes, optional snapshot fields, and session update behavior.
The protocol is intended for Tetris-like games, but it does not reserve piece
names for tetrominoes. Piece identifiers are strings, so clients and bots can
use nonstandard pieces without changing the core message shape. SBP has a
canonical piece geometry convention: each side must translate its internal
piece representation to the same identifiers, anchor coordinates, relative
cells, orientation names, spawn behavior, kicks, and lock rules.

The current stable version is:

- [SBP/1 specification](spec/sbp-1.md)

## Examples

- [Basic stdio session](examples/basic-session.jsonl)
- [Session with piece stream information](examples/piece-stream-session.jsonl)

## Credits

Created by Ruel Nathaniel Alarcon, originally descended from the [Tetris Bot Protocol](https://github.com/tetris-bot-protocol) by MinusKelvin.
