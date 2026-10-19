# Mods are server-side Content Packs in a capability sandbox

Modding is done by Content Packs: versioned bundles of data (Blueprints, Recipes, Layouts, Samples, terrain materials), script-backed Parts, generation stages and Elapse hooks (ADR-0001, ADR-0009, ADR-0012). A world runs a fixed set of Packs, and scripts run on the server only. The audience is a host who runs a world for friends or alone, not a professional operator, and who may install a Pack downloaded from strangers. Scripts therefore run in a capability sandbox that is a real security boundary, not just a determinism guard: no file, network or process access, and bounded CPU and memory per call, so a bad Pack cannot harm the host's machine or hang the server. The sandbox also enforces the determinism rules of ADR-0001 (zone RNG stream only, no wall clock, no unordered iteration).

A Pack cannot add Command types or Effect kinds. Commands stay a closed set (ADR-0008); new player verbs are data-defined abilities reached through `UseAbility`, and `Move` and `Interact` take on new meanings through the Parts involved. Effect kinds stay closed because clients apply Effects with a pure function shared with the server (ADR-0002); a Pack-defined Effect kind would force every client to carry the Pack's code. Packs compose the existing Effect vocabulary, and growing that vocabulary is a core change made for everyone. A Pack may change core rules only through the ordinary Query, Before and After handlers; there is no override mechanism.

Packs may patch other Packs' content, such as adding a Part to an existing Blueprint, but only through declared dependencies and an explicit load order. Conflicts are reported at load time and never resolved silently, and the order feeds the deterministic handler order of ADR-0001. The set of Packs, their versions and their order are covered by the content-pack hash (ADR-0010, ADR-0014).

A live world's Pack set never changes silently. Adding or updating a Pack happens at a save boundary under a new content-pack hash, with a migration for any Part state whose version changed. Removing a Pack requires an explicit operator migration that rewrites or deletes the Entities it defined, because a missing Blueprint is a hard error (ADR-0002).

Considered: scripts as trusted in-process C# (no sandbox against a downloaded Pack, and .NET gives no practical in-process sandbox); an open Command or Effect set registered by Packs (protocol change per mod, client code execution); last-Pack-wins override (silent conflicts that break replay); Packs that only add new IDs (too weak to be worth modding); live hot-swapping of Packs in a running world (breaks the content-pack hash that replay and the handshake depend on).

## Consequences

- The script language and host must support a hard sandbox with call budgets and integer-only simulation state (ADR-0014); this narrows the choice and is decided separately.
- Pack format, composition rules, the script API and the migration mechanics are decided in their own tickets; this ADR fixes only the boundary.
- Client presentation assets (glyphs, sprites, sounds) are not decided here, and no Pack code ever runs on a client.
- Untrusted player-uploaded mods on a shared public server, Pack-defined Commands and Effect kinds, and a Pack marketplace are out of scope.
