# Spatial services are deterministic integer queries over cell data

FOV, lighting and pathfinding are core services inside a Zone (ADR-0003), implemented in integer maths with a fixed iteration order and documented tie-breaks, because they feed AI decisions, visibility and replication and must replay identically (ADR-0014). Packs cannot replace them. Presentation-only maths on the client (smooth light falloff) may use floats because it never feeds back into the simulation.

**Cells.** Core ships a closed set of cell shapes (Solid, Open, Floor, Ramp, Stairs, Ladder, Liquid) with fixed geometric meaning: which faces are open to which neighbours. Materials are data-defined and pack-extensible (opacity, passability, move cost, light transmission). Half-walls, windows and similar are materials or Entities on a cell, not new shapes. Entities feed the services through a derived per-Zone dynamic layer (opaque, blocks movement, light emitted, light blocked, hazard cost) written by Parts through a declared contract on the After-event of the Effect that changed them. The layer is rebuilt from Entities on load and never saved; script-backed Parts use the same contract and are never queried per cell.

**Visibility.** A symmetric 3D shadowcast over shape face-openness, in integer maths with fixed octant order, bounded by sight radius, behind an `IVisibilityService`. It produces a per-viewer visible-cell set per resolved turn, packed as a bitset per Chunk, which is what ADR-0010 filters on. Geometry is symmetric; asymmetry comes only from light, sight radius and stealth. NPC "can see" is an on-demand query over the same core with a per-turn cache. Other senses (sound, scent) are an AI-framework concern, not part of replication visibility. A character's sight radius is a stat clamped to a configured Interest radius (overridable per Recipe within a hard ceiling), so baseline size stays bounded.

**Light.** Gameplay lighting: an integer light level per cell, computed server-side from light-source Entities and terrain opacity. A cell is seen if geometrically visible and either lit above the viewer's sight threshold or within a short darkvision radius. Clients render the level they are sent plus cosmetic falloff.

**Pathfinding.** Paths are advisory plans over a movement profile (capability bitset, size, cost multipliers; interned, data-defined), never calling Part handlers. The Action pipeline's legality check stays authoritative, so a stale path costs a failed attempt (ADR-0008). One facade offers bounded A* (default), Dijkstra maps for multi-target and flee within the Active Region, and hierarchical pathfinding over a Chunk-level abstract graph for long routes and travel-to. Budget exhaustion returns "no path within budget", never a best guess.

**Caches and execution.** Visibility is recomputed per player per resolved turn with an early-out when nothing relevant changed. Light is a per-Chunk integer buffer updated incrementally with a dirty set flushed once per Root Action. The abstract graph is per Chunk per profile, keyed on a Chunk terrain version. All caches are derivable, never persisted, and invalidated by the same Effects that change state, so replay rebuilds them. Per-player visibility for one Root Action may run in parallel against the post-Root-Action state and merge in player order; AI pathfinding stays serial in the scheduler step; the serial runner is the oracle (ADR-0014). Budgets are core constants, recorded in the Pack Lock if a pack tunes them.

**Terrain change.** One per-cell terrain Effect kind, with a batch container so a collapse is one cascade step, filtered by visibility like any Effect. An unseen change leaves the Remembered Map stale until reseen. Zone extent is fixed; runtime growth is not decided here.

**Client.** The client runs pathfinding over its known cells with the same core code and profiles, for path preview and travel-to (the server re-plans). It does not run FOV or gameplay lighting, which need hidden information.

An alternative was calling Parts per cell during searches and casts; it is slow and makes handler order a determinism hazard. Another was cosmetic lighting; it removes darkness and stealth as gameplay.

## Consequences

- Spatial code lives in the shared core library and is used by server, tests and client path preview.
- Shapes are a hard-to-extend core vocabulary; new geometry needs a core change.
- Performance budgets and enforcement tuning remain to be measured.
