# .NET 11 readiness and relevant features

Resolves #7 (part of map #1). Researched 2026-10-18 against first-party Microsoft and xUnit docs. Every claim cites its source; items I could not confirm are marked **Unconfirmed**.

## Short answer

- **Status:** .NET 11 is a **release candidate (RC2, 2026-10-13), not GA**. RC1 and RC2 are labelled "Go-live" (supported in production). GA is "expected in November 2026". [S1][S2]
- **Support term:** Not stated by Microsoft for 11. The policy says odd-numbered releases are STS (2 years) and even-numbered are LTS (3 years); .NET 11 is therefore *probably* STS. **Unconfirmed**: the policy page does not classify 11. [S3]
- **Pinning:** `global.json` with an exact `sdk.version` plus `rollForward`. The local machine has only SDKs 10.0.303 and 10.0.400, so SDK 11 must be installed first. Pinning to an RC SDK expires quickly (RC2 end of support is 2026-11-10), so plan to re-pin to the GA SDK. [S3][S4]
- **Features worth designing around:** C# 15 `union` types and `closed` hierarchies (command/event/effect variants, exhaustive switches), System.Text.Json union and closed-polymorphism support (save files, command logs), JSON Lines output (replay logs), runtime-native async, JIT/bounds-check gains. The new `Random` generics do **not** give a deterministic RNG (see below). [S5][S6][S7]
- **xUnit v3:** supports "NET 8 or later", so net11.0 is within the stated range, and the .NET 11 SDK's built-in test templates support xUnit v3 (defaulting to Microsoft.Testing.Platform). The xUnit docs do not mention .NET 11 explicitly. [S8][S5]
- **Recommendation:** Safe to *design* against .NET 11 / C# 15 now. Defer committing to C# 15 `union` syntax in core public API until GA plus a check that the feature is stable at the chosen `LangVersion` (docs disagree on preview status; see Risks). Keep the target framework a one-line change.

## Release status

| Item | Finding | Source |
|---|---|---|
| Latest build | 11.0.0-rc.2, SDK 11.0.100-rc.2.26504.105, 2026-10-13, "Go-live", contains security fixes | S1 |
| History | preview.1 2026-02-10 through preview.7 2026-08-11; rc.1 2026-09-08 | S1 |
| GA | "expected in November 2026" | S2 |
| Support end for RC2 | 2026-11-10 (RC builds are short-lived) | S3 |
| STS/LTS | Not stated for 11; policy: odd = STS 2 years, even = LTS 3 years | S3 |

## Pinning with global.json

Facts from the global.json reference [S4]:

- `sdk.version` must be a full version (e.g. `11.0.100`); no `11`, `11.0`, wildcards or ranges.
- Default `rollForward` when a version is given is `patch`. `latestFeature` accepts newer feature bands and patches of the same major.minor.
- `allowPrerelease` defaults to `true` outside Visual Studio when unset. Set it explicitly (`false` once GA ships) so CI never silently picks a preview.
- `paths` and `errorMessage` (.NET 10 SDK and later) allow a repo-local SDK and a friendly failure message.
- `test.runner` selects Microsoft.Testing.Platform for `dotnet test`.
- New in the 11 SDK: `dnx` scripts bypass global.json SDK selection (behavioral change) [S9].

Suggested shape while on RC (pin exact, re-pin at GA):

```json
{
  "sdk": {
    "version": "11.0.100-rc.2.26504.105",
    "rollForward": "disable",
    "allowPrerelease": true
  },
  "test": { "runner": "Microsoft.Testing.Platform" }
}
```

At GA, switch to `"version": "11.0.100"`, `"rollForward": "latestFeature"`, `"allowPrerelease": false`. The target framework (`net11.0`) is independent of SDK selection; an SDK older than 11 cannot build net11.0 (general SDK behavior; not separately cited). [S4]

## Features relevant to game-core design

### C# 15 [S6][S10]

