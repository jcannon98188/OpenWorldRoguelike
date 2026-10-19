# Content Packs are namespaced archives composed under a world lock

A Content Pack (ADR-0015) is authored in a folder, usually with the released content editor, and built into a single archive file that hosts drop into the server's `packs/` directory. The server reads archives only; a folder loader exists for development. The editor and the server share one loader and validator, so a Pack that loads in the editor loads on the server. The manifest carries the Pack id, version, dependencies, a `format` version (the file layout) and a script `api` version (the scripting contract, decided with the script API). The server loads the current and the previous format, and refuses others with a message naming the editor version to use.

Data files are documented, diffable JSON (comments allowed), validated against a published schema. The editor is how most modders author, but the format works without it for version control, repair and generation. Text files are normalized to LF before hashing, so the same Pack hashes identically on every host.

Every ID (Blueprint, Part, ability, Recipe, stage) is `packid/name`, and a Pack defines IDs only in its own namespace. The base game is a Pack with the reserved id `core`. A Pack id is unique per world and never changes, because renaming one orphans saved Entities.

A Pack requires others by semver range (`other-pack >=1.2 <2`), forming a dependency graph without cycles. A missing or out-of-range dependency stops loading and names the Pack. The load order is a topological sort of the dependencies, with ties broken by an explicit order list the host sets; there is no implicit alphabetical order. The world stores a Pack Lock: the exact enabled list (id, version, Pack hash, order) and the conflict resolutions. Manifest ranges are checked against it only when a Pack is added or updated.

A Pack changes another Pack's content through field-level patch operations: add or remove a Part, set a Part parameter, change a Blueprint's parent. Two Packs conflict when neither depends on the other and both touch the same field; additive edits to different fields never conflict. Conflicts are found at load and reported with both Pack names. They are resolved by a resolution entry in the Pack Lock naming the winning Pack per field, which the editor can generate. The server never picks a winner.

Each Pack has a hash over its canonical manifest, data and scripts. The world's content hash covers the ordered list of (id, version, Pack hash) plus the resolutions. The handshake (ADR-0010) exchanges the ordered list, so a mismatch names the Pack that is missing, outdated or different. Replay headers (ADR-0014) and the generated-Zone stamp (ADR-0012) carry the world hash. Presentation assets are listed separately and are not part of it.

Installing is dropping an archive in `packs/` or a `pack add` command. The server validates every Pack and reports all errors at once. Enabling, updating or removing a Pack in an existing world is staged and applied at the next save boundary (ADR-0015). The hash proves a Pack is intact, not who wrote it; signing is not provided.

Considered: whole-record override with last loaded winning, as in Bethesda games (two Packs editing different things on one Blueprint silently lose one edit); flat global IDs with collision reports (shadowing and fragile saves); exact-version-only dependencies (every update becomes a manual edit); one combined identity in the handshake (cannot tell a friend which Pack is outdated); YAML or TOML data files (implicit typing hazards, awkward Part trees); a custom format.

## Consequences

- ADR-0010's single content identity becomes this ordered list; the version and hash equality rule is unchanged.
- The pack loader, patch application and validator are one library shared by the server and the editor.
- The editor must show conflicts and write resolutions; its design is part of the content tool, which is not yet decided.
- Migration of Part state and removal of Packs from a live world is decided in its own ticket.
