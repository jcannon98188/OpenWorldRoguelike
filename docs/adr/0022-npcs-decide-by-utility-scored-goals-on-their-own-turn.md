# NPCs decide by utility-scored Goals on their own turn

An NPC Actor holds a Mind Part. When its turn is due on the Zone scheduler, the Mind is asked for an Action inside the normal Action pipeline (ADR-0001); there is no global AI loop. The Mind works in two layers. A utility layer scores a small set of data-defined Goals (flee, attack, eat, patrol, follow leader) with integer-only scorers expressed as Queries. The winning Goal runs a short data-defined behavior sequence that emits Actions. Runtime GOAP and general behavior trees were rejected: search cost and determinism auditing are not worth it, and designers get the same effect by composing Goals. Goals, scorers and sequences are Content Pack data, and a Goal may be script-backed through the Script Engine (ADR-0017).

A Goal's sequence persists across turns like a Standing Order and is re-scored only on interrupt or at a data-defined re-evaluation interval. Each decision has a deterministic budget counted in evaluations, never wall time, so replay matches (ADR-0014). A Mind that exceeds its budget keeps its previous Goal and the overrun goes to telemetry.

NPCs perceive through the same Visible Set and Light Level as players (ADR-0018) and keep a small per-NPC memory of last-known Entities and positions. Sound is a first-class perception: noisy Effects emit a Noise event with an integer loudness that propagates as a bounded integer cost flood over cell data and reaches nearby Minds. Scent is deferred.

Standing Orders (ADR-0008) and NPC Goal sequences share one closed set of interrupt kinds, evaluated by the server after each Root Action's Effects that the actor can perceive: hostile newly Revealed, took damage, Condition applied, path or target blocked, ally Downed, Noise above a threshold, inventory or encumbrance change, and the order's own completion. An order or Goal carries a mask over these kinds, with defaults per type. Packs cannot add interrupt kinds, consistent with closed Commands and Effects. Interruption only drops the order.

Away players get no autopilot: Away passes and the Character is later Stowed (ADR-0007, ADR-0011). A second decision-maker on player Actors was rejected as a source of surprise deaths and extra AI paths.

Resident NPCs may carry a Routine: a time-of-day table that biases utility scoring toward a Goal or Place. It is a strong bias, not an override, so danger still wins. Its Elapse hook (ADR-0009) places an idle NPC where the routine puts it at the target epoch, using the Movement Profile only to check reachability, falling back to the nearest legal cell.

## Consequences

- Minds, Goals, scorers and Routines need schemas in the Content Pack format (ADR-0016) and validation at startup.
- Noise propagation is a new bounded spatial service and needs a budget.
- The scorer vocabulary doubles as the coarse layer's (ADR-0023), so it must stay integer-only and cheap.
