# Open World Roguelike

A server-authoritative, persistent-world, small-group co-op roguelike core, independent of any front end.

## Language

### Simulation model

**Entity**:
Anything in a zone with identity: a creature, item, door, fire, trap or projectile. Defined by a blueprint and a set of Parts.
_Avoid_: Object, game object, thing

**Actor**:
An Entity that holds a Part giving it turns, so it can initiate Actions. A role, not a separate kind of Entity.
_Avoid_: Agent, character (a character is the player-owned Actor)

**Part**:
A unit of behavior and state attached to an Entity that reacts to events. The Entity's capabilities are the sum of its Parts.
_Avoid_: Component, trait, behaviour

**Blueprint**:
The data definition of an Entity: an identity, an optional parent Blueprint, and the Parts it is composed from with their parameters.
_Avoid_: Prefab, template, archetype

**EntityId**:
The stable handle by which one Entity refers to another. It may fail to resolve if the target is unloaded or gone.
_Avoid_: Reference, pointer

**Event**:
A notification dispatched to an Entity's Parts. Always one of three kinds: a Query (read a value), a Before (may veto or rewrite what is about to happen), or an After (react to what happened).
_Avoid_: Message, signal, callback

**Root Action**:
The Action that starts a cascade; every Effect it triggers, directly or through reactions, belongs to it.
_Avoid_: Transaction, turn

**Action**:
An immutable statement that an Actor attempts something, which resolves into Effects.
_Avoid_: Command (that is the player-facing input that may produce an Action), move, turn

**Effect**:
A discrete, recorded change to world state produced by an Action. The only way state changes.
_Avoid_: Mutation, delta, outcome

### World model

**Zone**:
A bounded three-dimensional region of the world with its own clock and scheduler, and the unit of simulation, save and interest. Its extent is chosen by its designer, so zones differ in size; vertical levels are part of the zone, not separate zones.
_Avoid_: Map, level, area, region

**ZoneId**:
The opaque, stable, never-reused handle of a Zone. Distinct from the key that tells world generation how to build the Zone the first time.
_Avoid_: Zone coordinates, zone key

**Chunk**:
A fixed-size block of a Zone's cells, used to store, save and replicate the Zone in parts. Not a unit of simulation or addressing.
_Avoid_: Tile block, sector, page

**Active Region**:
The part of a Zone that is currently simulated, because Actors are near it. The rest of the Zone is dormant.
_Avoid_: Loaded area, hot zone

**Zone Connection**:
A declared link between a cell range in one Zone and a cell range in another, through which Entities move from Zone to Zone.
_Avoid_: Portal, exit, transition

**Anchor**:
An Entity whose presence keeps part of a Zone in its Active Region. Every player is one; other Entities can be.
_Avoid_: Keep-alive, spawner

**Terrain**:
The material and shape of each cell of a Zone: floor, wall, water, slope, ladder. Not an Entity; Entities stand on it.
_Avoid_: Tile, map data

**Peek**:
A short, read-only view across a Zone Connection into the neighbouring Zone, marked as belonging to that Zone. It shows but cannot be acted on.
_Avoid_: Preview, cross-zone vision

### Time model

**Tick**:
The integer unit of game time. A normal action costs 100 Ticks.
_Avoid_: Turn, frame, step

**World Clock**:
The single monotonic game time of the world: the earliest clock among Zones holding a connected player. Idle Zones lag behind it.
_Avoid_: Global time, server time

**Drift Window**:
How far a Zone's clock may run ahead of the World Clock before it must wait.
_Avoid_: Lag tolerance, leash

**Lock-in**:
A player committing their Command for the Action that is due.
_Avoid_: Submit, confirm

**Away**:
A player state in which their Actor passes immediately, entered after repeated timeouts or a disconnect and left on any input.
_Avoid_: AFK, idle

### Input model

**Command**:
A player's intent, sent to the server and, if accepted, turned into an Action for their Actor. Not an Action itself.
_Avoid_: Input, order, request

**Turn Offer**:
The server's invitation to an Actor's player to act at one due turn; a Command only counts if it answers the current one.
_Avoid_: Prompt, turn token

**Standing Order**:
A multi-turn Command that supplies an Action each due turn until it completes or is interrupted.
_Avoid_: Macro, auto-action

**Refusal**:
The server declining a Command up front, with a reason, because a precondition fails; it costs no time.
_Avoid_: Failure (a failed attempt is an Action with Effects)

### Idle simulation

**Elapse**:
A Part's way of moving its state forward over time it was not simulated, producing ordinary Effects.
_Avoid_: Fast-forward, offline tick

**Epoch**:
The fixed span of game time in which Elapse is applied; catch-up always covers whole epochs.
_Avoid_: Era, period

**Catch-up**:
Bringing a lagging Chunk or Zone up to the World Clock before it is next simulated, by Elapse followed by a short stretch of real play.
_Avoid_: Fast-forward, resync

**Inbox**:
The stamped arrivals and changes waiting for an Unloaded Zone, applied at their times when it next catches up.
_Avoid_: Queue, mailbox
