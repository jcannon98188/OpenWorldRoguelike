# Rust transport: UDP library behind a transport trait

Research for issue #27 (part of #1). Written 2026-10-19. Primary sources only: crates.io metadata, upstream repositories (README, `Cargo.toml`, source), and release pages. Versions and dates below were read on 2026-10-19. Nothing here was benchmarked or run.

## Question

ADR-0020 moved the stack to Rust, so the gameplay transport (previously LiteNetLib, #5, ADR-0005 superseded) is open again. Candidates: `renet`, `laminar`, `quinn` (QUIC), ENet bindings, and a thin custom reliable-ordered layer over UDP. Criteria from the issue:

1. A reliable-ordered stream per Zone (ADR-0010).
2. Encryption, and a handshake carrying the pack list and protocol version (ADR-0010, ADR-0016).
3. Loopback and in-process listen-server fit.
4. NAT and LAN behaviour for hobbyist hosts.
5. Maintenance health and licence.
6. Trait shape for a core with no async runtime (ADR-0020).

Standing constraint: no general-purpose stack (HTTP, gRPC) for gameplay transport. This is why QUIC is judged as a raw transport and not as HTTP/3.

## Constraints taken from the ADRs

- ADR-0010: Effects travel on a reliable-ordered stream per Zone. Turn Offers and refusals travel on the same stream, ordered with the Effects. A Peek is a separate small stream. A party channel is separate again. Every join, reconnect and Zone crossing starts from a fresh baseline. The handshake exchanges the ordered Content Pack list (id, version, hash) and a protocol version, and any mismatch refuses the connection naming the Pack.
- ADR-0016: the pack list is ordered and each entry carries a hash. A mismatch must name the differing Pack, so a single world hash is not enough on the wire. The list must be sent in the handshake.
- ADR-0020: the core is single-threaded and synchronous, with no async runtime. I/O runs on separate threads behind channels. The wire encoding is a separate build-time benchmark, not part of this decision.
- ADR-0014: replay runs with no transport and no wall clock, and the headless server uses an in-memory transport trait implementation. The trait therefore must not require real sockets or `Instant::now()` inside the core.

## Candidates

### renet 2.0.0 (and the `renet2` fork)

- Version and health: `renet` 2.0.0 published 2026-01-20; `bevy_renet` 5.0.0 on 2026-06-20; repo last pushed 2026-06-20; MIT OR Apache-2.0; 25 open issues. Sources: crates.io, the `lucaspoffo/renet` repo.
- Model: message-based channels. The send types are `Unreliable`, `ReliableOrdered` and `ReliableUnordered`. It does fragmentation and reassembly. Channels are unidirectional and configured up front in `ConnectionConfig` (README, `renet/src/channel/mod.rs`).
- Constraint on channels: `channel_id` is a `u8`, so at most 256 per direction, fixed at connection setup. A reliable channel that exceeds `max_memory_usage_bytes` unacknowledged causes a disconnect (doc comment on `ChannelConfig`). Reliable-ordered delivery is per channel, so one channel is one head-of-line domain.
- No async runtime: the core `renet` crate depends only on `bytes`, `log` and `octets` (`renet/Cargo.toml`). The loop is poll-driven: `update(delta_time)`, then read messages, then `send_packets`. The caller supplies time as a `Duration`, which fits the no-wall-clock rule.
- Encryption and handshake come from `renetcode`/`renet_netcode`, an implementation of the netcode 1.02 standard, using `chacha20poly1305`. Its README claims encrypted and signed packets, connect tokens, and protection against zombie clients, man in the middle, amplification and replay.
  - `ServerAuthentication::Secure { private_key }` requires clients to present a `ConnectToken` minted by someone holding the key.
  - `ServerAuthentication::Unsecure` is documented as "useful for testing and prototyping" (`renetcode/src/server.rs`). The unsecure client builds its own token, so it must not be treated as confidential.
  - The connect token carries 256 bytes of `user_data` (`NETCODE_USER_DATA_BYTES`, `renetcode/src/lib.rs`) and a `protocol_id`. That is enough for a protocol version and a 32-byte world hash. It is not enough for the ordered pack list, so the pack list must be an application-level first message on a reliable channel before any baseline.
- In-process: `RenetServer::new_local_client` and `process_local_client` create a client that is connected to the server in memory, documented "use this for testing" (`renet/src/server.rs`). `RenetServer` and `RenetClient` do not own a socket, so the transport layer is replaceable.
- Fork `renet2` (UkoeHB): version 1.0.0 published 2026-10-16, pushed the same day, MIT OR Apache-2.0. It adds in-memory, WebTransport and WebSocket data transports, reworks the ack protocol, and its README says it fixes two netcode security advisories (GHSA-v29p-3vj4-vg4f, GHSA-3x95-24j9-7448). I did not check whether upstream `renet` 2.0.0 carries the same fixes, so treat that as open.
- NAT: UDP only, no traversal. A host behind NAT needs a port forward.

### laminar 0.5.0

- Latest release 2021-05-06, last push 2023-10-18. It is effectively unmaintained.
- The README's own Planned list still has "Handshake Protocol", "Cryptography" and "Congestion Control" unchecked. It has reliable-ordered streams, but no encryption or handshake, which are two of the hard criteria. It also warns "not fully stable yet".
- Rejected.

### quinn (QUIC)

- Version and health: `quinn` 0.11.12 and `quinn-proto` 0.11.19, published 2026-09-14 and 2026-09-30. Repo pushed 2026-10-18, 5.3k stars, MIT OR Apache-2.0. The `main` branch is already at 0.12.0, so a breaking release is coming. `quinn-proto` MSRV is 1.85 per crates.io.
- Fit for criterion 1: QUIC streams are independent and reliable-ordered with no head-of-line blocking between streams. That matches a stream per Zone, a Peek stream and a party stream, with no channel-id limit (the default is 100 concurrent streams of each kind and is configurable, `quinn-proto/src/config/transport.rs`).
- Encryption is built in: TLS 1.3 through `rustls`. Server identity needs a certificate. The README describes self-signed certificates via `rcgen` with trust-on-first-use or custom verification, so a hobbyist host can ship a pinned fingerprint in an invite. This is a real design task that renet's token model does not have (renet needs a token issuer instead).
- Handshake: ALPN can carry the protocol version. The pack list would still be the first application message on a stream.
- Async runtime: the high-level `quinn` API is future-based and needs `runtime-tokio` (default) or `runtime-smol` (`quinn/Cargo.toml`). This conflicts with ADR-0020 unless it is confined to an I/O thread behind channels, which ADR-0020 does allow. The alternative is `quinn-proto`, which the README describes as a "deterministic state machine" with "no I/O", suitable for custom event loops. Driving it directly means writing the socket loop, timers, TLS setup and stream bookkeeping ourselves.
- NAT: none. A UDP port forward is required. `iroh` 1.3.0 (n0-computer, published 2026-09-28, MIT OR Apache-2.0) layers hole-punching and public relay fallback on QUIC (it uses `noq`, a QUIC fork) and requires `tokio`. It is a possible later option for NAT, behind the same trait, but it adds third-party relay infrastructure and an async runtime.

### ENet bindings

- `enet` 0.3.0 (futile/enet-rs, wraps C ENet): last release 2022-03-30, last push 2022-06-11. Stale. `enet-sys` 1.0.5 (2026-03-16) is a raw C binding.
- `rusty_enet` 0.4.0 (2025-04-27, MIT): ENet 1.3.18 transpiled to Rust. Its README says it is "not ready for serious use" despite thorough testing, and quotes ENet itself omitting "authentication, lobbying, server discovery, encryption". So there is no encryption, and no handshake beyond ENet's own.
- ENet's protocol header caps channels at 255 per peer (`ENET_PROTOCOL_MAXIMUM_CHANNEL_COUNT`).
- Fails the encryption criterion and is a poor health fit. Rejected.

### Custom reliable-ordered layer over UDP

Would have to rebuild acks, resend, fragmentation, congestion control, connection tokens, replay protection and encryption. That is what `renetcode` and `renet` already provide, and none of the rest of the design benefits. Rejected unless every library above fails a hard requirement.

## Comparison

| Criterion | renet 2 / renet2 | quinn | laminar | rusty_enet |
|---|---|---|---|---|
| Reliable-ordered per Zone | Channels (u8 id, fixed at setup, shared head-of-line per channel) | Native streams, no cross-stream blocking | Yes | Yes (255 channels) |
| Encryption | netcode secure mode (needs token issuer) | TLS 1.3 (needs cert or pinned key) | None | None |
| Handshake data | 256 B token `user_data` plus app message | ALPN plus app message | None | None |
| No async runtime | Yes (poll-driven) | Only via `quinn-proto` or an I/O thread | n/a | Yes |
| In-process | `new_local_client`; renet2 adds memory sockets | Own implementation | n/a | n/a |
| NAT | Port forward | Port forward (iroh adds punching) | Port forward | Port forward |
| Health | Active (releases in 2026) | Very active | Dead | Slow |
| Licence | MIT/Apache-2.0 | MIT/Apache-2.0 | MIT/Apache-2.0 | MIT |

## Recommendation

1. Use `renet` (or its `renet2` fork) as the first implementation, behind our own transport trait, with the netcode secure mode. It is the only candidate that is poll-driven with no async runtime, ships encryption and replay protection, and has a documented in-memory client.
2. Map ADR-0010 streams onto a small fixed pool of `ReliableOrdered` server-to-client channels (own Zone, Peek, party, plus a few spare for neighbours), with the Zone-to-channel assignment decided per connection. Per-Zone head-of-line blocking is acceptable at turn-based rates. Treat the u8 channel cap and the "reliable overflow disconnects" rule as design limits.
3. Put the protocol version and world hash in the connect token `user_data`, and send the full ordered pack list as the first reliable message, so a mismatch can name the Pack (ADR-0016). The hobbyist host mints the token itself and hands out an invite; for LAN play, a local token issuer works the same way.
4. Keep `quinn` as the named fallback if channel multiplexing proves painful. Its stream model matches ADR-0010 better, but it costs an I/O thread with an async runtime (or a hand-written `quinn-proto` driver) and a certificate-trust story.
5. Do not use `laminar`, ENet bindings or a custom UDP layer. Plan on port forwarding for NAT; revisit `iroh` only if hosts cannot forward.

## Trait shape (sketch, not a decision)

The trait should be synchronous and poll-based, and time should be passed in. It carries opaque byte messages tagged with a lane:

```rust
pub trait Transport {
    fn update(&mut self, dt: Duration);                       // drive I/O, no wall clock inside the core
    fn poll_event(&mut self) -> Option<TransportEvent>;       // Connected / Disconnected / Refused(reason)
    fn send(&mut self, to: ConnId, lane: Lane, msg: Bytes);   // Lane: Zone(n) | Peek | Party | Control
    fn recv(&mut self, from: ConnId, lane: Lane) -> Option<Bytes>;
    fn flush(&mut self);
}
```

The `Lane` abstraction lets the renet implementation pool lanes onto channels and a QUIC implementation map each lane to its own stream. The in-memory implementation for headless and replay (ADR-0014) is trivial.

## Open questions and caveats

- Whether upstream `renet` 2.0.0 carries the netcode advisory fixes listed by `renet2` is unverified.
- The `bevy_renet` 5.0.0 / Bevy version compatibility was not checked. The client can use `renet` directly without `bevy_renet`.
- LAN discovery is not provided by any candidate in the sources read; it would be a separate mechanism.
- No throughput, latency or maximum message size testing was done; the wire encoding benchmark in ADR-0010 is the right place for that.

## Sources

- crates.io API pages for `renet`, `renet2`, `bevy_renet`, `laminar`, `quinn`, `quinn-proto`, `enet`, `rusty_enet`, `enet-sys`, `iroh` (read 2026-10-19).
- Repo metadata and files: `lucaspoffo/renet` (README, `renet/Cargo.toml`, `renet/src/channel/mod.rs`, `renet/src/server.rs`, `renetcode/src/lib.rs`, `renetcode/src/server.rs`, `renetcode/src/client.rs`, `renetcode/README.md`); `UkoeHB/renet2` (README, `renetcode2/NETCODE_EXTENSIONS.md`); `amethyst/laminar` (README); `quinn-rs/quinn` (README, `quinn/Cargo.toml`, `quinn-proto/src/lib.rs`, `quinn-proto/src/config/transport.rs`); `jabuwu/rusty_enet` (README); `lsalzman/enet` (`include/enet/protocol.h`); `n0-computer/iroh` (README, `iroh/Cargo.toml`).
