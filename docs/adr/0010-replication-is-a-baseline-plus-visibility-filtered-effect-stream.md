# Replication is a baseline plus a visibility-filtered Effect stream

A client's replica is built from a baseline snapshot of its interest set, taken at a quiescent point, followed by an ordered stream of Effects on a reliable-ordered stream per Zone, with a sequence number. One message carries all the visible Effects of one Root Action, tagged with its Tick and cause, so a client applies and animates a whole cascade at once and never renders half of one. Turn Offers and refusals (ADR-0008) travel on the same Zone stream, ordered with the Effects. Periodic full snapshots and diffing were rejected: turn-based play is low rate, snapshots lose the cause chain, and ADR-0002 already commits clients to applying Effects.

What a client receives has two layers. Chunk interest, coarse and cheap, is the set of Chunks within the player's perception volume; it decides what is baselined and streamed at all. Visibility filtering is computed once per player per resolved turn from the server's FOV service as a per-cell set: an Effect is sent only if it touches a visible cell, or an Entity the player owns or whose field is Public (ADR-0002). When an Entity enters view the server synthesizes a `Reveal` carrying its visible state, and when it leaves, a `Conceal`; these are the only synthesized Effects. Sending whole Chunks and letting the client hide the rest was rejected because it leaks hidden information.

Every join, reconnect and Zone crossing starts from a fresh baseline of the current interest set, then the stream resumes from the baseline's sequence number; resuming from a ring buffer is a later optimization behind the same messages. A Peek is a separate small stream and baseline for the cells within the connection's peek depth, filtered by the same visibility rule, tagged with the neighbour's ZoneId and shown read-only on the neighbour's own clock; a non-Resident neighbour shows last-known terrain and no Entities, marked stale. A party channel, separate from the Zone streams, carries each member's name, Zone name and coarse status (health band, Away, paused) and no world state; spectating other Zones is not supported.

The remembered map is character state owned by the server: a per-Chunk seen bitset of terrain, plus last-known `Reveal` snapshots of Entities with a cap, held in the character's save and sent in baselines. A client-owned memory was rejected because it is lost on a new device and cannot be validated.

Client and server must run the same content: the handshake exchanges a content pack identity of name, version and content hash, and a protocol version, and any mismatch refuses the connection. Presentation assets are not part of the hash. Effects and Commands are closed hierarchies with stable numeric type ids from one registry, shared by wire, save and replay formats, which are otherwise separate. The encoding (MemoryPack or hand-written writers) is a build-time benchmark, not an architecture decision.

## Consequences

- The server owns a per-player visible-cell set; the FOV and lighting algorithm is decided with the spatial services.
- Baseline size is bounded by the interest set, not the Zone.
- A later feature may stream definitions on mismatch without changing the replication contract.
