# 0003. One binary, two runtime modes: Local mode serves only the desktop app, Server mode serves browsers

- Status: Proposed
- Date: 2026-11-11

Decided in [Runtime modes: local and server](https://github.com/dyle71/flock/issues/9).

## Context

flock is to run on one computer for one person by default, and as a self-hosted server for
several people. The maintainer settled one Rust binary with two runtime modes while charting the
wayfinder map; Local mode is the default and is started by the Electron desktop app as a
sidecar. The charting also assumed that a browser on the same computer could use Local mode
against `localhost`.

The research on packaging a Rust sidecar inside Electron (`docs/research/electron-rust-sidecar.md`)
showed what that assumption costs. Every local process and every browser tab can reach a port on
`127.0.0.1`, through CSRF and DNS rebinding as well (Ollama, CVE-2024-28224). Serving a browser
from Local mode would need a `Host` check, a sign-in by launch token, CSRF defenses and a
dedicated host for cookies, since cookies are not isolated by port (RFC 6265 §8.5) — in an
application whose job is to render hostile HTML. The research also found that "the sidecar dies
with Electron" and "the sidecar keeps syncing without Electron" exclude each other, and that a
WebSocket cannot pass a custom-protocol proxy in the Electron main process while a streamed
response can.

The maintainer weighed these on 2026-11-11.

## Decision

flock is one binary with two runtime modes.

- **Local mode** serves only the desktop app: exactly one User, no sign-in. No browser talks to
  it; using flock in a browser means Server mode, also on one's own computer for one User.
- The sidecar dies with Electron. Syncing with the window closed means Electron stays alive in
  the tray; a permanently running backend is Server mode.
- The desktop app loads the SPA from a custom scheme `flock://app`. The Electron main process
  proxies every request to the sidecar on `127.0.0.1`, on a random port, adding a per-launch token
  that the sidecar received over stdin. The sidecar rejects any request without that token or with
  an unexpected `Host`. No renderer ever sees the token.
- Because a WebSocket cannot pass that proxy, the push channel from backend to client is
  Server-Sent Events.
- **Server mode** serves browsers and the desktop app. Users sign in; the binary speaks plain
  HTTP behind a reverse proxy and refuses sign-in over plain HTTP unless `--insecure` is set.
- The desktop app holds exactly one connection, to its own sidecar or to one server. Against a
  server it loads that server's SPA, so SPA and API always ship together.

## Consequences

- Local mode carries none of the defenses a browser-facing loopback service needs beyond the
  token and the `Host` check; a later hardening may move the channel to a Unix socket or named
  pipe.
- The SPA has a stable origin in the desktop app, so its browser storage survives restarts, and
  SPA and API share that origin without CORS or cross-site cookies.
- SSE is the only push channel; the API contract describes it, and nothing relies on a WebSocket.
- A person who wants flock in a browser must run Server mode, with sign-in, even alone on one
  computer.
- Syncing in the background requires the desktop app to keep running in the tray; flock in Local
  mode does not register an OS service.
- Version skew exists only between the desktop shell's small bridge (tray, notifications,
  `mailto:`) and a remote server, not between SPA and API.
- The terms are fixed in `CONTEXT.md` (**Local mode**, **Server mode**, **Admin**).
