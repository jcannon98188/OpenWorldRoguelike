# Characters are Builds that arrive by world event

A player has at most one living character per world, identified by a random per-world player token the host approves; there are no accounts or external identity. A character is created from a Build: a fully resolved, validated record of creation choices (origin, chosen options, name, starting kit). All randomness happens when a Build is made, so constructing an Entity from a Build is deterministic and reusing a Build yields an identical starting character. There are three ways to a Build: server-rolled offers from data-defined origin tables on the world RNG stream, picked or rerolled a bounded number of times; a full customization validated server-side against data-defined rules (budget, allowed options; only the name is free text); and Reuse Last, the previous character's stored Build re-validated against the current data pack, falling back to another path if refused. Creation exchanges are Commands (ADR-0008) and so appear in the replay log.

A character enters play by an arrival event, an ordinary Effect at a Zone's current Tick. The event picks an arrival point, a Zone-authored marker, in or connected to a Zone that has a living party member, falling back to a settlement hub, which is also where a party with no living member arrives. Blocked points are skipped (ADR-0003); the delay from death to arrival is configuration, and 0 means the same turn. Joining a world with no character and rejoining with a stowed one use the same arrival path.

Death is ordinary Effects: the Actor becomes a corpse Entity with its gear dropped in place, and both persist and decay like any Entity. A legacy record (name, deeds, cause and place of death, and the Build) is kept with the character and posted to the coarse world layer as an event. Nothing carries over to the next character; inheritance can be added later as an arrival-time Part. A player may retire a living character, which stows it permanently and makes it a retired NPC from its legacy record, then create a new one.

A disconnected player's Actor is Away (ADR-0007) for G Ticks of game time, then stowed: removed from the Zone, held in the character record, and returned by an arrival Effect at its stow cell, or the nearest passable one, on reconnect. G is configuration.

A living Actor is world state, saved in its Zone's Chunks with every other Entity, so there is one source of truth. The character table holds only what is not Entity state: the owning token, the Build, the Remembered Map (ADR-0010), the legacy record, and, while stowed, the serialized Entity and stow cell. Stowing and un-stowing commit atomically across the world and character tables, with the world index marking the Entity stowed. Characters cannot move between worlds, because EntityIds and the data pack belong to the world.

Considered: inheritance on death, erasing the dead, a fixed respawn hub, indefinite Away-in-place offline (a free kill), immediate stow on disconnect, alts per player, and storing the living Entity in the character table.

## Consequences

- The data pack must define origin tables and creation rules.
- Corpses, retired NPCs and stowed Entities are ordinary Entities or world-index states, not new kinds of object.
