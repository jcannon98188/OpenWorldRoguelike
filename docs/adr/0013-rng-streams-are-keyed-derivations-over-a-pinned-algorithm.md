# RNG streams are keyed derivations over a pinned algorithm

Randomness comes from streams derived from a 64-bit hash of `(world seed, purpose tag, ids...)`, for example `(seed, "gen", ZoneId, stageId)` or `(seed, "elapse", ZoneId, id, epoch)`. The algorithm is our own pinned implementation (a counter-based generator, or SplitMix64 seeding xoshiro256++) covered by test vectors, never `System.Random`, and its identity goes in save and replay headers with the runtime version (ADR-0006). Only stateful streams persist a position: the world stream (character creation offers, ADR-0011) and each Zone's simulation stream (ADR-0001). Derived streams hold no state.

A single global stream consumed in order was rejected: any change in consumption order changes the world. Keyed derivation is what makes generation (ADR-0012) and catch-up (ADR-0009) independent of timing and order, and lets a new consumer be added without disturbing any other.

## Consequences

- Every consumer names its purpose tag; adding one is a registry change.
- Changing the algorithm or the hashing changes every world, so it is versioned like a save format.
