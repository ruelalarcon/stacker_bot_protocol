# Changelog

## SBP/1

- Initial stable specification.
- Uses newline-delimited JSON objects for process transports.
- Separates authoritative client state from bot suggestions.
- Defines registration, rule negotiation, snapshots, suggestion requests,
  accepted-placement advances, queue updates, runtime info, and shutdown messages.
- Supports arbitrary string piece identifiers.
- Defines capability negotiation for randomizers, kicksets, movement features,
  spin and back-to-back behavior, board dimensions, piece streams, spawn
  positions, and mid-session board replacement.
- Defines optional state and request data including piece stream alignment,
  incoming garbage chunks, and extension-scoped runtime data.
