# Players never block the party indefinitely

Game time is purely turn-based, advancing only as Actors spend Ticks, and the simulation never reads the wall clock. Slow or absent players are handled by turning a timeout into an ordinary recorded Command, so seed + command-log replay stays exact.

Players due at the same Tick are asked together and their Commands resolve in EntityId order. When the simulation cannot advance without a human's input and at least one human has already locked in, a timer of X seconds starts; anyone who has not locked in when it expires is skipped, which the server records as a pass costing a normal action. After N consecutive skips a player is marked Away: their Actor passes the moment its turn is due, with no wait, until they send any input. A disconnect marks the player Away immediately.

Any player can pause, any player can unpause, and a pause has no time cap; it freezes the timer for everyone. A pause is server state, not a game-time event, so it does not affect replay. X and N are session configuration, off for solo play.

Considered: strictly serial player turns (a slow player blocks overlap), a pure wait with no timeout (one AFK player holds the party), a timer running from the first lock-in regardless of whether anything is blocked, per-player timers (skip a player before the party was ever held up), and an autopilot for skipped players (an AI feature, deferred; Away gives it somewhere to attach).

## Consequences

- The timer fires only while the simulation is blocked, so with Drift Window W > 0 a lock-in often lets the Zone keep running first.
- Autopilot for Away players, if wanted, is an AI-ticket decision.