- **Union types** (`public union Pet(Cat, Dog, Bird);`): compiler-enforced exhaustive `switch`. The compiler generates a struct implementing `IUnion`, storing contents as a single `object?`, so **value-type cases box**. Custom unions (class- or struct-based, with an optional non-boxing `TryGetValue`/`HasValue` pattern) avoid boxing. The docs list command dispatching and result-or-error as intended uses. `UnionAttribute`/`IUnion` ship in the runtime since .NET 11 Preview 5. "Some features from the proposal specification aren't yet implemented." [S6][S10]
- **Closed hierarchies** (`closed record class GateState`): closed to derivation outside the assembly; switches over direct descendants are exhaustive without a default arm. Good fit for Effect/Action/Event families inside the core assembly. [S6]
- **Collection expression arguments** (`[with(capacity: n), ..xs]`), **extension indexers**, **labeled `break`/`continue`** (useful in grid/FOV loops). [S6]
- **Memory safety** (pointer relaxations, `unsafe(expr)`): requires `LangVersion=preview` and is a multi-release preview effort; avoid in core. [S6]
- **Source generators:** no new C# 15 source-generator language feature is listed; the feature-status page only lists older Source Generator V2 and interceptors (experimental) entries. [S11]

### Runtime and performance [S7]

- **Runtime Async** is still a **preview feature**; opt in with `<Features>runtime-async=on</Features>` (no `EnablePreviewFeatures` needed on net11.0). Runtime libraries are compiled with it on. Relevant to server I/O and host loops, not the synchronous simulation tick. Opt-out property: `UseRuntimeAsync=false`.
- JIT: more bounds-check elimination (including `values[^1]`), devirtualization of generic virtual and default interface methods, switch folding. Free wins for grid and entity-part dispatch; no code changes needed.
- ReadyToRun: `Comparer<T>.Default`/`EqualityComparer<T>.Default` are specialised (docs claim up to 20x for default-comparer collection ops).
- **Minimum hardware raised** to x86-64-v2 (and Windows Arm64 requires LSE); .NET 11 refuses to start on older CPUs. Matters for dedicated-server hosting targets. [S7]

### Serialization (System.Text.Json) [S12]

- Union support: serializes the active case with no wrapper or `$type`; case picked by JSON token kind, or a structural classifier (`JsonUnion(TypeClassifier = ...)`) for multiple object cases. The classifier does not support nested unions, polymorphic cases, or `ReferenceHandler.Preserve`.
- `closed` hierarchies: `[JsonPolymorphic(InferClosedTypePolymorphism = true)]` infers derived types; works with source-generated contexts.
- Source generator: supports non-public accessors/constructors via attributes, `GetTypeInfo<T>()`; `PascalCase` naming policy; member/type-level naming and ignore controls.
- **JSON Lines**: `SerializeAsyncEnumerable` with `topLevelValues: true` writes JSONL (LF endings), suited to an append-only command/replay log. `Utf8JsonWriter.Reset` allows writer pooling for frequent snapshots.
- Hex-float formatting (`ToString("X")`, `NumberStyles.HexFloat`) round-trips floats exactly, useful for golden-file tests.
- The article says it was last updated for RC2 and does not mark these features preview/experimental; details may still change before GA.

### Determinism-relevant caveats [S12][S9][S7]

- `Random.NextInteger<T>`/`NextBinaryFloat<T>` are new generic helpers; the docs do not document the algorithm or cross-version stability. Do **not** use `System.Random` for the seeded world/command-log RNG; own a pinned algorithm (PCG/xoshiro) behind the core's RNG interface. This matches the map's "reproducible from seed plus command log" goal.
- Behavior changes with .NET 11: `decimal`/`BigInteger` <-> floating-point conversions are now correctly rounded; `Math.Round` returns correctly rounded results; unchecked float-to-small-int conversions now saturate. Avoid floating-point in simulation state or pin golden tests per runtime version.
- Recording the runtime version in save and replay headers is cheap insurance.

### Hosting and tooling [S5][S9]

- Breaking change: `IHost.RunAsync`/`StopAsync` now throw when a `BackgroundService` fails (relevant to the Generic Host convention in the map).
- Some Microsoft.Extensions packages are now in the shared framework; the options validation generator reports SYSLIB1205 for explicit `Validate` implementations.
- `dotnet test` gets several improvements; built-in templates support xUnit v3 on Microsoft.Testing.Platform. NativeAOT CLI and the MSBuild server are on by default in the 11 SDK.

## xUnit v3 compatibility [S8][S5]

