# Rust crates for flock's mail protocols

Research for dyle71/flock#4 (part of the v1 specification map, #1).
Data collected on **2026-11-11**.

## Question

Which Rust crates can carry flock's mail protocols, and how mature,
maintained, async/tokio-compatible and GPLv3-compatible is each one? The
protocols in scope are:

- IMAP, including IDLE, CONDSTORE/QRESYNC, UIDPLUS, MOVE and COMPRESS
- SMTP submission
- POP3
- MIME parsing and building, including charsets and malformed real-world mail
- OAuth2/XOAUTH2 for Google and Microsoft, including the device and
  authorization-code flows
- a JMAP client for a later release

## Method and sources

- Versions, release dates, licenses, features and dependencies come from the
  crates.io API (`https://crates.io/api/v1/crates/<name>`, plus
  `/<version>/dependencies`).
- Repository activity ("last push") and archive status come from the GitHub
  REST API. Capability claims come from each crate's README, its docs.rs
  pages or its source code, linked inline.
- Protocol and provider facts come from RFCs, Google and Microsoft developer
  documentation, and the FSF license list.
- "Last release" means the newest non-yanked version on crates.io.
  "Downloads" is the all-time total on crates.io.

### License compatibility basis (FSF license list)

Source: <https://www.gnu.org/licenses/license-list.html>

- **Expat ("MIT")** is "compatible with the GNU GPL".
- **Apache-2.0** is "compatible with version 3 of the GNU GPL" (not GPLv2).
  Every `MIT OR Apache-2.0` crate is therefore usable from GPLv3 code under
  either option.
- **MPL-2.0**: section 3.3 gives indirect compatibility with the GPL. The
  MPL-covered files stay available under the MPL.
- **0BSD** "can operate as a free software license compatible with the GNU
  GPL".
- **EUPL-1.2** on its own is incompatible with the GPL, but it allows
  relicensing to GPLv3. melib is in any case dual-licensed
  `EUPL-1.2 OR GPL-3.0-or-later`, so its GPL option applies directly.
- **AGPLv3** is not strictly compatible with GPLv3, but each license's
  section 13 permits combining the two. The AGPL terms then still apply to
  the AGPL part.

**Result:** every crate surveyed below has a GPLv3-compatible license. None
of them is a license blocker.

### Async runtime note

