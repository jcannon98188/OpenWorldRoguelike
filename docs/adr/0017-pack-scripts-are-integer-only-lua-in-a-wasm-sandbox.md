# Pack scripts are integer-only Lua running in a Wasm sandbox

> **Amended by [ADR-0020](0020-the-stack-is-a-rust-workspace.md):** the language is Rust, so Parts are Rust (an `enum` of built-ins plus a script variant) and the `Wasmtime` host is the native Rust crate, not a .NET binding. The design below is otherwise unchanged.

Content Pack scripts (ADR-0015) are written in Lua and run on the server inside Wasmtime, through the `Wasmtime` .NET binding. The Lua interpreter is a first-party engine, compiled once to a WebAssembly module and shipped with the server; pack authors never build Wasm. Wasmtime is the sandbox: the engine module is linked only with the game's host imports, with no WASI, so a script can reach nothing the host did not hand it. Each call runs with a fuel budget (deterministic, never wall-clock epochs) and a hard memory cap, clamped by server policy over the Pack manifest's requested limits. Memory growth is limited to a fixed ceiling so success or failure is deterministic, and running out of fuel or memory aborts the call with a recorded error that is part of the simulation, so replay (ADR-0014) fails the same way.

One language and one host serve every script entry point: Part handlers, generation stages (ADR-0012) and Elapse hooks (ADR-0009). They differ only in the host imports linked and in their budgets. Stages get large budgets, handlers small ones.

The engine is an integer-only Lua 5.4 dialect enforced by how it is built, not by a linter. There is no float subtype, no `math.random`, no `os`, `io`, `load`, `dofile` or `string.dump`, and iteration order is deterministic. Randomness and queries come only from host imports: the zone RNG stream (ADR-0013) and read access to game state. A script that uses a removed construct fails at load or at run with an error naming the Pack and line, so drift cannot be silent. Because the engine itself is trusted code compiled with float instructions, the load-time float-opcode validator applies to Pack-supplied Wasm, not to the first-party engine module, which is pinned by hash and covered by the world hash.

Authors write plain `.lua` files in the Pack folder with no build step. The shared loader and validator (ADR-0016) syntax-check scripts and run them in this sandbox, so a script that loads in the editor loads on the server. A Pack ships LuaLS-style annotations for the host API so ordinary editors give completion. Raw Wasm guest modules are not supported at launch; the `IScriptHost` seam keeps them possible without changing anything above it.

Considered: raw Wasm only, with authors compiling AssemblyScript or Rust (a toolchain and build step for hobbyists); native Lua through NLua/KeraLua (in-process native C with deny-list sandboxing, so an interpreter bug is a host compromise); Jint (documented as not a security boundary, double-based numbers, only statement counts are deterministic); managed Lua interpreters (floats and no hard memory cap as documented); Roslyn scripting and collectible `AssemblyLoadContext` (no security boundary on modern .NET); different languages per entry point (two sandboxes and two things to learn); a pack linter in place of a restricted engine (dynamic Lua escapes static checks).

## Consequences

- The engine build is ours to own: the integer-only Lua port, its deterministic iteration, and the update path for upstream Lua fixes. It is the largest cost of this decision.
- If the Lua-in-Wasm engine fails its spike (cold start, per-call overhead, memory per instance), the fallback is raw Wasm authoring on the same Wasmtime host, which changes only the authoring story.
- Spike before build, none yet run: `Wasmtime` on .NET 11 and its native asset RIDs, Query/Before/After call overhead on the dispatch hot path with pooled instances, and byte-identical results across platforms. These are also named by the research findings.
- Script state encoding, the host import table and the error contract are the script API's (not yet decided); this ADR fixes only the language, the host and the budgets model.
- Pack-supplied Wasm, if ever allowed, needs its own validator for floats and growth; out of scope here.
