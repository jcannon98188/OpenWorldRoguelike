# Terrain is cell data, not Entities

The floor, walls, water and slopes of a Zone are cell data (a material and a shape per cell), not Entities. Things that need identity or behavior (doors, levers, fires, traps) are Entities placed on cells. Terrain still changes only through Effects, so digging and collapse obey the rule that Effects are the only state change.

A Zone can hold millions of cells, and an Entity per cell would be too heavy to simulate, save or replicate. The cost is two kinds of thing in the world: a wall that must do something has to be a terrain cell with an Entity on it, or be promoted to one by an Effect.

## Consequences

- Vertical movement is derived from cell shape (ladders climb, ramps connect levels, open air drops), checked by the mover's Parts, with no explicit links between levels inside a Zone. Pathfinding builds its graph from the same rules.
- Terrain changes are a distinct kind of Effect that clients apply like any other.