`async-std` is officially discontinued. Its crates.io description reads
"Deprecated in favor of `smol`", and its README says "`async-std` has been
discontinued; use `smol` instead"
([repo](https://github.com/async-rs/async-std)). Crates that default to
`runtime-async-std` therefore need their tokio feature selected explicitly.

---

## IMAP

| Crate | Last release | License | Runtime | Downloads / repo activity | Link |
|---|---|---|---|---|---|
| `async-imap` | 0.12.0 (2026-09-30) | MIT OR Apache-2.0 | `runtime-tokio` or `runtime-async-std` (default: async-std) | 2.2M; repo `chatmail/async-imap` pushed 2026-11-11 | [crates.io](https://crates.io/crates/async-imap) · [repo](https://github.com/chatmail/async-imap) |
| `imap-proto` (parser under async-imap) | 0.17.0 (2026-09-28) | MIT OR Apache-2.0 | n/a (nom parser) | 4.8M; pushed 2026-11-10 | [repo](https://github.com/djc/imap-proto) |
| `imap-codec` / `imap-types` | stable 1.0.0 (2023-08-22); current 2.0.0-alpha.10 / 2.0.0-alpha.8 (2026-11-02) | MIT OR Apache-2.0 | sans-I/O (no runtime), `no_std` since 2026-09 | 249k; pushed 2026-11-09 | [repo](https://github.com/duesee/imap-codec) |
| `imap-next` | 0.3.4 (2026-02-18) | MIT OR Apache-2.0 | sans-I/O core; optional tokio `stream` feature (default) | 89k; pushed 2026-04-24 | [repo](https://github.com/duesee/imap-next) |
| `imap-client` (pimalaya) | 0.3.1 (2026-05-17) | MIT | tokio | 79k; **repo archived**, "Project moved: at https://github.com/pimalaya/io-imap" | [repo](https://github.com/pimalaya/imap-client) |
| `io-imap` (pimalaya) | 0.8.1 (2026-11-03); first release 2026-06-03 | MIT OR Apache-2.0 | sans-I/O coroutines; bundled "full client" is **blocking** | 71k; pushed 2026-11-03 | [repo](https://github.com/pimalaya/io-imap) |
| `imap` (rust-imap) | 2.4.1 stable; 3.0.0-alpha.15 (2025-02-08) | Apache-2.0 OR MIT | blocking only | 2.5M; pushed 2026-08-08 | [repo](https://github.com/jonhoo/rust-imap) |
| `melib` (IMAP backend) | see "Higher-level engines" | | | | |

### Capability coverage

**async-imap**
([Session docs](https://docs.rs/async-imap/0.12.0/async_imap/struct.Session.html),
[CHANGELOG](https://github.com/chatmail/async-imap/blob/main/CHANGELOG.md))

| Capability | Status |
|---|---|
| IDLE | Yes, via `extensions::idle`. Since 0.11.2, `Handle::wait()` times out after 29 minutes. |
| MOVE | Yes: `mv`/`uid_mv` (RFC 6851). |
| UIDPLUS | `uid_expunge` (RFC 4315) is provided. `append` returns `()` and does not surface APPENDUID. `imap-proto` does parse APPENDUID and COPYUID (`src/rfc4315.rs`). |
| CONDSTORE | `select_condstore`. |
| QRESYNC | No typed API. `imap-proto` parses `VANISHED (EARLIER)` (`src/rfc7162.rs`), so QRESYNC SELECT and FETCH parameters can only be sent through the raw `run_command*` methods. |
| COMPRESS=DEFLATE | Since 0.10.2, through the `compress` feature (`src/extensions/compress.rs`, using `async-compression`). |
| Other extensions | ID, QUOTA, GETMETADATA, and Gmail labels and message ID (X-GM-EXT-1, since 0.11.2). |
| Authentication | `Client::authenticate` takes a user-supplied `Authenticator`. XOAUTH2 means building the SASL string yourself. |
| TLS | Not bundled since 0.8.0: "TLS streams should be created by the library users". |

Maturity: maintained by the chatmail/Delta Chat project. The `imap-codec`
README says `imap-proto` "already succeeded in production as it is
(transitively) used in imap, async-imap, and Delta Chat."

**imap-codec / imap-types**
([imap-types docs](https://docs.rs/imap-types/2.0.0-alpha.8/imap_types/),
[README](https://github.com/duesee/imap-codec))

- **Scope:** parser and serializer only, with no networking. It implements
  the "complete formal syntax of IMAP4rev1" and serves both the client and
  the server side.
- **Supported without a feature flag:** LITERAL+/-, MOVE, UNSELECT, SASL-IR,
  COMPRESS (command syntax only; the DEFLATE stream is the caller's job),
  ENABLE, IDLE, QUOTA, UIDPLUS, BINARY, SORT/THREAD.
- **Behind features the docs mark "Unfinished":** `ext_condstore_qresync`,
  ID, LIST-EXTENDED, METADATA and the referrals extensions.
- **Malformed server output:** `quirk_*` features handle known server
  deviations, and a `Fragmentizer` supports "two-stage parsing" so a bad
  message can be dropped without losing the stream.
- **Testing:** fuzz-tested, with high coverage.
- **Gap:** the stable line is 1.0.0 from 2023. All current development and
  every dependent crate (imap-next, io-imap, melib) are on the
  **2.0.0-alpha** series.

**imap-next**

- A "thin sans I/O abstraction" over three IMAP flows: literal handling,
  AUTHENTICATE and IDLE.
- Its features gate `ext_condstore_qresync`, `ext_id`, `ext_metadata` and
  others through imap-codec.
- No high-level command API. It is a building block for a client, not a
  client.
- Depends on `imap-codec ^2.0.0-alpha.7`.

**io-imap** ([README](https://github.com/pimalaya/io-imap)), the successor
of `imap-client`

| Capability | Status |
|---|---|
| RFC coverage table | RFC 2177 (IDLE), 2971 (ID), 3501, 3691 (UNSELECT), 4315 (UIDPLUS, including APPENDUID and COPYUID), 4959 (SASL-IR), 5161 (ENABLE), 5256 (SORT/THREAD), 6851 (MOVE), **7162 (CONDSTORE and QRESYNC)**, 7628 (OAUTHBEARER), 7677 (SCRAM-SHA-256) |
| SASL mechanisms | PLAIN, LOGIN, XOAUTH2, OAUTHBEARER, SCRAM-SHA-256, ANONYMOUS |
| Message transfer | Streams bodies "never held in memory whole" |
| COMPRESS | **Not listed** |
| Async | The coroutines are "usable from any blocking or async runtime". The ready-made client is blocking, so a tokio driver loop would be flock's own code. |
| Maturity | Very young (first release June 2026, 7 stars) with a fast-moving API: 0.8.0 shipped a breaking `api!` change on 2026-11-06. Funded by NLnet (NGI Zero Commons Fund 2026–2027). |
| Built on | `imap-codec ^2.0.0-alpha.10` |

---

## SMTP submission

| Crate | Last release | License | Runtime | Downloads / activity | Link |
|---|---|---|---|---|---|
| `lettre` | 0.11.23 (2026-08-03) | MIT | sync, `tokio1` or `async-std1` features (default features are blocking + native-tls) | 23.8M; 2.3k stars; pushed 2026-11-10 | [repo](https://github.com/lettre/lettre) |
| `mail-send` (Stalwart) | 0.7.0 (2026-11-01) | Apache-2.0 OR MIT | tokio only ("Full async (requires Tokio)") + rustls | 639k; pushed 2026-11-01 | [repo](https://github.com/stalwartlabs/mail-send) |
| `async-smtp` (chatmail) | 0.10.2 (2025-05-13) | MIT OR Apache-2.0 | tokio (default) or async-std | 725k; pushed 2025-05-20 | [repo](https://github.com/chatmail/async-smtp) |
| `io-smtp` (pimalaya) | 0.7.1 (2026-11-03); first release 2026-06-03 | MIT OR Apache-2.0 | sans-I/O; bundled client blocking | 17k | [repo](https://github.com/pimalaya/io-smtp) |
| `smtp-proto` (Stalwart) | 0.2.5 (2026-10-01) | Apache-2.0 OR MIT | parser only | 680k | [repo](https://github.com/stalwartlabs/smtp-proto) |

### Capability coverage

**lettre**

- **SASL mechanisms** ([Mechanism](https://docs.rs/lettre/0.11.23/lettre/transport/smtp/authentication/enum.Mechanism.html)):
  `Plain`, `Login`, `Xoauth2`. There is **no OAUTHBEARER**.
- **Recognized ESMTP extensions** ([Extension](https://docs.rs/lettre/0.11.23/lettre/transport/smtp/extension/enum.Extension.html)):
  8BITMIME, SMTPUTF8, STARTTLS, AUTH. The enum is non-exhaustive.
- **Other:** a connection pool, a DKIM feature, and a message builder.
- **Gap:** the README says "Lettre does not provide (for now): Email parsing".

**mail-send** ([README](https://github.com/stalwartlabs/mail-send), `src/smtp/auth.rs`)

- **Authentication:** RFC 4954 AUTH with automatic mechanism negotiation:
  CRAM-MD5, DIGEST-MD5, XOAUTH2, LOGIN and PLAIN. The source also has a
  `Credentials::OAuthBearer` variant (`AUTH_OAUTHBEARER`).
- **Other:** STARTTLS and implicit TLS, DKIM signing through `mail-auth`,
  and it builds on `mail-builder` and `smtp-proto`.

**async-smtp**

- A fork of lettre's SMTP transport, without a message builder.
- Mechanisms: PLAIN, LOGIN, XOAUTH2.
- Last activity was May 2025.

**io-smtp**

- **RFCs:** 5321, 1870 (SIZE), 3207 (STARTTLS, with an injection check),
  3461 (DSN), 3463, 4954, and the PLAIN, LOGIN, ANONYMOUS, XOAUTH2,
  OAUTHBEARER and SCRAM-SHA-256 mechanisms.
- **Maturity:** same as io-imap: young, and sans-I/O.

---

## POP3

| Crate | Last release | License | Runtime | Downloads / activity | Link |
|---|---|---|---|---|---|
| `async-pop` (Dust-Mail) | 1.1.3 (2026-02-28); previous release 1.1.1 (2023-10-28) | MIT | `runtime-tokio` or `runtime-async-std` (default async-std + async-native-tls) | 30k; 6 stars; last code commit 2025-12-24, then dependency bumps | [repo](https://github.com/Dust-Mail/async-pop) |
| `async-pop2` | 1.1.1 (2025-01-08), single release | MIT | same codebase | 2.3k | [crates.io](https://crates.io/crates/async-pop2) |
| `rust-pop3-client` | 0.3.0 (2026-01-28) | MPL-2.0 | blocking, rustls | 3.8k | [repo](https://github.com/falk-werner/rust-pop3-client) |
| `pop3-codec` (duesee) | 0.3.0 (2022-06-12) | MIT OR Apache-2.0 | sans-I/O parser/serializer | 3.3k | [repo](https://github.com/duesee/pop3-codec) |
| `pop3` | 1.0.6 (2017-03-02) | MIT | blocking | unmaintained | [repo](https://github.com/mattnenterprise/rust-pop3) |

### async-pop coverage

Sources: [Client docs](https://docs.rs/async-pop/1.1.3/async_pop/struct.Client.html)
and `src/sasl.rs`.

| Capability | Status |
|---|---|
| RFC 1939 commands | USER/PASS, APOP, STAT, LIST, RETR, DELE, NOOP, RSET, QUIT, TOP, UIDL |
| CAPA (RFC 2449) | Yes |
| AUTH (SASL) | Pluggable `Authenticator`. "The mechanisms for PLAIN and XOAUTH2 are already present". LOGIN was added in December 2025. |
| STLS (STARTTLS) | No method for it on `Client` |
| Maintenance | Low: one maintainer project, 8 open issues |

No POP3 client crate is both actively developed and widely used.
POP3 itself is small: RFC 1939, plus CAPA (RFC 2449), STLS (RFC 2595) and
SASL AUTH (RFC 5034).

---

## MIME parsing and building

| Crate | Last release | License | Role | Downloads / activity | Link |
|---|---|---|---|---|---|
| `mail-parser` (Stalwart) | 1.0.1 (2026-11-02); 1.0.0 on 2026-10-24 | Apache-2.0 OR MIT | parse | 5.5M; 479 stars; pushed 2026-11-02 | [repo](https://github.com/stalwartlabs/mail-parser) |
| `mail-builder` (Stalwart) | 2.0.1 (2026-11-02) | Apache-2.0 OR MIT | build | 2.1M | [repo](https://github.com/stalwartlabs/mail-builder) |
| `mailparse` | 0.18.0 (2026-10-05) | 0BSD | parse | 16.2M; pushed 2026-10-05 | [repo](https://github.com/staktrace/mailparse) |
| `lettre` `message` module | 0.11.23 | MIT | build only | see SMTP | |
| `melib` `email` module | 0.9.0 | EUPL-1.2 OR GPL-3.0-or-later | parse and build | see below | |
| `eml-codec` (Deuxfleurs) | 0.4.1 (2026-08-18) | GPL-3.0-or-later | parse | 4.6k | [repo](https://git.deuxfleurs.fr/Deuxfleurs/eml-codec) |
| `encoding_rs` / `charset` | 0.8.44 / 0.1.5 | (Apache-2.0 OR MIT) AND BSD-3-Clause / Apache-2.0 OR MIT | charset decoding | 633M / 27.8M | [encoding_rs](https://github.com/hsivonen/encoding_rs) |

### mail-parser coverage

Source: [README](https://github.com/stalwartlabs/mail-parser).

| Area | Detail |
|---|---|
| Standards | "Fully conforms" to RFC 5322 and RFC 2045–2049. Also RFC 2047, 2231 (parameter continuations), 2557, 2183, 2392, 6532 (internationalized headers), 2152 (UTF-7), 2369, 2919, 6854, and others. |
| Body lists | RFC 8621 §4.1.4 text, HTML and attachment lists, with HTML-to-text and text-to-HTML fallback. This is the same shape JMAP uses. |
| Charsets | "41 different character sets including obsolete formats such as UTF-7", plus every WHATWG Encoding Standard label. The multi-byte CJK sets (Shift_JIS, ISO-2022-JP, Big5, GB18030, EUC-KR, …) need the `full_encoding` feature (through `encoding_rs`); without it they decode as UTF-8 and are reported as an unknown charset. |
| Malformed mail | Follows Postel's law: "best effort to parse non-conformant e-mail messages". "Battle-tested with millions of real-world e-mail messages dating from 1995 until today". |
| Testing | Every function is fuzzed (parse, limits, fields, charsets, kernels and mbox targets) and Miri-tested. |
| Design | Zero-copy, lazy decoding, SIMD scanning. Includes an mbox reader. |
| Production use | Stalwart Mail Server |
| Gap | **MSRV is Rust 1.98** for 1.0.x. A parser only, so building needs `mail-builder`. |

### Other MIME crates

**mail-builder**

- RFC 5322 and MIME (RFC 2045–2049) generation that picks the best encoding
  per part, with RFC 2047 header encoding.
- Its only required dependency is `encodify`. It runs on wasm32.

**mailparse**

- Minimal dependencies: `charset`, `data-encoding` and `quoted_printable`.
- Older and smaller API. Parse only.

---

## OAuth2 / XOAUTH2

### Provider facts (primary sources)

**Google** ([XOAUTH2 protocol](https://developers.google.com/workspace/gmail/imap/xoauth2-protocol),
[device flow](https://developers.google.com/identity/protocols/oauth2/limited-input-device))

- **Scope:** "The scope for IMAP, POP, and SMTP access is
  `https://mail.google.com/`". Apps using this scope must complete
  verification.
- **SASL format:**
  `base64("user=" {User} "^Aauth=Bearer " {Access Token} "^A^A")`.
- **OAUTHBEARER:** the page documents XOAUTH2 only.
- **Device flow:** "The OAuth 2.0 flow for devices is supported only for the
  following scopes". The list holds OpenID (`email`, `openid`, `profile`),
  two Drive scopes and two YouTube scopes. **No Gmail scope is included**, so
  Gmail IMAP, SMTP and POP access cannot use Google's device flow; it needs
  the authorization-code flow (with PKCE).

**Microsoft** ([Authenticate IMAP/POP/SMTP using OAuth](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth),
updated 2025-10-17)

- **Accounts covered:** "available for both Microsoft 365 … and Outlook.com
  users".
- **Flows:** authorization code and device authorization grant, plus client
  credentials for app-only access.
- **Scopes:** `https://outlook.office.com/IMAP.AccessAsUser.All`,
  `…/POP.AccessAsUser.All` and `…/SMTP.Send`. Add `offline_access` to get
  refresh tokens.
- **SASL:** XOAUTH2, in the same format as Google's. For shared mailboxes,
  the username field carries the shared mailbox address.

### Crates

| Crate | Last release | License | Runtime | Flows | Link |
|---|---|---|---|---|---|
| `oauth2` (ramosbugs) | 5.0.0 (2025-01-21); repo pushed 2026-02-22 | MIT OR Apache-2.0 | async through the `AsyncHttpClient` trait (any closure returning a future); `reqwest` default; no runtime lock-in | Auth code + PKCE, device authorization, client credentials, refresh, revocation (RFC 7009), introspection (RFC 7662) | [docs](https://docs.rs/oauth2/5.0.0/oauth2/) · [repo](https://github.com/ramosbugs/oauth2-rs) |
| `openidconnect` (same author) | 4.1.0 (2026-10-31) | MIT | same model | OIDC on top of `oauth2` | [repo](https://github.com/ramosbugs/openidconnect-rs) |
| `yup-oauth2` | 12.1.2 (2026-01-07) | MIT OR Apache-2.0 | tokio/hyper | Google-oriented: installed-app, device and service-account flows | [repo](https://github.com/dermesser/yup-oauth2) |
| `io-oauth` (pimalaya) | 0.5.2 (2026-11-11) | MIT OR Apache-2.0 | sans-I/O; blocking full client | RFC 6749, 7523, 7591 (dynamic registration), 7636 (PKCE), 8628 (device) | [repo](https://github.com/pimalaya/io-oauth) |
| `oauth-lib` (pimalaya, older) | 2.0.0 (2024-12-09) | MIT | async | superseded by `io-oauth` | |

`oauth2` is the widely used choice, with 63.9M downloads and 1.2k stars.
XOAUTH2 itself is only a string format. The SASL step is done by the
IMAP/SMTP/POP crate, which supports it as follows:

| Crate | SASL support |
|---|---|
| async-imap | custom `Authenticator` |
| lettre, mail-send, async-smtp | `Xoauth2` |
| mail-send, io-imap, io-smtp | also OAUTHBEARER |
| async-pop | `OAuth2Authenticator` |

---

## JMAP client (later release)

| Crate | Last release | License | Runtime | Coverage | Link |
|---|---|---|---|---|---|
| `jmap-client` (Stalwart) | 0.4.3 (2026-09-28) | Apache-2.0 OR MIT | async (tokio, reqwest); optional `blocking`; `websockets` feature | RFC 8620 (core), 8621 (mail), 8887 (WebSocket), JMAP Sieve draft; EventSource push | [repo](https://github.com/stalwartlabs/jmap-client) |
| `io-jmap` (pimalaya) | 0.7.0 (2026-11-03) | MIT OR Apache-2.0 | sans-I/O | used by `io-email` | [crates.io](https://crates.io/crates/io-jmap) |
| `jmap-base-client` and related crates | 0.5.0 (2026-11-03); first release 2026-05 | MIT OR Apache-2.0 | async | RFC 8620 base, SSE, WebSocket; separate types crates for mail, contacts and calendars | [repo](https://github.com/MarkAtwood/crate-jmap) |
| `melib` JMAP backend | 0.9.0 | EUPL-1.2 OR GPL-3.0-or-later | smol/async-io | full backend with EventSource | see below |

`jmap-client` details:

- 108k downloads and 126 stars. It ships releases at a slower pace
  (0.3.2 in December 2023, then 0.4.x through 2026).
- It is written by the authors of Stalwart, which is itself a JMAP server.

---

## The Stalwart crates

Stalwart publishes these permissively licensed (Apache-2.0 OR MIT) client and
format crates on crates.io:

- `mail-parser`
- `mail-builder`
- `mail-send`
- `smtp-proto`
- `mail-auth` (DKIM/SPF/DMARC/ARC; 0.15.1, 2026-10-30)
- `jmap-client`

The Stalwart **server** is a different case:

- It is AGPL-3.0 dual-licensed with the proprietary SELv2
  ([README](https://github.com/stalwartlabs/stalwart)).
- Its protocol crates in `crates/` (`imap-proto`, `jmap-proto`, `pop3`,
  `smtp`, and others) are server-side workspace crates, not published client
  libraries. Note that this `imap-proto` is a different crate from djc's
  `imap-proto`.
- The published crates are actively maintained. Most released within the
  last two weeks, and `mail-parser` and `mail-builder` reached 1.0 and 2.0 in
  October 2026.

---

## Higher-level engines (does a sync engine crate exist?)

| Crate | Last release | License | Runtime | What it is | Status / gaps |
|---|---|---|---|---|---|
| `melib` (meli) | 0.9.0 (2026-11-01) | EUPL-1.2 OR GPL-3.0-or-later | **smol/async-io** (not tokio) | Full mail-client backend: IMAP, JMAP, Maildir, mbox, notmuch, NNTP, SMTP, plus an SQLite cache. Its IMAP side advertises IDLE, CONDSTORE, COMPRESS=DEFLATE, MOVE, UIDPLUS, ENABLE, LIST-STATUS, SPECIAL-USE, LITERAL+, SASL-IR, XOAUTH2 and OAUTHBEARER (`melib/src/imap/mod.rs`). | `resync_condstoreqresync` logs "function unimplemented" and falls back to CONDSTORE (`melib/src/imap/sync/mod.rs`). No POP3. 35k downloads; 898 stars; pinned to `imap-codec =2.0.0-alpha.6`. |
| `io-email` (pimalaya) | 0.1.0 (2026-06-05) | MIT OR Apache-2.0 | sans-I/O; blocking client | One API over IMAP, JMAP, Gmail API, Microsoft Graph, Maildir, m2dir and SMTP, with `diff_mailboxes`/`diff_envelopes` and `watch_mailbox`. | A single release, 349 downloads. A primitives layer rather than a full store-plus-sync engine. |
| `email-lib` (pimalaya/core) | 0.27.0 (2026-02-19) | MIT | tokio | Older pimalaya library with a `sync` feature (IMAP to Maildir). | Depends on the archived `imap-client`, `mail-parser ^0.9` and `mail-send ^0.4`. Superseded by the io-* crates. |
| `mail-sync-kit` | 0.1.2 (2026-10-24); first release 2026-09-11 | MIT OR Apache-2.0 | tokio | Store-agnostic engines: IMAP (CONDSTORE deltas, UIDVALIDITY handling, IDLE with polling fallback) behind a `MailStore` trait, JMAP delta sync, and an SMTP outbox with an offline gate. | 418 downloads, 0 stars, one author. Built on `imap-next` and `lettre`. QRESYNC is named in the crate description but not in the README's engine list. |
| `vomit-sync` | 0.10.1 (2024-05-24) | GPL-3.0-or-later | blocking (`imap` 3.0 alpha) | IMAP to Maildir sync | No release since May 2024 |

No mature, widely adopted, tokio-native sync engine crate was found that
does both of these:

- syncs into an application-defined store, and
- implements QRESYNC.

---

## Implications for flock

These are facts that bear on later decisions, not decisions.

**Licenses**

1. Every candidate has a GPLv3-compatible license. MIT, Apache-2.0, 0BSD and
   MPL-2.0 can all be combined, and melib, eml-codec and vomit-sync are
   themselves GPL-3.0-or-later options. No crate is excluded on license
   grounds.

**IMAP**

2. No published async IMAP client crate offers a typed API for all of IDLE,
   CONDSTORE, QRESYNC, UIDPLUS, MOVE and COMPRESS together:
   - `async-imap`, the tokio-capable crate with production use (Delta Chat),
     covers IDLE, MOVE, CONDSTORE (select), UID EXPUNGE and COMPRESS. It has
     no QRESYNC API and does not surface APPENDUID. The parser underneath
     does handle VANISHED, APPENDUID and COPYUID.
   - `io-imap` covers CONDSTORE, QRESYNC and UIDPLUS, but not COMPRESS. It
     is five months old and changes its API quickly.
   - The `imap-codec` 2.x / `imap-next` stack gives full IMAP syntax
     (including COMPRESS syntax and gated CONDSTORE/QRESYNC types), but no
     client command layer. Its stable 1.0.0 is from 2023, and current work
     is on 2.0 alphas.
3. QRESYNC-based resync logic is not available off the shelf in any
   surveyed crate. melib leaves it unimplemented, and io-imap provides the
   protocol coroutines but not a store-level sync algorithm.

**Runtimes**

4. tokio compatibility:
   - Native tokio: `mail-send`, `jmap-client`, `email-lib`, `mail-sync-kit`.
   - Opt-in tokio feature: `async-imap`, `lettre`, `async-pop`.
   - Runtime-agnostic: `oauth2`, `imap-next` (sans-I/O, with a tokio
     stream), and the pimalaya `io-*` crates (sans-I/O).
   - melib uses smol/async-io.
   - async-std, the default runtime of `async-imap` and `async-pop`, is
     discontinued.

**SMTP**

5. Two maintained tokio options with XOAUTH2 cover SMTP submission:
   - `lettre`: largest user base; no OAUTHBEARER.
   - `mail-send`: has OAUTHBEARER and DKIM; pairs with `mail-builder`.

**POP3**

6. POP3 is the weakest area:
   - `async-pop` is the only async client. It has CAPA, UIDL, TOP and SASL
     (PLAIN, XOAUTH2, LOGIN), but no STLS and little activity.
   - The protocol is small (RFC 1939, 2449, 2595, 5034).

**MIME**

7. `mail-parser` 1.0 is the most complete parser surveyed. It covers
   charsets including CJK (`full_encoding`) and UTF-7, is fuzzed and
   Miri-tested, has a large real-world corpus, and produces the RFC 8621
   body lists. It requires Rust ≥ 1.98. `mail-builder` is its building
   counterpart.

**OAuth**

8. The flows that must be supported are set by the providers, not by the
   crates:
   - Gmail IMAP, SMTP and POP need the `https://mail.google.com/` scope,
     which Google's device flow does not allow. Google therefore needs the
     authorization-code flow, and the scope requires app verification.
   - Microsoft (M365 and Outlook.com) allows both authorization-code and
     device-code flows.
   - Both providers use SASL XOAUTH2.
   - The `oauth2` crate supports PKCE, the device flow and refresh, with a
     pluggable async HTTP client.

**JMAP**

9. For the later release, `jmap-client` (Stalwart) covers RFC 8620, 8621
   and 8887 with push, and it is async/tokio.

**Ecosystem**

10. The pimalaya project (NLnet-funded) is moving its whole stack (IMAP,
    SMTP, JMAP, OAuth, Gmail API, Microsoft Graph) to sans-I/O `io-*`
    crates. These are new (since mid-2026) and churning, but they are the
    only family covering every protocol in the question with one design.
