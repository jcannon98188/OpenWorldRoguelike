# Entities are composed of Parts; Effects are the only state change

Entities are one type composed of Rust Parts that own their serializable state and react to events (Query, Before, After), not archetype ECS and not data-only components with separate systems. Actions resolve into first-class typed Effect records, and applying an Effect is the only way state changes. Each Effect is dispatched to Before handlers (which may veto or rewrite it) before it is applied, and After handlers react by queueing new Effects on a breadth-first queue owned by the Root Action, bounded by a depth and count budget.

This gives a total, deterministic order (handler priority, then Blueprint order, then fixed cross-Entity order), which seed + command log reproducibility needs. It also gives replay diffs, a cause-chain audit trail, and the stream clients render from. Blueprints compose and parameterize Parts only; new behavior needs a Rust Part or a script-backed Part, so there is no data expression language.

## Consequences

- Handlers must use only the zone RNG stream, no wall clock, no unordered iteration, and IDs from the world allocator.
- A script-backed Part must be indistinguishable from a Rust Part to dispatch: registered by ID, same event context, serializable state, deterministic, referenced from Blueprints. Any scripting design must meet this.
- Autonomous processes (fire, poison) are Actors or Part-scheduled events on the zone scheduler, with no global update loops.
