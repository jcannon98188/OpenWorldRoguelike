# Network transport: LiteNetLib and alternatives

Research for #5 (part of map #1). Researched 2026-10-18. Every claim cites its owning source; items I could not confirm are marked **Unverified**.

## Recommendation

Use **LiteNetLib 2.1.4** behind an `ITransport` interface, paired with **MemoryPack** (or a hand-rolled `NetDataWriter` codec) for message serialization. It is pure managed code with no dependencies, MIT licensed, actively maintained, and covers what this game needs: reliable ordered delivery for commands, unreliable/sequenced delivery for optional traffic, automatic fragmentation, and opt-in NAT punch-through. Runner-up if LiteNetLib proves inadequate: Valve GameNetworkingSockets (better features and encryption, but native C++ dependency and only third-party C# bindings).

## Context fit

Turn-timeout, server-authoritative, no client prediction, small group: traffic is low-rate commands up and state/events down. Reliable ordered delivery does most of the work, and raw throughput is not a differentiator. What matters is operational: managed-only deployment, maintenance, direct-connect story, and a clean seam.

## Candidates

### LiteNetLib (recommended)
- Latest release 2.1.4, published 2026-05-19 ([GitHub releases API](https://api.github.com/repos/RevenantX/LiteNetLib/releases/latest)). Repo not archived, last push 2026-10-04, MIT, ~3.6k stars ([GitHub repo API](https://api.github.com/repos/RevenantX/LiteNetLib)).
- Delivery methods: reliable ordered, reliable unordered, reliable sequenced (only latest is guaranteed), ordered unreliable with duplicate filtering, plain unreliable; multiple data channels; automatic MTU detection; reliable fragmentation ([README](https://github.com/RevenantX/LiteNetLib), [docs](https://revenantx.github.io/LiteNetLib/index.html)).
- NAT: UDP hole punching via `NatPunchEnabled` (off by default). It needs an introducer/rendezvous server you host; the library does not provide relay ([docs](https://revenantx.github.io/LiteNetLib/index.html)). Relay hosting is out of scope per #1.
- Connection auth: a connection key string/writer passed to `Connect()` and checked server-side (`AcceptIfKey`) ([docs](https://revenantx.github.io/LiteNetLib/index.html)). Useful as a join gate, not as security.
- Encryption: the docs do not mention it. **Unverified**; assume none and treat as a limitation for public internet play.
- Threading: poll model (`PollEvents()`, default `UpdateTime` 15 ms) fits a tick-driven server; `UnsyncedEvents` exists but is experimental ([docs](https://revenantx.github.io/LiteNetLib/index.html)). Use the poll model.
- Dependencies: none ([NuGet](https://www.nuget.org/packages/LiteNetLib)).
- Built-in serializer helpers exist ("Fast packet serializer", `NetDataWriter`/`NetDataReader`) ([README](https://github.com/RevenantX/LiteNetLib)).
- Exact channel count and per-channel semantics are not stated in the pages read. **Unverified**; check `NetManager.ChannelsCount` in source before committing a channel plan.

### .NET 11 compatibility
- The csproj targets `net8.0;netstandard2.1` ([LiteNetLib.csproj](https://raw.githubusercontent.com/RevenantX/LiteNetLib/master/LiteNetLib/LiteNetLib.csproj)). NuGet lists net8.0 and netstandard2.1 as compatible and later TFMs as computed ([NuGet](https://www.nuget.org/packages/LiteNetLib)). A net11.0 app consumes the net8.0 asset by normal forward compatibility. The README mentions .NET 8 optimized socket calls ([README](https://github.com/RevenantX/LiteNetLib)).
- Nothing in the sources states it was tested on .NET 11. **Unverified**: put a loopback smoke test in CI on the pinned SDK.
- It uses `System.Net.Sockets` internally. The #1 constraint is read as: do not write our own socket transport; a UDP library that wraps them is the intended choice.

### Lidgren.Network (rejected)
README states it is "not actively developed any more; only accepting trivial or minor bug fixes"; targets .NET Framework 4.6 era, .NET Core only a roadmap item ([GitHub](https://github.com/lidgren/lidgren-network-gen3)).

### ENet-CSharp (rejected)
Wrapper over native ENet; NuGet ships native binaries for Windows/Linux/macOS x64 only, other platforms require building via CMake ([nxrighthere/ENet-CSharp](https://github.com/nxrighthere/ENet-CSharp)). The SoftwareGuy fork README says the original repo was archived and the fork may lag upstream ([SoftwareGuy/ENet-CSharp](https://github.com/SoftwareGuy/ENet-CSharp)). Native dependency plus fork churn is worse than LiteNetLib for no needed gain.

### Valve GameNetworkingSockets (runner-up)
Reliable and unreliable messages with fragmentation, lanes (priorities/weights), AES-GCM-256 encryption, ICE-based P2P NAT traversal with bring-your-own signaling, BSD-3-Clause, active repo. C++ with C interface; C# only via third-party bindings (ValveSockets-CSharp, Facepunch.Steamworks); Steam Datagram Relay is Steam-only ([GitHub](https://github.com/valvesoftware/GameNetworkingSockets)). Native builds per platform and unofficial bindings make it a heavier dependency. Revisit if encryption or real P2P traversal becomes a requirement.

### Excluded by constraint
gRPC (e.g. MagicOnion is built on gRPC/HTTP2 per [its repo](https://github.com/Cysharp/MagicOnion)), SignalR and raw sockets are excluded by the map.

## Serialization pairing

- **MemoryPack** (Cysharp): source-generator, reflection-free, AOT friendly; packages target `netstandard2.1` and `net7.0`; version-tolerant mode (`GenerateType.VersionTolerant`) for additive protocol evolution, with a CI schema check ([README](https://github.com/Cysharp/MemoryPack)). The README does not mention .NET 10+; **Unverified** on .NET 11, same smoke-test advice. Default mode forbids deleting/reordering members, so a protocol version handshake (use the LiteNetLib connection key) is advisable.
- Fallback: LiteNetLib's own `NetDataWriter`/`NetDataReader` with hand-written or generated codecs. Zero extra dependency, explicit wire format, more boilerplate.

## Suggested seam

`ITransport` (server and client sides) exposing: connect/listen, peer connected/disconnected events, `Send(peerId, ReadOnlySpan<byte>, Delivery)` where `Delivery` is a small enum (ReliableOrdered, Unreliable, Sequenced) plus a channel id, and a `Poll()` pumped by the host loop. The listen server uses the same server code with an in-process loopback transport (no sockets) alongside the LiteNetLib implementation. Serialization stays above the interface so the transport deals only in bytes.

## Open items for later tickets
- Replication protocol (snapshot vs event stream) decides the channel plan; LiteNetLib supports either.
- Reconnection: session resume is the protocol's job, not the transport's.
- Confirm channel count, encryption absence, and .NET 11 behavior with a spike when build starts.
