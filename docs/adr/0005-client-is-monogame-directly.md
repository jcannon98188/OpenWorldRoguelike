# The client is built directly on MonoGame, not SadConsole

> **Superseded by [ADR-0020](0020-the-stack-is-a-rust-workspace.md):** the client is built on Bevy, not MonoGame. The reasoning about dropping SadConsole still stands.

There is one client, and it renders with MonoGame directly. SadConsole was considered and dropped. It would have saved building a cell-grid renderer, but it fixes the client's presentation to a glyph console grid, and we do not want the whole client design to inherit that limit. Direct MonoGame leaves open sprite, tile or hybrid rendering and custom UI, at the cost of building the grid rendering, text and UI layers ourselves.

The core stays presentation-agnostic: it carries no rendering types, so the headless server and tests share it, and the client is a consumer of replicated state and a producer of Commands. This is why dropping SadConsole changes only the client, not the simulation architecture.

Whether the client renders a pure glyph grid, sprites, or both is not decided here.

## Consequences

- No dependency on SadConsole types anywhere. The command and input layer is designed for one client on MonoGame input, not for several front ends.
- Rendering, font and text layout, UI widgets, and audio are client-side work that SadConsole would have provided; they are client design topics, not core ones.
- Supersedes the earlier assumption, recorded in the map's Notes, of "MonoGame with SadConsole on top".
