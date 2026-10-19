# Bodies are limbed, downing is not dying, and items are Entities with Parts

An Actor's body is a body plan of Body Parts, each with its own HP, flags (vital, grasping, locomotion) and a damage record, in the manner of Kenshi. A part at zero HP is disabled and degrades what depends on it; past a deeper threshold it is severed and becomes a real Entity. Downing and dying are separate: an Actor is Downed when a vital part is disabled or its blood is critical, takes only forced passes (as in ADR-0007), and can be rescued by others; it dies only from bleeding out of its Blood pool, a destroyed vital part, or a deliberate finishing blow. Damage is a hit-point model per part with no tissue layers.

An attack is an Action resolved by Before handlers into Effects: a roll-based contest (attacker skill against dodge or block) on the Zone RNG stream, a part chosen by size weight or by an optional called shot at a to-hit penalty, a data-defined damage type reduced by the armour covering that part, then `PartDamaged` and `BloodLost` Effects carrying presentation context. Any stage may rewrite the numbers (parry, shield, immunity).

Items are ordinary Entities with Parts (`Weapon`, `Consumable`, `AbilityGrant`, optional `Durability`, `Owner`). There is one canonical containment edge: the container holds the ordered child list and the child's location is derived and validated on load. Stackable items are one Entity with a `Quantity`, split and merged by Effects. Equipment slots come from the body plan: armour lists the part kinds it covers and a held item needs a free grasping part, so severed limbs and unusual bodies need no special cases. Equipping costs Ticks, and carried weight feeds speed through a Query. A worn-out item contributes nothing until repaired. Prosthetics are items attached to an empty body slot.

A stat is a Query evaluated in fixed layered order (base, additive, multiplicative, clamps), integer-only with a documented rounding rule, memoized per Entity and invalidated by Effects. Stats are a data-defined registry of IDs with a small core set the engine reads (speed, perception, carrying capacity, healing). Skills are stats raised by use through `SkillGained` Effects with a per-Tick cap; there are no levels or spendable points, and the creation Build sets starting skills only.

Timed effects are Conditions: data-defined records held in one `Conditions` Part per Entity, which owns one scheduled event for the next due record. Each condition declares its stacking policy (refresh, stack, strongest). Cure and dispel address conditions by ID, not Entity identity. Wounds and bleeding live in a `Wounds` Part with an Elapse hook (ADR-0009): healing and bleeding fold over whole epochs, and an idle wounded Actor can bleed out or recover during catch-up. Natural recovery runs at a rate set by the healing stat. Severed parts never regrow in the core. Magic healing is a temporary bonus to the healing stat and never an HP grant. Treatment is a Command using items and the medicine skill.

Severing and death are cascades of ordinary Effects. `PartSevered` creates the part Entity where the Actor stands, and held items and worn armour drop through an After handler on the body. `Died` leaves a corpse with gear dropped (ADR-0011), and loot and reactions are After handlers. Exceeding the cascade budget fails the Root Action closed with a recorded overflow Effect, never a silent truncation.

Refusal against attempt follows ADR-0008 by one rule: if the answer depends only on state the player can already see, it is a Refusal (out of reach, no free grasping part, over the weight cap, broken item, occupied slot, resource missing, acting while Downed); if a roll or hidden state decides it, it is an attempt that costs Ticks (a miss, a parry, a failed lockpick or treatment). Refusals return typed reasons.

Considered: a single hit-point pool, Dwarf Fortress layered tissue wounds (unaffordable to build and tune), stored modifier lists for stats, one Entity per condition, a fixed global slot list, XP levels with spendable points, and magic regrowth in the core.

## Consequences

- The data pack defines body plans, damage types, the stat registry, conditions, and item Blueprints; packs may extend all of them, but the Command and Effect sets stay closed (ADR-0008, ADR-0002).
- Every Part with an Elapse hook needs a test that fold and any closed form agree (ADR-0009).
- Ownership beyond the `Owner` Part (theft, crime, shops, faction property) is decided with the AI and faction framework.
- Downed player rescue interacts with the death-to-arrival flow of ADR-0011: a Downed character is alive, so no replacement arrives until it dies.
