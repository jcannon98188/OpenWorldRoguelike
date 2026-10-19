# The stack is a Rust workspace

The core, headless server, client and content editor are one Cargo workspace in Rust. The earlier .NET 11 and C# 15 choice rested on the owner's comfort; code is written by AI tooling, so the stack optimizes for performance, memory layout and control instead. Nothing was built under the old stack, so the cost of the change is edits to documents. The architecture decisions (Effects, Zones, time, replication, catch-up, generation, determinism, modding) are language-independent and carry over unchanged.

**Core.** Parts are Rust. Built-in Parts are a closed `enum` with one script variant, dispatched by `match`, so there is no dynamic dispatch on the hot path. Part state lives in typed per-kind arenas indexed by EntityId, and an Entity holds the ordered list of its Part handles. This keeps the event semantics, the Effects-only rule and the total handler order of ADR-0001, with a data-oriented layout. A real ECS was rejected because it would reopen the cascade and ordering decisions of ADR-0001, 0002 and 0006. The simulation of a Zone is single-threaded and synchronous, with no async runtime in the core; I/O runs on separate threads behind channels (ADR-0009, persistence).

**Client.** The client is built on Bevy, pinned to an exact version and confined to the client crate; the core never depends on it. Bevy releases breaking changes about every three months, so upgrades are deliberate and scheduled. The client renders 2D with depth (layered z-levels, lighting). Game UI is code or markup with a small widget kit and hot reload; whether it is Bevy UI or an embedded toolkit is decided in the UI and audio ticket. Gum was considered and dropped because it is C#-only with no Rust runtime. Debug overlays and the content editor may use `egui`.

**Scripting.** ADR-0017 stands: integer-only Lua 5.4 compiled to Wasm and run in the native `wasmtime` crate with fuel and memory budgets. Only the host binding changes. Rust-compiled Wasm guest modules remain a later option behind the same host seam.

**Conventions.** The toolchain is pinned in `rust-toolchain.toml`; warnings are errors (`clippy` with deny); tests run with `cargo-nextest`, with `proptest` for contract vectors. Sim state never iterates a `HashMap` with a random hasher: ordered maps or a fixed hasher only, extending the determinism rules of ADR-0001 and ADR-0013.

Considered: keeping .NET 11 (comfort, not performance); a non-Rust client on a Rust core (two toolchains, and the shared apply-Effect function of ADR-0002 would need binding); Bevy ECS for the simulation; a hand-assembled `winit` + `wgpu` client (rejected once the render style was 2D with depth and the UI and asset needs were counted); Slint or a Gum runtime port for UI (deferred to the UI ticket).

## Consequences

- Library-level choices made under .NET are open again and have research tickets: transport, persistence with the Zone blob encoding, serialization (replacing MemoryPack/MessagePack), Bevy fitness for the client, and the Wasmtime spike of ADR-0017.
- ADR-0005 is superseded. ADR-0001, ADR-0010, ADR-0012, ADR-0014 and ADR-0017 had their language mentions amended.
- AI tooling tends to write stale Bevy API, so the client crate needs pinned version notes and examples in the repo's agent docs.
