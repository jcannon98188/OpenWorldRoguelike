# Zones are designer-sized 3D volumes and the seams of simulation

A Zone is a bounded three-dimensional volume with a designer-chosen extent, and vertical levels are part of the Zone, not separate Zones. A Zone is the unit of simulation, scheduling, save and interest, so a locale such as a town and the dungeon beneath it is one Zone, and climbing a ladder inside it is an ordinary move with no handoff. Zones link to each other only through explicitly declared Zone Connections (cell range to cell range); there is no implicit grid adjacency, which is what lets Zones of different sizes meet.

Only Entities cross a Zone boundary, on their own turn. A pursuer follows by taking its own turn onto the same connection and arrives with its identity and energy intact; there is no follower logic. Effects, projectiles and targeting never cross. Reads cross a little: a connection may declare a peek depth, within which the neighbour's terrain and Entities are shown read-only and marked as belonging to another Zone, so the boundary is visible but not actionable.

An alternative was a single infinite chunk-streamed plane, and another was depth as chains of small Zones. The first conflicts with per-Zone clocks, schedulers and save blobs. The second makes every ladder a cross-Zone handoff and breaks the "simulation never stops" feel.

## Consequences

- The core has no standard Zone size. Extent is data, validated at startup against a configurable cell budget with an explicit per-Zone override.
- A Zone is stored, saved and replicated in Chunks, and only its Active Region ticks: chunks near an anchor (players, or Entities with an anchor Part), with Effects waking dormant chunks they touch. A Zone is either Unloaded or Resident.
- EntityIds are world-global and never reused, indexed to their current ZoneId, or "in transit". A crossing is validated against a Resident destination, preloaded when a mover nears a connection, and recorded in the world index so a crash cannot lose or duplicate the Entity. Arrival always finds a passable cell or the crossing is refused before departure.
- FOV, pathfinding and lighting are 3D services within a Zone and stop at its boundary except for the peek.
- Players may be in different Zones at once with independent clocks; reconciling those clocks is a time-model decision.
