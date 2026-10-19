# Catch-up is an epoch fold of Elapse hooks, with a short real tail

A Chunk or Zone that lags the World Clock is brought current by Elapse, not by replaying its scheduler. A Part may opt in to an Elapse hook that moves its state forward through ordinary Effects: regrowth, decay, regeneration, fires burning out, stocks refilling. Parts without a hook do not change while idle. After Elapse, the real scheduler runs for a short tail of K Ticks so Actors settle into plausible positions before a player sees them. Replaying the whole gap and doing nothing were both rejected: the first is unaffordable for a gap of months, the second leaves the world dead.

Elapse is defined as a fold over whole epochs of E Ticks (configuration, e.g. one game hour). Each epoch's randomness is derived by hashing `(ZoneId, Entity or Part id, epoch index)`, so catching up in one step or several gives exactly the same result. A closed-form shortcut is allowed only where a test shows it matches the fold. Resident and Unloaded transitions are deterministic functions of simulation state (Anchor presence plus a grace period in Ticks), never memory pressure or the wall clock; a memory cap refuses a new Resident Zone and the crossing waits in transit (ADR-0006) rather than evicting a live one. This keeps catch-up a pure function of saved state and target epoch, so replay needs no residency log.

The unit of catch-up is the Chunk. Each Chunk records the epoch it has been elapsed through; waking a Dormant Chunk catches it up before the waking Effect applies, and catching up a Zone is catching up its Chunks as they load. A Part whose Elapse reads neighbouring Chunks declares them, and they are brought to the same epoch together.

Each hook declares a saturation horizon beyond which its state stops changing, which bounds the cost of a long gap. For a Zone that is Loaded but not yet Resident, catch-up runs on a worker thread, since it touches only that Zone's state, starting at preload (ADR-0003). The real tail runs on the simulation thread when the Zone becomes Resident. A crossing whose destination has not finished waits in transit, bounded by the Drift Window.

Things that move between idle Zones, or are expected to have acted while a Zone was idle (a caravan, a raid), belong to a separate coarse world layer on the World Clock over abstract state. It never touches Zone state directly; it deposits stamped arrivals and changes into an Unloaded Zone's inbox, which catch-up applies as Effects at their stamped Ticks. This is the same mechanism as an in-transit crossing. What the layer models (factions, quests, NPC schedules) is decided with the AI framework.

## Consequences

- Elapse output is Effects, so saves, replay and the audit trail see it like any other change.
- Each Part with an Elapse hook needs a test that the fold and any closed form agree.
- A per-Zone catch-up budget (Parts times epochs) is configuration with a warning when exceeded.
