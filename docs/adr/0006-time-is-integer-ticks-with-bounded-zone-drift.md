# Time is integer ticks, with a world clock and bounded Zone drift

Each Zone's scheduler is an absolute-time priority queue of Actors keyed by `(nextActAt, EntityId)`, in 64-bit integer Ticks. A normal action costs 100 Ticks, speed scales cost with integer arithmetic and a pinned rounding rule, and an Actor's "energy intact" across a crossing is its remaining delay relative to the Zone clock. Ties order by EntityId, so the order is a pure function of saved state and identical after a load. No floating point appears in simulation time.

There is a single monotonic World Clock, defined as the minimum Zone clock over Zones holding a connected player. Every Zone's clock is at most the World Clock; the gap is the catch-up debt paid when an idle Zone becomes Resident. NPC-only Resident Zones are pulled up to the World Clock but never hold it down, and a Zone with no connected player anywhere freezes the World Clock. A Zone may run at most a Drift Window W ahead of the World Clock, then parks until the slowest player Zone catches up. W is configuration.

A crossing arrives at `max(source clock, destination clock)`: if the destination is behind, the Entity is held in transit until it reaches the source time; if ahead, the Entity arrives at the destination's time. The window bounds the hold, and time never runs backwards in a Zone. A joining or reconnecting player's Zone first catches up to the World Clock, and only then counts towards the minimum.

Zones may run on separate threads. Crossings are timestamped messages consumed only once the receiving Zone's clock reaches the stamp, ordered by `(time, EntityId)`, and the window is the lookahead that makes this safe. The command log is per Zone, tagged `(ZoneId, Tick)`, plus a world-level log of crossings. A first implementation may run Zones serially behind the same interface.

Considered: energy accumulation per global tick (O(Actors) per tick, and a harder catch-up), unbounded Zone drift (a crossing could stall forever or time-travel), hard lockstep (a slow player stalls every Zone), and a wall-clock-driven world (non-reproducible).

## Consequences

- World Clock, Zone clock and `nextActAt` are saved; scheduler queue order needs no saved ordering state.
- Catch-up for an idle Zone is popping events up to the target time. How coarse it may be is the off-screen simulation design.
- The World Clock and runtime version go in save and replay headers.
