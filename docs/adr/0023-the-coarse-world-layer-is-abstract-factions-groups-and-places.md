# The coarse world layer is abstract Factions, Groups and Places

The coarse world layer promised by ADR-0009 models three kinds of record with integer-only state and no Entities. Factions have strength, resources and a goal. Groups are rosters of Blueprint counts plus named Notables, positioned on the World Plan graph (a ZoneId or a Connection in transit), with a Goal and a speed. Places are settlements and sites with stocks and an owning Faction. The layer steps once per Epoch on the World Clock, running data-defined utility scorers (the vocabulary of ADR-0022 at coarse resolution) per Group and Faction. Randomness comes from keyed derivations of `(record id, epoch)` (ADR-0013), so stepping is replayable and order-independent. Retired NPCs and Legacy records (ADR-0011) enter as Notables or as events on Places. The layer never touches Zone state: it only deposits stamped arrivals into Unloaded Zone Inboxes.

Active Zones report to the layer through a closed set of World Events posted as a side output of declared notable Effects: a Group member or Notable died, a Place's stock changed, a Character legacy, standing changed, a quest milestone reached. They are queued in World Clock order and consumed at the next Epoch step. Packs may declare which of their Parts' Effects raise World Events but cannot add event kinds.

Hostility is data. Entities belong to Factions, an integer standing table gives Faction-to-Faction standing, and Characters carry per-Faction standing. An "attitude toward X" Query returns hostile, wary, neutral or friendly by thresholds. Standing changes only through Effects, so saves and replay see them; the layer reads and writes the same table. Per-NPC memory of grudges overrides the table for that one Entity.

Authority over a Group is exclusive. While a Group is in the layer or an Inbox, the layer owns it. On arrival, stamped Effects materialize its roster as Entities near the arrival cell, each with a Group-membership Part, and the Zone owns them; the layer keeps a "materialized in Zone" stub. When the Zone unloads, or the Group's Goal carries it out of the Zone, survivors are absorbed: a World Event is emitted and the layer rebuilds the roster from survivors (counts per Blueprint, Notables by id), so casualties persist. Notables are real Entities that stay in their Chunks while idle and are layer-only only while travelling. Partial materialization was rejected as complexity without payoff.

Quests are data-defined state machines in Content Packs. States advance on World Events and standing changes, never by polling. Objectives are predicates over those events. State lives in the layer, keyed by `(quest id, holder)`, where the holder is the party or a single Character as the Blueprint declares. Rewards are ordinary Effects through a grant event. Failure, expiry and impossibility (the giver died) are explicit transitions, so no quest is orphaned. Packs may compose quests from the closed predicate set; script predicates must be pure over a World Event.

## Consequences

- The layer's state is saved with the world and included in the Hash Trail (ADR-0014).
- Group-versus-Group coarse resolution is left to a strength comparison with keyed randomness, finalized when a Faction scenario needs it.
- Dialog and quest-offer Commands belong to the UI ticket.
