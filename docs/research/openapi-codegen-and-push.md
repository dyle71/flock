# OpenAPI codegen from Rust to TypeScript, plus a push channel

Research for [issue #6](https://github.com/dyle71/flock/issues/6) (wayfinder map #1). Written 2026-11-11 against
primary sources only (docs.rs, crates.io and npm registry metadata, the projects' repositories and docs, the WHATWG,
IETF and OpenAPI/AsyncAPI specifications). Every fact carries a bracketed source tag resolved in the list at the end.
Versions, licenses and dates are as of the writing date. The decision itself is a separate ticket.

Starting point: klicker.ing's ADR 0011 settled a **code-first** contract: the server's route definitions generate a
committed `docs/api/openapi.json` (OpenAPI 3.1.0) and a committed `schema.d.ts`; the browser calls the API only
through `openapi-fetch` with types from `openapi-typescript`; a guard test fails when either generated file is
stale; cookies only; errors as RFC 9457 `application/problem+json` [K1][K2]. klicker.ing has no push channel: its
screens poll [K3]. flock replaces the TypeScript server (Hono + `@hono/zod-openapi`) with an async Rust binary, so
the question is which Rust crate takes the place of `@hono/zod-openapi`, and what pushes new-mail, sync-progress and
folder-change events.

## 1. Rust side: who produces the OpenAPI document

| Fact                      | utoipa (+ utoipa-axum)                                                                                                                                                                | aide                                                                                                                                         | poem-openapi                                                                                               | dropshot                                                                                                                                           |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Current version           | utoipa 6.0.0 (2026-09-23), utoipa-axum 0.3.0 [R1][R2]                                                                                                                                 | 0.15.1 stable; 0.16.0-alpha.4 on main (2026-04-14) [R1][R6]                                                                                  | 5.1.16 [R1]                                                                                                | 0.18.0 (2026-10-30) [R1]                                                                                                                           |
| License                   | MIT OR Apache-2.0 [R1][R3]                                                                                                                                                            | MIT OR Apache-2.0 [R1][R5]                                                                                                                   | MIT OR Apache-2.0 [R1]                                                                                     | Apache-2.0 [R1]                                                                                                                                    |
| HTTP framework            | axum (plus actix-web, rocket and others) [R3]                                                                                                                                         | axum only; actix-web dropped in 0.5 "due to lack of developer capacity" [R4]                                                                 | poem only [R7]                                                                                             | its own server on hyper; not axum [R9]                                                                                                             |
| Approach                  | code-first; `#[utoipa::path(...)]` attributes on handlers, `#[derive(ToSchema)]` on types; `OpenApiRouter` + `routes!` register a handler once for both router and doc [R3][R10] | code-first; `ApiRouter::api_route` infers operations from extractor and response types; types derive schemars `JsonSchema` [R4]             | code-first; `#[OpenApi]` on an impl block; "if your code compiles, it is fully compliant" [R7]             | code-first, with API traits (`#[dropshot::api_description]`) that produce the document without an implementation, checked in CI by an API manager [R9] |
| OpenAPI version emitted   | **3.1.0 by default**; **3.2.0 opt-in** via `OpenApiVersion::Version32`, "Opt-in until OpenAPI 3.2 tooling support is widespread" [R2][R11]                                           | "Currently only Open API version `3.1.0` is supported" [R4]                                                                                  | **3.0.0** (hard-coded `OPENAPI_VERSION`) [R8]                                                              | **3.0.3** (hard-coded) [R9a]                                                                                                                       |
| Schema engine             | own derive (`ToSchema`), not schemars [R3]                                                                                                                                            | schemars 0.9 on 0.15; schemars 1.x on 0.16-alpha [R4][R6]                                                                                    | own derive [R7]                                                                                            | schemars, via `openapiv3` [R9]                                                                                                                     |
| SSE in the document       | 6.0 adds `item_schema = ...` on `responses(...)`, mapping to 3.2's `itemSchema` "for streaming/event-based media types such as `text/event-stream`" [R2][R12]                       | not mentioned [R4]                                                                                                                           | `EventStream<T>` payload, content type `text/event-stream`, schema `items: T` [R8a]                        | not supported for responses (request streaming and NDJSON constant only) [R9]                                                                      |
| WebSocket in the document | no                                                                                                                                                                                    | `axum-ws` feature gate (route is documented, messages are not) [R4][R6]                                                                      | `websocket` feature on poem; not in the document [R7]                                                      | `#[channel]` endpoints, marked with vendor extension `x-dropshot-websocket` [R9][R9b]                                                              |
| Activity                  | ~4.1k stars, pushed 2026-10-20; ~22 M downloads in the last 90 days [R1][R13]                                                                                                         | ~0.7k stars, pushed 2026-09-21; last stable release 2025-08-19 [R1][R13]                                                                     | poem repo pushed 2026-11-06; poem-openapi crate last updated 2025-07-28 [R1][R13]                          | pushed 2026-11-09; maintained by Oxide Computer for their own product [R13]                                                                        |

Notes:

- **utoipa** is the de facto default for axum: two orders of magnitude more downloads than aide [R1]. Its weak spot
  is duplication: the path, method and parameters live both in the axum routing and in the `#[utoipa::path]`
  attribute; `utoipa-axum` removes the routing half by registering each handler once and splitting the result into
  an `axum::Router` and an `OpenApi` (`split_for_parts()`) [R10]. The document is an ordinary value
  (`ApiDoc::openapi().to_pretty_json()`), so writing it to a committed file from a test or a small binary, the way
  klicker.ing's `tools/generateOpenApi.ts` does, needs no running server [R3][K1].
- **aide** infers more from types (less duplication) but needs `*_with` calls for descriptions, examples and
  status-specific responses; handlers must return `impl IntoApiResponse` instead of `IntoResponse` [R4]. It is a
  one-maintainer project whose next major has been in alpha since before April 2026 [R6][R13].
- **poem-openapi** and **dropshot** each require leaving axum and are pinned to OpenAPI 3.0.x [R8][R9a].
  poem-openapi is the only crate here that already describes SSE in a 3.0/3.1 document out of the box [R8a];
  dropshot is the only one that marks WebSocket endpoints in the document [R9b].
- **Spec-first** (write the YAML, generate the server) is thin in Rust: OpenAPI Generator's `rust-axum` generator is
  marked **BETA** and lists callbacks as unsupported [R14]; Oxide's `progenitor` (MPL-2.0) generates Rust *clients*
  from OpenAPI 3.0.x, not servers [R15]. klicker.ing rejected contract-first because the schemas would exist twice
  [K1]; in Rust, the serde types would be the second copy.

## 2. TypeScript side: types, client, TanStack Query hooks

| Fact                         | openapi-typescript + openapi-fetch + openapi-react-query                                                                                            | @hey-api/openapi-ts                                                                                                                                                                                                                     | orval                                                                                                  | kubb                                                                                                                                                                           |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Current version              | 7.13.0 / 0.17.0 / 0.5.4, all released 2026-02-11 [T1][T2]                                                                                          | 0.99.0 (2026-06-22); only `next` builds since [T1]                                                                                                                                                                                      | 8.47.0 (2026-11-07) [T1][T7]                                                                           | 5.18.1 (2026-11-11) [T1][T9]                                                                                                                                                   |
| License                      | MIT [T1]                                                                                                                                            | MIT [T1][T5]                                                                                                                                                                                                                            | MIT [T1]                                                                                               | MIT [T1]                                                                                                                                                                       |
| What it generates            | types only (`paths`, `components`), no runtime; `openapi-fetch` is a typed `fetch` wrapper with middleware; `openapi-react-query` is a 1 kB TanStack Query v5 wrapper over it [T3][T4][T6] | SDK functions + types, bundled fetch/axios/ky/ofetch clients, a TanStack Query v5 plugin (query keys, query/infinite/mutation options), zod/valibot validators, transformers, MSW mocks [T5][T5a]                                       | models, request functions and hooks for React/Vue/Svelte/Solid/Angular Query, zod schemas, MSW mocks; fetch or axios [T7]                 | types, fetch/axios clients, TanStack Query and SWR hooks, zod, faker, MSW; Vite plugin `unplugin-kubb` [T9]                                                                     |
| OpenAPI input                | "Supports OpenAPI 3.0 and 3.1" [T3]; 3.2 `itemSchema` PR (#2683) open since 2026-03-13 [T3a]                                                        | parsers for 2.0.x, 3.0.x and `>=3.1.0` (3.2 goes through the 3.1 parser; `itemSchema` handling not documented) [T5b]                                                                                                                    | v3 and Swagger 2; internally upgrades 2.0 to 3.1 [T7][T8]                                              | "OpenAPI 2.0, 3.0, 3.1, and 3.2"; 3.2 `itemSchema` reading merged 2026-11-05 [T9][T9a]                                                                                        |
| SSE                          | none generated; `parseAs: "stream"` returns the raw body stream [T4]                                                                                | since 0.81.0, `text/event-stream` responses become typed async-iterable streams; fetch-based, sends `Last-Event-ID` on reconnect, honors `retry:`, configurable backoff and max attempts, custom headers allowed [T5c][T5d] | NDJSON streaming guide; no SSE support found [T8a]                                                      | 3.2 `itemSchema` is read by the adapter; generated SSE clients not documented [T9a]                                                                                           |
| Stability                    | 0.x for fetch/react-query; low release cadence in 2026 (last release Feb, dependency bumps since) [T2]                                              | "initial development", pin an exact version, migration notes per breaking release [T5]                                                                                                                                                  | 8.x, weekly releases [T7]                                                                              | 5.x since 2026-08-17, near-daily releases [T9]                                                                                                                                |
| klicker.ing precedent        | yes: `openapi-typescript` 7.13.0 + `openapi-fetch` 0.17.0 against a 3.1 document [K1]                                                              | no                                                                                                                                                                                                                                      | no                                                                                                     | no                                                                                                                                                                             |

Notes:

- The openapi-typescript family generates no code at runtime: hooks are `$api.useQuery("get", "/path", {...})`
  calls typed against `paths` [T6]. Query keys are derived by the wrapper, not generated per endpoint.
- hey-api, orval and kubb generate one function or hook per operation, which means `operationId`s become public
  TypeScript names; utoipa derives the `operationId` from the handler function name unless overridden [R3a].
- All four are MIT and run as development-time tools; none is shipped to users except the small runtime clients.

## 3. End-to-end pipelines known to fit together

1. **utoipa (3.1) → openapi-typescript → openapi-fetch → openapi-react-query.** The direct port of klicker.ing's
   ADR 0011: the Rust side emits 3.1.0 by default [R11], the TS side reads 3.1 [T3], and klicker.ing already runs
   the TS half against a 3.1 document [K1]. SSE is outside the generated surface: the stream is consumed by hand
   (`EventSource`, or `parseAs: "stream"` [T4]) and its payload types come from `components.schemas`.
2. **utoipa (3.1) → @hey-api/openapi-ts with `@tanstack/react-query` plugin.** Same document; generated per-endpoint
   query options and keys [T5a], plus typed SSE streams for operations whose response content type is
   `text/event-stream` [T5c]. Costs: a 0.x tool that asks to pin exact versions [T5].
3. **utoipa 6 with `Version32` + `item_schema` → kubb 5.18+.** The only pair where both ends explicitly handle
   OpenAPI 3.2's `itemSchema` for SSE [R2][R12][T9a]. Both halves of this landed in the last two months, and
   openapi-typescript cannot read the 3.2 field yet [T3a].
4. **aide (3.1) → any of the above.** Same TS side; aide replaces utoipa on the Rust side [R4].
5. **dropshot (3.0.3) → progenitor / any TS generator.** Coherent, but leaves axum and OpenAPI 3.1 [R9a][R15].

All Rust crates and TS tools above are MIT, Apache-2.0 or MPL-2.0, each listed by the FSF as compatible with
GPLv3 (Apache-2.0 with GPLv3 only; MPL-2.0 through its section 3.3) [L1].

## 4. Push channel: SSE vs WebSocket

| Fact                            | Server-Sent Events (`EventSource`)                                                                                                                                                                                                                                                                         | WebSocket                                                                                                                                                                                                                                                                  |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Direction                       | server to client only; commands stay on the REST API                                                                                                                                                                                                                                                       | both directions                                                                                                                                                                                                                                                            |
| axum support                    | `axum::response::sse::{Sse, Event, KeepAlive}`; `Event::id`, `event`, `retry`, `json_data`; `KeepAlive` default 15 s, an empty comment [P1][P2][P3]                                                                                                                                                        | `axum::extract::ws::WebSocketUpgrade` (feature `ws`, `tokio-tungstenite` underneath) [P4]                                                                                                                                                                                 |
| Browser API                     | `new EventSource(url, { withCredentials })`; no way to set request headers [P5]                                                                                                                                                                                                                            | `new WebSocket(url, protocols)`; no way to set request headers [P6]                                                                                                                                                                                                       |
| Auth by cookie                  | cookies go with the request; cross-origin requests go through CORS, credentials only with `withCredentials` [P5]                                                                                                                                                                                         | cookies may be sent in the handshake [P7 §4.1]; no CORS, so the server "SHOULD verify the Origin field" and answer 403 [P7 §10.2]; the protocol "doesn't prescribe" an auth method [P7 §10.5]                                                                           |
| Auth by bearer                  | not with native `EventSource`; needs a fetch-based client (hey-api's generated SSE client sets headers [T5d]; `@microsoft/fetch-event-source` is MIT but last released 2021-04 [T10]) or a token in the query string                                                                                     | not in the handshake from a browser; token in the query string, in the `Sec-WebSocket-Protocol` value, or as the first message [P6]                                                                                                                                      |
| Reconnect and resume            | built in: the browser reconnects after a network error and sends `Last-Event-ID` with the last `id:` it saw; `retry:` sets the delay; a `204` stops reconnection; a non-200 status or wrong content type fails the connection for good [P5]                                                              | none in the protocol; the application writes backoff, resubscription and replay                                                                                                                                                                                         |
| Proxies                         | an ordinary long GET; legacy proxies "drop HTTP connections after a short timeout", so send a comment "every 15 seconds or so" [P5]; nginx buffers responses unless `proxy_buffering off` or the response carries `X-Accel-Buffering: no`; `proxy_read_timeout` 60 s by default [P8] | `Upgrade` is hop-by-hop, so a reverse proxy must forward it explicitly (`proxy_set_header Upgrade` / `Connection "upgrade"`); idle connections close after 60 s unless the timeout is raised or the server pings [P9]                                                 |
| Connection limit                | over HTTP/1.1 "the limit is per browser + domain ... (6)", across all tabs, "Won't fix" in Chrome and Firefox; over HTTP/2, streams are negotiated, default 100 [P10]                                                                                                                                      | not subject to the HTTP/1.1 six-connection pool                                                                                                                                                                                                                           |
| In OpenAPI                      | 3.2.0 (2025-09-19) defines `itemSchema` for sequential media types and a section on `text/event-stream`, with a generic SSE event schema and an example [S1]; in 3.0/3.1 only as an untyped or ad-hoc `schema` [S1]                                                                                      | not covered by OpenAPI [S1]                                                                                                                                                                                                                                               |
| In AsyncAPI                     | possible, but the `http` binding is not SSE-specific                                                                                                                                                                                                                                                       | AsyncAPI 3.1.0 (2026-01-31) [S2]; WebSockets binding 0.1.0: the channel is the connection; `method`, `query`, `headers` for the handshake [S3]. Rust generator: `asyncapi-rust` 0.5.0 (MIT OR Apache-2.0, first published 2025-11) [S4]; TS types: Modelina (Apache-2.0) [S5]; kubb's AsyncAPI adapter is an open issue [S6] |
| TanStack Query integration      | events invalidate or patch cached queries; `experimental_streamedQuery` turns an `AsyncIterable` into a query whose data is the chunks so far [P11]                                                                                                                                                       | same, from `onmessage`                                                                                                                                                                                                                                                    |

Browser and Electron: Electron's renderer is Chromium, so both APIs exist there as in a browser. What differs is
the origin the SPA is loaded from:

- If the Electron window loads the SPA from the Rust binary's own HTTP origin (as a browser would), REST calls,
  `EventSource` and the WebSocket handshake are same-origin and cookies behave as in the browser.
- If the SPA is loaded from `file://` or a custom scheme, the API is cross-site. Electron recommends a custom
  protocol over `file://` [P12]; cookies and web storage work on a custom scheme only when it is registered as
  `standard` [P13]; a cookie for a cross-site request needs `SameSite=None; Secure` [P14] plus CORS with
  credentials. `Secure` is relaxed on localhost [P14].

Local mode on plain `http://127.0.0.1` means HTTP/1.1: browsers negotiate HTTP/2 through TLS ALPN (`h2`), and the
cleartext `h2c` upgrade "was never widely deployed and is deprecated" [P15]. That puts the six-connection limit
[P10] on every SSE stream opened from browser tabs against the local server; Electron is one window per app
unless flock opens several.

## 5. Implications for flock

Facts and trade-offs only; no decision.

- **Rust crate.** Staying on axum and on OpenAPI 3.1 (klicker.ing's version [K2]) leaves utoipa and aide. utoipa
  has the larger user base, a fresh 6.0 major, the default 3.1 output and an opt-in path to 3.2 for typed SSE
  [R2][R11]; it costs per-handler attributes that restate what axum already knows [R3][R10]. aide infers more from
  types and derives schemas through schemars, but its next major has sat in alpha for months [R4][R6]. poem-openapi
  and dropshot mean a different web framework and OpenAPI 3.0 [R8][R9a].
- **Code-first fits klicker.ing's practice.** The committed-document-plus-freshness-guard pattern of ADR 0011 [K1]
  ports as is: a Rust test or `xtask` writes `docs/api/openapi.json`, `openapi-typescript` (or another generator)
  writes the TS file, and CI fails on a diff. Spec-first has no mature axum server generator [R14].
- **TS generator.** openapi-typescript + openapi-fetch is what the maintainer already runs [K1] and is the lightest
  (types only), but has had no release since February 2026 and no OpenAPI 3.2 support yet [T2][T3a].
  openapi-react-query covers TanStack Query hooks with one tiny wrapper [T6]. hey-api adds generated query keys,
  options, zod validators and an SSE client with resume, at the price of a 0.x tool that breaks between releases
  [T5][T5c]. kubb is the only TS generator that has documented 3.2 `itemSchema` reading [T9a]. orval is actively
  released but has no SSE support [T8a]. TanStack Form/Table need no generator support; zod schemas from hey-api,
  orval or kubb could feed TanStack Form validation.
- **Push payloads in the contract.** Whatever the channel, the event payloads (new mail, sync progress, folder
  change) can be `components.schemas` in the same OpenAPI document, so the TS side gets their types from the one
  generator. Describing the stream endpoint itself as typed SSE needs OpenAPI 3.2 [S1]; with 3.1 the endpoint is
  documented as `text/event-stream` with a loose schema. A WebSocket channel is invisible to OpenAPI and needs a
  separate AsyncAPI document, whose Rust and TS tooling is young [S2][S4][S6].
- **SSE vs WebSocket for the three events.** All three flow server to client; user actions (mark read, move,
  sync now) stay REST calls either way. SSE brings reconnect and `Last-Event-ID` resume from the browser for free
  [P5]; if the server stamps events with a monotonic id and can replay or answer "resync" for a gap, resume after
  sleep or network loss is cheap. WebSocket needs that machinery written by hand, but is not bound by the HTTP/1.1
  six-connection limit [P10] that local mode (plain HTTP on loopback) runs into with several browser tabs [P15].
  Both pass through nginx-style reverse proxies in server mode only with configuration: buffering off for SSE,
  Upgrade forwarding for WebSocket, and a heartbeat under the 60 s read timeout for both [P8][P9].
- **Auth.** With cookies (klicker.ing's choice [K1]), both channels work from browser and Electron as long as the
  SPA is served from the server's origin. SSE inherits CORS protection; a cookie-authenticated WebSocket needs an
  explicit `Origin` check, since klicker.ing's same-origin rule covers unsafe methods and the handshake is a GET
  [K1][P7 §10.2]. A bearer token (for example, if Electron or a future third-party client uses tokens) rules out
  native `EventSource` and pushes toward a fetch-based SSE client or a WebSocket with in-band auth [P5][P6][T5d].
- **Electron origin.** Loading the SPA from the Rust binary's HTTP origin keeps one code path for browser and
  Electron; loading it from a custom scheme makes every call cross-site and forces `SameSite=None`, CORS with
  credentials and a `standard` scheme registration [P13][P14].
- **Licenses.** Every candidate is MIT, Apache-2.0 or MPL-2.0, all GPLv3-compatible per the FSF [L1].

## Sources

klicker.ing (read-only, `/home/dyle/Source/headcode.space/klicker.ing/main`)

- [K1] `docs/adr/0011-rest-api-described-by-openapi-no-server-functions.md`
- [K2] `docs/api/openapi.json` (`"openapi": "3.1.0"`, cookie security schemes `userSession`, `deviceSession`)
- [K3] `docs/adr/0006-a-click-travels-through-a-persisted-outbox-screens-poll.md`

Rust

- [R1] crates.io API `https://crates.io/api/v1/crates/<name>` for utoipa, utoipa-axum, aide, poem-openapi,
  dropshot, progenitor, axum (versions, licenses, downloads), queried 2026-11-11
- [R2] utoipa CHANGELOG, 6.0.0 section: https://github.com/juhaku/utoipa/blob/master/utoipa/CHANGELOG.md
- [R3] utoipa README: https://github.com/juhaku/utoipa (code-first, axum integration, "OpenAPI 3.1", license)
- [R3a] utoipa `#[utoipa::path]`, `operation_id` "By default this is mapped to function name":
  https://docs.rs/utoipa/6.0.0/utoipa/attr.path.html
- [R4] aide 0.15.1 docs: https://docs.rs/aide/0.15.1/aide/ and https://docs.rs/aide/0.15.1/aide/axum/index.html
- [R5] aide README, License: https://github.com/tamasfe/aide
- [R6] aide `crates/aide/Cargo.toml` on main (0.16.0-alpha.4, schemars 1.0.4, axum 0.8) and commit history:
  https://github.com/tamasfe/aide
- [R7] poem-openapi docs: https://docs.rs/poem-openapi/latest/poem_openapi/
- [R8] poem-openapi `src/registry/ser.rs`, `const OPENAPI_VERSION: &str = "3.0.0"`:
  https://github.com/poem-web/poem/blob/master/poem-openapi/src/registry/ser.rs
- [R8a] poem-openapi `src/payload/event_stream.rs` (`CONTENT_TYPE = "text/event-stream"`, `items` schema):
  https://github.com/poem-web/poem/blob/master/poem-openapi/src/payload/event_stream.rs
- [R9] dropshot docs: https://docs.rs/dropshot/latest/dropshot/
- [R9a] dropshot `src/api_description.rs`, `openapi.openapi = "3.0.3"`:
  https://github.com/oxidecomputer/dropshot/blob/main/dropshot/src/api_description.rs
- [R9b] dropshot `src/websocket.rs`, `WEBSOCKET_EXTENSION = "x-dropshot-websocket"`:
  https://github.com/oxidecomputer/dropshot/blob/main/dropshot/src/websocket.rs
- [R10] utoipa-axum 0.3.0 docs: https://docs.rs/utoipa-axum/latest/utoipa_axum/
- [R11] utoipa 6.0.0 `OpenApiVersion`: https://docs.rs/utoipa/6.0.0/utoipa/openapi/enum.OpenApiVersion.html
- [R12] utoipa-gen CHANGELOG (`item_schema` attribute):
  https://github.com/juhaku/utoipa/blob/master/utoipa-gen/CHANGELOG.md
- [R13] GitHub repository metadata (license, stars, last push) via `gh api repos/<owner>/<repo>`, 2026-11-11
- [R14] OpenAPI Generator, rust-axum: https://openapi-generator.tech/docs/generators/rust-axum/
- [R15] progenitor README: https://github.com/oxidecomputer/progenitor

TypeScript

- [T1] npm registry `https://registry.npmjs.org/<name>` (latest version, license, publish times), 2026-11-11
- [T2] openapi-typescript releases: https://github.com/openapi-ts/openapi-typescript/releases
- [T3] openapi-typescript README: https://github.com/openapi-ts/openapi-typescript/tree/main/packages/openapi-typescript
- [T3a] PR #2683 "feat: support OpenAPI 3.2 itemSchema (SSE/streaming)", open:
  https://github.com/openapi-ts/openapi-typescript/pull/2683
- [T4] openapi-fetch API docs (`parseAs`, middleware):
  https://github.com/openapi-ts/openapi-typescript/blob/main/docs/openapi-fetch/api.md
- [T5] Hey API get started and license: https://heyapi.dev/docs/openapi/typescript/get-started,
  https://github.com/hey-api/hey-api
- [T5a] Hey API TanStack Query plugin:
  https://github.com/hey-api/hey-api/blob/main/web/src/content/docs/docs/openapi/typescript/plugins/tanstack-query.mdx
- [T5b] Hey API parser dispatch `packages/shared/src/openApi/index.ts` (`>=3.0.0 <3.1.0`, `>=3.1.0`):
  https://github.com/hey-api/hey-api
- [T5c] Hey API migration notes, v0.81.0 "Server-Sent Events (SSE)":
  https://github.com/hey-api/hey-api/blob/main/web/src/content/docs/docs/openapi/typescript/migrating.mdx
- [T5d] Hey API client bundle `serverSentEvents.ts` (Last-Event-ID, retry, backoff, headers):
  https://github.com/hey-api/hey-api/blob/main/packages/openapi-ts/src/plugins/@hey-api/client-core/bundle/serverSentEvents.ts
- [T6] openapi-react-query README:
  https://github.com/openapi-ts/openapi-typescript/tree/main/packages/openapi-react-query
- [T7] orval README and releases: https://github.com/orval-labs/orval
- [T8] orval input configuration (2.0 to 3.1 upgrade):
  https://github.com/orval-labs/orval/blob/master/docs/content/docs/reference/configuration/input.mdx
- [T8a] orval NDJSON streaming guide and issue search for `event-stream`/SSE:
  https://github.com/orval-labs/orval/blob/master/docs/content/docs/guides/stream-ndjson.mdx
- [T9] kubb README and releases: https://github.com/kubb-labs/kubb
- [T9a] kubb PR #4541 "read OpenAPI 3.2 itemSchema and streaming fields", merged 2026-11-05:
  https://github.com/kubb-labs/kubb/pull/4541
- [T10] `@microsoft/fetch-event-source` on npm (2.0.1, 2021-04-25): https://www.npmjs.com/package/@microsoft/fetch-event-source

Push channel, web platform, Electron

- [P1] axum SSE module: https://docs.rs/axum/latest/axum/response/sse/index.html
- [P2] axum `Event`: https://docs.rs/axum/latest/axum/response/sse/struct.Event.html
- [P3] axum `KeepAlive`: https://docs.rs/axum/latest/axum/response/sse/struct.KeepAlive.html
- [P4] axum WebSocket module: https://docs.rs/axum/latest/axum/extract/ws/index.html
- [P5] WHATWG HTML, Server-sent events: https://html.spec.whatwg.org/multipage/server-sent-events.html
- [P6] MDN, `WebSocket()` constructor: https://developer.mozilla.org/en-US/docs/Web/API/WebSocket/WebSocket
- [P7] RFC 6455, The WebSocket Protocol, §4.1, §10.2, §10.5: https://www.rfc-editor.org/rfc/rfc6455.html
- [P8] nginx `ngx_http_proxy_module` (`proxy_buffering`, `X-Accel-Buffering`, `proxy_read_timeout`):
  https://nginx.org/en/docs/http/ngx_http_proxy_module.html
- [P9] nginx, WebSocket proxying: https://nginx.org/en/docs/http/websocket.html
- [P10] MDN, `EventSource`: https://developer.mozilla.org/en-US/docs/Web/API/EventSource
- [P11] TanStack Query, `streamedQuery`: https://tanstack.com/query/latest/docs/reference/streamedQuery
- [P12] Electron security checklist, item 18: https://www.electronjs.org/docs/latest/tutorial/security
- [P13] Electron `protocol.registerSchemesAsPrivileged`: https://www.electronjs.org/docs/latest/api/protocol
- [P14] MDN, `Set-Cookie` (`SameSite`, `Secure` on localhost):
  https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie
- [P15] RFC 9113, HTTP/2, §3.1: https://www.rfc-editor.org/rfc/rfc9113.html

Specifications

- [S1] OpenAPI Specification 3.2.0 (19 September 2025), Media Type Object `itemSchema`, "Sequential Media Types",
  "Special Considerations for Server-Sent Events", example "Server-Sent Event Streams":
  https://spec.openapis.org/oas/v3.2.0.html
- [S2] AsyncAPI specification releases (v3.1.0, 2026-01-31): https://github.com/asyncapi/spec/releases
- [S3] AsyncAPI WebSockets bindings 0.1.0: https://github.com/asyncapi/bindings/blob/master/websockets/README.md
- [S4] `asyncapi-rust` on crates.io: https://crates.io/crates/asyncapi-rust
- [S5] AsyncAPI Modelina: https://github.com/asyncapi/modelina
- [S6] kubb issue #3629 "adapter-asyncapi", open: https://github.com/kubb-labs/kubb/issues/3629

Licenses

- [L1] FSF, Various Licenses and Comments about Them: https://www.gnu.org/licenses/license-list.html