- xUnit v3 supports ".NET 8 or later" (and .NET Framework 4.7.2+); net11.0 falls in that range. The docs' own examples were run with SDK 10.0.102 and do not mention .NET 11 or RC SDKs. **Unconfirmed in xUnit docs**: explicit net11.0 verification.
- Package: `xunit.v3.mtp-v2` (4.0.0-pre.108 in the docs example, i.e. the example uses a pre-release package). Test projects are `OutputType=Exe`. With a .NET 10+ SDK, `global.json` `test.runner = Microsoft.Testing.Platform` is how `dotnet test` uses MTP.
- The .NET 11 SDK's built-in test templates include xUnit v3 with MTP default. [S5]
- The VSTest dependency on Newtonsoft.Json is removed in the 11 SDK (binary/source breaking for VSTest-based tooling). [S9]
- Verification step before adopting: create a net11.0 xUnit v3 test project on the RC SDK and run `dotnet test`.

## Risks of building on it now

1. **Not GA.** RC2 expires 2026-11-10 and RCs are replaced by newer builds; expect at least one re-pin. Go-live status means supported for production, but APIs can still change. [S1][S3]
2. **Language-version ambiguity.** The Learn .NET 11 overview says C# 15 is the default for net11.0 and lists unions, closed hierarchies etc. The C# 15 page still frames C# 15 as "the latest C# preview", says some union spec features are not yet implemented, and the memory-safety work needs `LangVersion=preview`. The dotnet.microsoft.com download page lists C# 14.0 against rc.2 (while preview.7 listed 15.0), and the Roslyn feature-status page lists the features as targeting "C# 15" without marking them merged. **Resolve empirically** with the RC SDK before committing to `union`. [S1][S2][S6][S11]
3. **Union design limits.** Compiler-generated unions box value-type cases and always compile to a struct over `object?`; the STJ structural classifier has restrictions (no nested unions or polymorphic cases). Hot-path (per-tick) message types may need custom non-boxing unions or plain `closed` records. [S6][S12]
4. **Ecosystem lag.** Third-party libraries (SadConsole, MonoGame, LiteNetLib, SQLite providers, analyzers) were not checked for net11.0 support in this note; most target netstandard or older net, so likely fine, but verify per package. **Not researched.**
5. **STS term.** If 11 is STS, support ends roughly two years after GA, so a move to .NET 12 (LTS) is on the roadmap anyway. **Unconfirmed**. [S3]
6. **Hardware floor** for dedicated servers and developer machines (x86-64-v2). [S7]
7. **Tooling:** the RC2 release notes recommend Visual Studio 2026 Insiders on Windows; VS Code with C# Dev Kit works. [S13]

## Suggested decisions for the map

- Target `net11.0` with `global.json` pinned (exact RC now, `latestFeature` at GA); keep `TargetFramework` and `LangVersion` in `Directory.Build.props` so a fall-back to `net10.0`/C# 14 is a two-line change.
- Do not depend on `System.Random` or floating-point determinism across runtime versions; own the RNG; put the runtime version in save/replay headers.
- Prototype C# 15 `union`/`closed` for command and effect families in a throwaway spike on the RC SDK before the ADR; fall back to sealed record hierarchies plus analyzers if the feature is not stable at GA.
- Use STJ source generation plus JSONL for save and command-log formats, subject to the SQLite/persistence ticket.

## Sources

- S1 https://dotnet.microsoft.com/en-us/download/dotnet/11.0 (release table, go-live status, language versions)
- S2 https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/overview (updated for RC2; GA expected November 2026; C# 15 default)
- S3 https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core (support policy, RC end-of-support dates)
- S4 https://learn.microsoft.com/en-us/dotnet/core/tools/global-json
- S5 https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/overview (SDK and libraries sections)
- S6 https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15
- S7 https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/runtime
- S8 https://xunit.net/docs/getting-started/v3/getting-started
- S9 https://learn.microsoft.com/en-us/dotnet/core/compatibility/11 (partial list; "work in progress")
- S10 https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/union
- S11 https://github.com/dotnet/roslyn/blob/main/docs/Language%20Feature%20Status.md
- S12 https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/libraries (first 100,000 of ~105,000 characters read)
- S13 https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/rc2/11.0.0-rc.2.md

Method note: pages were retrieved through a fetch tool that summarizes content, so recheck specific wording at the source before quoting it in an ADR.
