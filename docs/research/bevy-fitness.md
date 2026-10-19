# Bevy fitness for the thin client

Research for issue #29 (part of #1). Checked 2026-10-19 against primary sources only: bevy.org release notes and migration guides, the crates.io API, upstream READMEs, docs.rs and the Bevy repository. Where a claim is my inference rather than a source statement, it is marked **(inference)**. Nothing here edits an ADR.

## Constraints being tested

- ADR-0020: Bevy pinned to an exact version, confined to the client crate, upgrades deliberate; 2D with depth (z-levels, lighting); UI is code or markup with a widget kit and hot reload; `egui` allowed for debug overlays and the editor; AI tooling writes stale Bevy API.
- ADR-0002 and ADR-0010: the client holds a replica and animates from the Effect stream. The render layer is cosmetic and must not feed the simulation.
- ADR-0018: gameplay light level is server-sent; the client renders it plus cosmetic float falloff.
- ADR-0004: terrain is cell data, not entities.
- Issue #18 (UI and audio seams) and #26 (Content Pack assets reaching clients that join a friend's server; asset bytes are not part of the pack hash) consume this research.

## Recommendation (summary)

1. Bevy remains fit as the client engine. Pin the exact stable version (currently 0.20.0) and treat each upgrade as a scheduled task; the cadence is roughly one release every 3.5 to 5 months and each guide has 75 to ~120 entries.
2. Do not take a hard dependency on a third-party render ecosystem crate at the pin. Ecosystem crates lag the engine by days to months, and `bevy_ecs_tilemap` has no 0.20 release yet. Render the 2D-with-depth view with Bevy's own `TilemapChunk` plus a small client-owned chunk layer, and keep third-party crates (`bevy_light_2d`, `bevy_kira_audio`, `bevy_egui`) behind client-crate seams.
3. For game UI, start on Bevy UI (with `EditableText`, the new `bevy_text` on Parley, and BSN scenes). Treat Feathers as editor-flavoured and do not depend on it for game UI. No visual layout editor exists for Bevy UI or BSN; the `.bsn` asset file and the official editor are explicitly not yet shipped. Slint-in-Bevy is the fallback only after a spike on the target OS.
4. Asset hot reload and a custom source for Content Pack assets are supported by documented engine features (`file_watcher`, `embedded_watcher`, `AssetSourceBuilder`, asset processing). The design in #26 can sit on a custom `AssetReader` rooted at a pack cache.
5. Keep stale-API generation in check with repo-local pinned docs: a client-crate `AGENTS.md`, the exact-version migration guides, and the Bevy `examples/` tree at the pinned tag. Bevy ships no official versioned agent doc (the proposal, bevy#23867, is open and unaccepted).

## 1. Current stable release and migration burden

- crates.io: `bevy` max stable is 0.20.0, published 2026-10-08, with `rust_version` 1.97.1. 0.20.0-rc.1 was 2026-09-15 and rc.2 2026-09-28. (https://crates.io/api/v1/crates/bevy)
- Recent history from the same endpoint: 0.17 (2025-09-30 per release notes), 0.18.0 (2026-01-13), 0.18.1 (2026-03-04), 0.19.0 (2026-06-19), 0.19.1 (2026-08-13), 0.20.0 (2026-10-08). Gaps between minor releases: about 3.5, 5 and 3.7 months. The release list is at https://bevy.org/news/.
- MSRV: Bevy's setup page states the minimum supported Rust version is "the latest stable release". The crates.io `rust_version` rose from 1.89 (0.18) to 1.95 (0.19) to 1.97.1 (0.20). Consequence for ADR-0020's pinned `rust-toolchain.toml`: each Bevy upgrade will normally also be a toolchain upgrade. (https://bevy.org/learn/quick-start/getting-started/setup/)
- Migration burden, counted from the official guides' tables of contents (my grouping by area is approximate):
  - 0.19 to 0.20: 75 entries; about 21 rendering, 19 ECS, 14 UI, 5 text, 4 assets, 0 audio. Notable breaks: BSN syntax (`@` scene references, `--` entity separator), observer type changes (`On<E, B>` removed), WESL shaders replacing naga_oil (custom shaders must be rewritten), exclusive systems unified, `iter_many` yields `Result`. (https://bevy.org/learn/migration-guides/0-19-to-0-20/)
  - 0.18 to 0.19: about 103 top-level sections plus 16 subsections; resources now imply components (`#[derive(Resource)]` types cannot also be components), text moved from cosmic-text to Parley, `bevy_scene` renamed `bevy_world_serialization`, the `audio` feature is no longer implied by `2d`/`3d`/`ui` (audio can silently stop working). (https://bevy.org/learn/migration-guides/0-18-to-0-19/)
- Implication: the client crate is the only place this applies (ADR-0020). A thin client that applies Effects and draws is mostly rendering, UI, text, assets and input, which are the areas that churn most. Keeping gameplay logic out of Bevy ECS limits the ECS-side churn to the replica-to-render bridge. **(inference)**

## 2. 2D with depth: z-layers, lighting, tilemaps

- Z-layers: sprites and 2D meshes use `Transform.translation.z` for ordering in the 2D pipeline; I did not fetch a primary source for the ordering rules, so verify against the pinned docs in the spike. In 0.20 the sprite renderer is unified with 3D infrastructure ("Sprite Render Backend Unification"), with sprite materials (`SpriteMaterial`, `MaterialExtension2d`) allowing custom shaders per sprite. (https://bevy.org/news/bevy-0-20/)
- Built-in tilemap: since 0.17 Bevy has `TilemapChunk` ("a performant way to render tilemap chunks", a first step to a built-in tilemap). The repo example `examples/2d/tilemap_chunk.rs` shows `TilemapChunk { chunk_size, tile_display_size, tileset, .. }` and `TilemapChunkTileData` as a `Vec<Option<TileData>>`; the tileset is an array texture (one layer per tile). It lives in `bevy::sprite_render`. A second example covers mirrored and rotated tiles. (https://bevy.org/news/bevy-0-17/, https://github.com/bevyengine/bevy/blob/main/examples/2d/tilemap_chunk.rs). I could not confirm from these pages whether the type is unchanged in the 0.20.0 tag; confirm at the tag.
- Fit with the ADRs: a Zone is a 3D volume of Chunks (ADR-0003, ADR-0004). One `TilemapChunk` per Chunk per visible z-level, spawned from the replica's Chunk data, matches that shape and keeps terrain out of the entity world **(inference)**. Entities (creatures, items) are sprites or glyphs layered above.
- `bevy_ecs_tilemap`: crates.io max stable 0.19.0 (2026-07-04, previous 0.18.1 on 2026-01-16). The README compatibility table maps bevy 0.19 to 0.19 and `main` to a `bevy-track` branch; there is no 0.20 row. It has therefore been out of step with the engine for 11 days at the time of writing. (https://crates.io/api/v1/crates/bevy_ecs_tilemap, https://github.com/StarArawn/bevy_ecs_tilemap) Its history shows it tracks each Bevy minor within days to roughly three weeks, so it is a fair choice but would hold the pin back.
- 2D lighting: Bevy core has no 2D lighting; the 0.19 and 0.20 notes list only 3D lighting (contact shadows, area lights, Solari) and say nothing about 2D lighting. (https://bevy.org/news/bevy-0-19/, https://bevy.org/news/bevy-0-20/) The third-party `bevy_light_2d` 0.11.0 (2026-10-09) targets Bevy 0.20 per its README compatibility table (0.10 for 0.19, 0.9 for 0.18) and offers point lights, occluders sorted by z-depth, dynamic shadows and camera ambient light. It does not state normal-map support. (https://crates.io/api/v1/crates/bevy_light_2d, https://github.com/jgayfer/bevy_light_2d)
- Fit with ADR-0018: gameplay light is an integer level per cell sent by the server and the client adds cosmetic falloff. That is a per-cell tint or a shader parameter over the chunk, not dynamic shadow casting; a simple custom material on the chunk (sprite materials exist in 0.20) is likely enough, so `bevy_light_2d` is optional and should not be a dependency of the core design **(inference)**. Recommend the spike prove a per-cell light-level texture sampled by the tile shader.
- Glyph versus sprite: both are supported by the above (`Text2d` for glyphs; tileset array texture for sprites). A tileset of glyph cells rendered through `TilemapChunk` avoids per-cell text entities **(inference)**.

## 3. Text and UI

- Text: since 0.19 Bevy text layout uses Parley (moved from cosmic-text). `FontSource` selects by handle, family or generic category; `TextFont` has weight, width and style (variable fonts); `FontSize` supports `rem` and viewport units; `LetterSpacing` added. In 0.20, `InlineBox` and `InlineImage` reserve space and flow images within text, and the default font size became `rem(1)`. (https://bevy.org/news/bevy-0-19/, https://bevy.org/news/bevy-0-20/)
- Text input: 0.19 added an upstream `EditableText` with cursor, selection, clipboard, IME and multiline; placeholder text, undo/redo and password masking were not yet implemented in 0.19. Examples for text input, IME and multiline exist in the repo. (https://bevy.org/news/bevy-0-19/, https://github.com/bevyengine/bevy/blob/main/examples/README.md)
- Widgets: Feathers is described as "an opinionated editor-centric UI toolkit"; 0.20 adds Color Input, List View and Scrollbar, Dropdown, Lazy Menu, scrubbable Number Input, headless Tab widgets and contextual theming. Because it targets the future Bevy editor, it is a poor default for a game HUD **(inference)**; verify with the gallery example (`ui/widgets/feathers_gallery.rs`).
- Scenes as markup: BSN (`bsn!` macro) arrived in 0.19 and was reworked in 0.20. Asset-based scenes work at engine level, but 0.19 shipped no first-party `.bsn` loader, and 0.20's "What's Next?" lists a human-readable, hot-reloadable `.bsn` format and a scene editor as planned. (https://bevy.org/news/bevy-0-19/, https://bevy.org/news/bevy-0-20/) So today UI "markup" means Rust `bsn!` code, not a hot-reloadable file. This is a gap against ADR-0020's "code or markup with hot reload".
- Visual layout editor: none is first-party. The search for 0.20 status found that the official editor "is not part of this release"; a third-party editor, Jackdaw, is early (its README warns to expect bugs) and was mid-upgrade to 0.20. I found no mature visual editor for Bevy UI. (https://bevy.org/news/bevy-0-20/; https://github.com/jbuehler23/jackdaw)
- egui: `bevy_egui` 0.43.1 (2026-10-10) is compatible with Bevy 0.20 (0.42 and below with 0.19). It was updated within two days of the engine release. Setup is `EguiPlugin::default()` and systems in `EguiPrimaryContextPass`. It fits ADR-0020's allowance for debug overlays and the content editor. (https://crates.io/api/v1/crates/bevy_egui, https://github.com/vladbat00/bevy_egui)
- Slint: Slint 1.18.1 (2026-09-21) is current; it is licensed GPL-3.0, a Royalty-free-2.0 license, or a commercial license, so the license terms need reading before adoption. Slint 1.12 notes describe a wgpu integration for embedding into wgpu-driven apps such as Bevy, and the Slint repo has `examples/bevy/{bevy-hosts-slint, bevy-hosts-slint-gpu, slint-hosts-bevy}`. Its wgpu feature must match Bevy's wgpu version. A January 2026 discussion on Bevy 0.17.3 and Slint 1.10 on Windows reports duplicate-ICU linker errors with Skia, and the maintainer asked for a minimal reproduction; there is no resolution recorded. I did not verify the example's Bevy version or that it builds on 0.20 on Windows. (https://crates.io/api/v1/crates/slint, https://github.com/slint-ui/slint/tree/master/examples/bevy, https://github.com/slint-ui/slint/discussions/10681)
- Verdict for #18: Bevy UI is viable now for a turn-based HUD (panels, lists, text, input) and has the most momentum; its weak point is no hot-reloadable markup or visual editor yet. Slint offers a markup language with a live preview, but with integration and platform risk; run a short Windows spike on the pinned versions before choosing it.

## 4. Audio

- Built-in: Bevy has `bevy_audio` (`audio` feature collection = `bevy_audio` + `vorbis`); 0.19 moved to Rodio 0.22 and made `audio` a separate feature, so `default-features = false` clients must enable it explicitly. The repo has examples for playback, control, spatial 2D and 3D, soundtrack-by-state and a custom `Decodable`. (https://github.com/bevyengine/bevy/blob/main/docs/cargo_features.md, https://bevy.org/learn/migration-guides/0-18-to-0-19/, examples README)
- `bevy_kira_audio` 0.27.1 (2026-10-13) targets Bevy 0.20 (0.26 for 0.19, 0.25 for 0.18). It adds channels (pause, stop, volume, speed and panning per channel), limited spatial audio (volume and panning only), ogg/mp3/flac/wav by feature, and web support. (https://crates.io/api/v1/crates/bevy_kira_audio, https://github.com/NiklasEi/bevy_kira_audio)
- Fit: a turn-based game needs channels (music, effects, UI, ambient) and cue-by-Effect triggers; built-in audio may be enough, `bevy_kira_audio` is the low-risk addition if channel mixing is needed. Put either behind the audio seam of #18.

## 5. Asset loading and hot reload for Content Pack assets (#26)

- Features: `file_watcher` (hot reload from disk), `embedded_watcher` (hot reload for in-memory providers), `asset_processor` (built-in processing), all opt-in cargo features. Repo examples: hot asset reloading, asset processing, custom asset IO, extra asset source, embedded asset. (cargo_features.md, examples README)
- Custom sources: `AssetSourceBuilder` takes a reader factory (and optionally writer, watcher and processed variants), per docs.rs. The `custom_asset_reader` example registers a source with `app.register_asset_source(...)` in a plugin added before `DefaultPlugins`, wrapping an `AssetReader` with `read`, `read_meta`, `read_directory` and `is_directory`. (https://docs.rs/bevy_asset/0.20.0/bevy_asset/io/struct.AssetSourceBuilder.html, https://github.com/bevyengine/bevy/blob/main/examples/asset/custom_asset_reader.rs)
- Implication for #26 **(inference)**: a pack-asset cache directory can be exposed as a named asset source (for example `pack://<id>@<hash>/...`) with a custom reader, or simply as a file-based source rooted in the cache with `file_watcher` on during development. Assets arriving over the network after the handshake would be written to the cache before the first load, which sidesteps in-memory streaming. Asset bytes are outside the pack hash (ADR-0010), so integrity checking is a separate decision for #26.
- 0.20 texture compression: `CompressedImageSaver` has a new backend with automatic mipmaps; `compressed_image_saver_universal` keeps the previous Basis Universal behavior. (https://bevy.org/news/bevy-0-20/)
- Gap: the 0.19 and 0.20 notes say nothing on hot reload beyond the planned `.bsn` format; the above comes from feature docs only.

## 6. Compile times

From the official setup page (https://bevy.org/learn/quick-start/getting-started/setup/):

- `dynamic_linking` is "the most impactful compilation time decrease"; do not ship with it.
- Faster linker: LLD (set `linker = "rust-lld.exe"` for `x86_64-pc-windows-msvc`); mold on Linux; the default on macOS is already fast.
- Nightly-only options: Cranelift (about 30% faster per the page; works best on Linux; Wasm builds do not work; macOS builds can crash), and `-Zshare-generics=y`. ADR-0020 pins the toolchain in `rust-toolchain.toml`, so adopting nightly would be a deliberate conflict with a stable pin; stable plus `dynamic_linking` plus LLD is the baseline.
- `opt-level = 1` for the dev profile and `3` for dependencies is for runtime speed in debug builds.
- Workspace implication **(inference)**: because the core never depends on Bevy (ADR-0020), core and server crates build and test without Bevy and stay quick. Only the client crate and the editor, if it embeds Bevy, pay the engine cost. The spike should record actual clean and incremental build times on the target Windows machine.

## 7. Keeping AI-generated Bevy API fresh

- Bevy has an AI policy for contributors (disclosure required; majority AI-generated PRs cannot be merged). It applies to contributing to Bevy, not to using it. (https://bevy.org/learn/contribute/policies/ai/)
- Bevy issue bevyengine/bevy#23867 (open, opened 2026-04-17, labelled S-Needs-Goal and X-Needs-SME) proposes a generated, version-matched `AGENTS.md` for Bevy users; there is no accepted official artifact and no `llms.txt` on bevy.org that I found. (https://github.com/bevyengine/bevy/issues/23867)
- Churn evidence: 75 entries (0.20) and ~120 (0.19) of breakage per release, including renames of core concepts, so model training data from before the pin is routinely wrong. (migration guides above)
- Recommended repo practice **(inference)**, matching the ADR-0020 consequence:
  - A client-crate `AGENTS.md` stating the exact pinned version of Bevy and each third-party crate, the Rust toolchain, a short list of "do not use" removed APIs observed in practice, and the instruction to read local sources before writing.
  - Vendor or pin the version-matched docs in the repo: the 0.19-to-0.20 and prior migration guides, the release notes for the pin, and a copy of the `examples/` directory at the pinned tag (or a pointer to `~/.cargo/registry` source via `cargo doc` / `cargo vendor`). Agents can then grep real signatures.
  - Compile-driven correction: `cargo check` with `-D warnings` already catches removed API; add a small compiling "golden examples" crate (one example each for spawning a chunk, a UI panel, playing a sound, loading a pack asset) that agents copy from and that fails CI when an upgrade breaks it.
  - Pin crates with `=` requirements in the client crate and commit `Cargo.lock`.

## Risks and open questions

- Third-party render and audio crates trail Bevy; `bevy_ecs_tilemap` 0.20 does not exist yet. Prefer first-party features at the pin and delay upgrades until the crates in use publish.
- BSN is new (two releases) and its file format is unshipped; avoid building the UI pipeline on `.bsn` assets.
- `TilemapChunk` stability and exact 0.20 API are unconfirmed in this note (only the main-branch example was read).
- Z-layer ordering semantics, 2D camera and depth behavior were not verified from primary text here; verify in the first spike.
- Slint on Windows with Bevy 0.20 is unverified.
- Rust MSRV rising every release (1.89, 1.95, 1.97.1) means toolchain upgrades are coupled to engine upgrades.

## Suggested spike (small)

One scratch client crate on Bevy 0.20.0 that: draws a 3-level Zone from replica Chunk data via `TilemapChunk`; samples a per-cell light-level texture in a tile material; shows a Bevy UI panel with `EditableText`; plays a cue on an Effect; loads a texture through a custom asset source rooted in a pack cache; and records clean and incremental Windows build times with `dynamic_linking` and LLD.
