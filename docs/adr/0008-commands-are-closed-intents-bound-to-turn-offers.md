# Commands are closed intents bound to turn offers

A Command is the player's intent, sent by the single client and validated by the server, which turns it into an Action for the player's Actor. Commands are a closed set of intent records (`Move(dir)`, `Interact(target)`, `UseAbility(abilityId, target)`, `Pass`, and so on); abilities are data-defined, so the game extends through `UseAbility` and the Command set itself stays small. Pause, unpause, chat and previews are session messages on a separate channel and never enter the simulation log. Mapping keys, mouse and bindings to Commands is entirely client-side; the core knows no input device.

Each due turn is a server-issued turn offer naming the Actor and Tick. A Command must quote the current offer, so a stale, duplicate or late message cannot land on the wrong turn. A player may replace their Command on the same offer until the batch resolves, and only the final one is accepted. One player can be offered turns for several Actors.

The server decides what an intent means: `Move(dir)` into a cell is resolved by the Parts involved into a step, attack, swap, open or refusal, because the client's replica may be stale or incomplete. A refusal (a precondition checkable without trying, from a pure Query-style pass over the Actor's Parts) costs no Ticks, is not logged, and returns a reason. An attempt that then fails is an ordinary Action with Effects and costs Ticks. A multi-step interaction is assembled on the client and sent as one complete Command; a confirmation is a refusal reason of "needs confirmation" followed by a resend with a confirm flag. A cascade is never suspended to ask a question.

A multi-turn Command (travel, rest, explore) sets a standing order, from which the server derives one Action per due turn until it completes or an interrupt condition raised by a Part fires; it counts as locked in each turn, so it never trips the timeout.

Previews are a closed set of typed read-only queries (`ValidTargets`, `Path`, `Interactions`, `AvailableAbilities`), answering only with what the player can see and rate limited. Pure geometry may later run client-side on the replica.

The command log records only inputs that cannot be re-derived: per-Zone entries `(ZoneId, Tick, ActorId, Command)` for final accepted Commands, including the server's timeout passes as ordinary `Pass(Timeout)` Commands, plus a world-level log of session events (connect, disconnect, reconnect) and crossings. Standing-order continuations and Away auto-passes are re-derived on replay; Away itself is derived from consecutive logged timeout passes and logged disconnects. Entries carry a log-format version.

Considered: an open Command set registered by mods (every extension becomes a protocol change), client-decided bump semantics (stale replica), server-suspended prompts (breaks the quiescent-point rule of ADR-0001 and ADR-0002 and the timer of ADR-0007), a generic dry-run preview (runs handlers on scratch state and can leak hidden information), and logging every resolved Action (mixes inputs with outputs).

## Consequences

- Capability queries are part of the contract between core and client, kept narrow and typed.
- Replication must carry turn offers and refusal reasons alongside Effects.
