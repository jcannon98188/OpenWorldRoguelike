# Clients apply Effects and never run handlers

Clients hold a replica of the state they are allowed to see and update it by applying server-sent Effects through a pure `(state, Effect) -> state` function shared with the server. Event handlers are server-only. Part state fields carry a visibility level (Public, Owner, Server), defaulting to Server, and Effects are filtered by the same rule. Clients animate locally from the Effect stream, but presentation is cosmetic and never gates the simulation.

This follows from thin clients with no prediction, and keeps hidden information off the wire by default. Snapshots (saves and replication baselines) are taken only at quiescent points, between Root Actions.

## Consequences

- Effects must carry enough presentation context (source, target, kind, magnitude) to animate without extra queries.
- Whether the wire carries snapshots, Effects or both is left to the replication protocol decision.
- Saves store the Blueprint ID plus Part runtime state and per-instance overrides; static parameters are re-read from Blueprints on load, so Part state is versioned and a missing Blueprint is a hard error.
