# Rust building blocks for server-mode sign-in

Research for [dyle71/flock#17](https://github.com/dyle71/flock/issues/17), part of the wayfinder
map [dyle71/flock#1](https://github.com/dyle71/flock/issues/1). Builds on the settled runtime modes
of [dyle71/flock#9](https://github.com/dyle71/flock/issues/9).

Researched 2026-11-11. Versions, release dates, licenses and dependency ranges come from the
crates.io API; repository activity ("last push", archived, open issues) from the GitHub REST API;
behavior from each crate's README, docs or source at the cited path. "Last release" is the newest
non-yanked version. Statements marked _(inference)_ are my own reasoning from the cited facts. No
source states them directly.

## Question

Which Rust crates and patterns carry flock's Server mode sign-in on axum: password hashing, cookie
sessions and their storage, CSRF protection for a cookie-authenticated JSON + SSE API, TOTP with
recovery codes, invite links, rate limiting of sign-in attempts, trusting `X-Forwarded-Proto`
behind a reverse proxy, and later passkeys and OIDC? For each: version and last release, license,
maintenance, axum/tower compatibility, and what it assumes about the store. Which OWASP guidance
applies?

Settled context: one async Rust binary, axum the leading candidate; password (argon2) + TOTP in
v1, passkeys and OIDC later; closed sign-up, the first User is the Admin and invites by link; plain
HTTP behind a reverse proxy, sign-in refused unless the request came over HTTPS
(`X-Forwarded-Proto`) or `--insecure` is set; SPA and API on one origin; SSE push; the store
(SQLite or PostgreSQL) is open.

## Summary

- **Baseline.** axum 0.8.9 (2026-04-14) is current; its `main` branch is still 0.8.9, so no 0.9
  is in sight [G-axum]. Every crate below targets `http` 1, `tower` 0.5 and axum 0.8 (or is
  framework-free). Every license is MIT, Apache-2.0, BSD-3-Clause, MPL-2.0 or LGPL-3.0, all
  GPLv3-compatible per the FSF [W-fsf].
- **Passwords.** RustCrypto `argon2` 0.6.0 defaults to Argon2id with m=19 MiB, t=2, p=1, exactly
  OWASP's minimum [G-argon2-params][O-pw]. `password-auth` 2.0.0 wraps it in two functions.
- **Sessions.** `tower-sessions` 0.15.0 is the de-facto session layer. Its defaults already match
  OWASP: `HttpOnly`, `Secure`, `SameSite=Strict`, cookie name `id`, 128-bit random IDs, and a
  `cycle_id()` for fixation [G-ts-service][G-ts-session]. `tower-sessions-sqlx-store` covers SQLite,
  PostgreSQL and MySQL, but the release on crates.io still pins sqlx 0.8 while sqlx is at 0.9.
  Stores keep the raw session ID as the primary key.
- **axum-login** 0.18.0 adds `AuthSession`, `login()` (cycles the ID) and route guards on top. Its
  release depends on `tower-sessions` 0.14, not 0.15; `main` has moved to 0.15 but is unreleased.
  It knows nothing of a second factor.
- **CSRF.** `tower-http` 0.7.0 (2026-06-15) added `CsrfLayer`, a port of Go 1.25's
  `CrossOriginProtection` built on `Sec-Fetch-Site` and `Origin` [G-th-csrf]. That matches the rule
  klicker.ing already runs [K1] and OWASP's Fetch Metadata guidance [O-csrf]. Token crates exist
  (`axum_csrf`) but are built for server-rendered forms.
- **TOTP.** `totp-rs` 6.0.0 is maintained and RFC 6238 compliant, but leaves replay prevention to
  the caller [G-totp]. No crate handles recovery codes; they are random strings hashed in the store.
- **Invite links.** No crate; the pattern is a CSPRNG token, stored hashed, single-use, expiring
  [O-forgot].
- **Rate limiting.** `tower_governor` 0.8.0 is in-memory GCRA per IP. Its last commit is
  2025-08-14, and its `SmartIpKeyExtractor` takes the **leftmost** `X-Forwarded-For` entry, which a
  client controls. OWASP wants the failed-login counter on the account, not the IP [O-auth].
- **Proxy headers.** No Rust crate validates `X-Forwarded-Proto` against a trusted proxy.
  `axum-extra`'s `Scheme` extractor trusts it blindly and is deprecated [G-axum-scheme].
  `axum-client-ip` 1.x does client IPs right (rightmost `X-Forwarded-For`, configurable source).
- **Later.** `webauthn-rs` 0.6.2 (MPL-2.0) is the relying-party library; it insists its ceremony
  state stays server-side [G-webauthn]. `openidconnect` 4.1.0 is the OIDC RP library; it needs
  PKCE verifier, nonce and state kept in the session between redirect and callback.

## 1. Versions, licenses and fit at a glance

| Crate | Last release | License | Repo last push | axum / tower | Store |
|---|---|---|---|---|---|
| `argon2` | 0.6.0 (2026-08-27) | MIT OR Apache-2.0 | 2026-11-09 | none | PHC string |
| `password-hash` | 0.6.1 (2026-04-08) | MIT OR Apache-2.0 | 2026-11-09 | none | PHC string |
| `password-auth` | 2.0.0 (2026-11-05) | MIT OR Apache-2.0 | 2026-11-09 | none | PHC string |
| `tower-sessions` | 0.15.0 (2026-02-01) | MIT | 2026-10-26 | axum-core 0.5 | `SessionStore` |
| `tower-sessions-sqlx-store` | 0.15.1 (2026-11-08) | MIT | 2026-11-11 | via core 0.15 | sqlx ^0.8 |
| `tower-sessions-rusqlite-store` | 0.15.0 (2026-10-04) | MIT | n/a | core 0.15 | rusqlite |
| `tower-sessions-redis-store` | 0.16.0 (2025-01-01) | MIT | 2026-11-11 | core **0.14** | Redis |
| `tower-sessions-moka-store` | 0.15.0 (2025-01-01) | MIT | 2026-11-11 | core **0.14** | memory |
| `axum-login` | 0.18.0 (2025-07-20) | MIT | 2026-11-11 | axum 0.8, ts **0.14** | via t-s |
| `axum-extra` (cookies) | 0.12.6 (2026-04-14) | MIT | 2026-11-10 | axum 0.8 | none |
| `tower-cookies` | 0.11.0 (2025-01-01) | MIT | 2025-09-10 | axum-core 0.5 | none |
| `cookie` | 0.18.2 (2026-08-08) | MIT OR Apache-2.0 | 2026-08-08 | none | none |
| `tower-http` (`csrf`) | 0.7.1 (2026-08-31) | MIT | n/a | tower 0.5 | none |
| `axum_csrf` | 0.12.0 (2026-10-22) | MIT | 2026-10-22 | axum-core 0.5 | cookie |
| `axum-csrf-sync-pattern` | 0.3.2 (2024-04-05) | MPL-2.0 | 2026-11-08 | **axum 0.6** | session |
| `totp-rs` | 6.0.0 (2026-08-06) | MIT | 2026-10-30 | none | caller's |
| `totp-lite` | 2.0.1 (2023-10-30) | MIT | 2026-06-23 | none | caller's |
| `governor` | 0.10.4 (2025-12-16) | MIT | 2026-08-24 | none | in memory |
| `tower_governor` | 0.8.0 (2025-08-14) | MIT OR Apache-2.0 | 2025-08-14 | axum 0.8 | in memory |
| `axum-client-ip` | 1.3.1 (2026-01-22) | MIT | 2026-07-17 | axum 0.8 | none |
| `webauthn-rs` | 0.6.2 (2026-10-31) | MPL-2.0 | 2026-11-05 | none | caller's |
| `openidconnect` | 4.1.0 (2026-10-31) | MIT | 2026-10-31 | none | caller's |
| `oauth2` | 5.0.0 (2025-01-21) | MIT OR Apache-2.0 | 2026-02-22 | none | caller's |
| `axum-oidc` | 0.6.0 (2025-01-12) | LGPL-3.0-or-later | n/a | axum 0.8 | session |

Sources: [C-api] for versions, licenses and dependency ranges; [G-repos] for pushes. "n/a" means
not checked (Codeberg or not on GitHub's API in this pass). `tower-http`'s repository was not
queried; its release dates come from crates.io.

**Licenses.** Per the FSF list [W-fsf]: Expat (MIT) and Modified BSD (`subtle`, BSD-3-Clause) are
GPL-compatible; Apache-2.0 is compatible with GPLv3; MPL-2.0 is compatible through its section 3.3
(the MPL files stay MPL); LGPL-3.0 is compatible with GPLv3. No crate here blocks a GPL-3.0 binary.

**Retired options.** `axum-sessions` 0.6.1 (2023-10-13) says "Development of this crate has moved
to `tower-sessions` … Please consider migrating" [G-axum-sessions]. `async-session` 3.0.0 dates
from 2021-06-01 [C-api]. `tower-csrf` 0.1.0 is archived; its README says it was merged into
`tower-http` and the crate will be depublished around 2027-01-01 [G-tower-csrf].

## 2. Password hashing

- **`argon2` 0.6.0** (RustCrypto `password-hashes`). The 0.6 line went through rc.4 (2025-12-10)
  to rc.8 (2026-03-22) before 0.6.0 on 2026-08-27 [C-api]. `Params::DEFAULT` is
  `m_cost = 19 * 1024` (KiB), `t_cost = 2`, `p_cost = 1` [G-argon2-params]. OWASP: "Use Argon2id
  with a minimum configuration of 19 MiB of memory, an iteration count of 2, and 1 degree of
  parallelism" [O-pw]. Output is a PHC string (`$argon2id$v=19$m=…`) through `password-hash` 0.6,
  so salt and parameters live in the one text column. MSRV 1.85.
- **`password-auth` 2.0.0** (same repo) is the high-level wrapper: `generate_hash` and
  `verify_password`. Its README: "By default, it supports Argon2 (using the latest OWASP
  recommended parameters)", with optional `pbkdf2` and `scrypt` features to verify legacy hashes
  [G-password-auth]. Its SemVer note warns that `getrandom` may be bumped without a major version.
- **Blocking work.** Argon2 at 19 MiB is CPU- and memory-heavy. axum-login's SQLite example runs
  `verify_password` inside `tokio::task::spawn_blocking` with the comment "Verifying the password
  is blocking and potentially slow" [G-al-example].
- **Store.** One `TEXT` column for the PHC string. Works the same on SQLite and PostgreSQL.

OWASP also asks for a maximum input length to avoid long-password DoS, a minimum of 8 characters
when MFA is on (15 without), at least 64 allowed, no composition rules, and blocklist screening
[O-auth]. None of the crates enforce this; it is handler code.

## 3. Sessions and cookies

### tower-sessions 0.15.0

- **Model.** A `tower` layer plus an axum `Session` extractor; key-value data serialized as JSON;
  storage behind the `SessionStore` trait; sessions load lazily only in handlers that touch them
  [G-ts-readme]. Modeled on Django's sessions middleware.
- **Cookie defaults** in `SessionConfig::default()`: name `"id"` (with a comment citing OWASP's
  session-ID-name-fingerprinting advice), `http_only: true`, `same_site: SameSite::Strict`,
  `secure: true`, `always_save: false` [G-ts-service]. Builders: `with_name`, `with_secure`,
  `with_same_site`, `with_http_only`, `with_path`, `with_domain`, `with_expiry`, and, behind the
  `signed` / `private` features, `with_signed(key)` / `with_private(key)`.
- **IDs.** `Id(pub i128)` from `rand::rng().random()`: 128 random bits from rand's thread-local
  CSPRNG [G-ts-session]. OWASP's floor is 64 bits [O-session].
- **Fixation.** `Session::cycle_id()`: "Cycles the session ID while retaining any data … helps
  prevent session fixation attacks". It deletes the old record from the store and creates a new
  one on save [G-ts-session].
- **Expiry.** `Expiry::OnSessionEnd`, `OnInactivity(Duration)`, `AtDateTime(OffsetDateTime)`.
  "Reading a session is not considered activity … expiration is computed from the last time the
  session was _modified_" unless `with_always_save(true)` [G-ts-session]. There is one expiry per
  session: idle **and** absolute timeouts together (OWASP asks for both [O-session]) need the
  absolute deadline stored in the session data and checked by the app _(inference)_.
- **Compatibility.** Depends on `tower-cookies` ^0.11, `tower-layer` 0.3, `http` 1 and optional
  `axum-core` ^0.5, i.e. axum 0.8 [C-deps].
- **Maintenance.** 0.13.0 (2024-09-03), 0.14.0 (2025-01-01), 0.15.0 (2026-02-01): roughly one
  breaking release a year, 5 open issues, last push 2026-10-26. The README still shows
  `tower-sessions = "0.14.0"` in its install snippet [G-ts-readme].

### Stores

- **`tower-sessions-sqlx-store` 0.15.1** (2026-11-08), features `sqlite`, `postgres`, `mysql`.
  Data is MessagePack; `migrate()` creates the table [G-sqlx-store-readme].
  - SQLite: `id text primary key not null, data blob not null, expiry_date integer not null`.
  - PostgreSQL: schema `tower_sessions`, `id text … , data bytea …, expiry_date timestamptz …`.
  - `ExpiredDeletion::continuously_delete_expired(period)` is a background task that deletes
    expired rows [G-ts-store-trait].
  - The published 0.15.1 depends on `sqlx ^0.8.0`; the repo's `sqlx-store/Cargo.toml` on `main`
    already says `sqlx = "0.9.0"` but carries version 0.15.0 and is unreleased
    [C-deps][G-sqlx-store].
    sqlx 0.9.0 was released 2026-05-21. A flock on sqlx 0.9 would pull sqlx 0.8 in as well until
    a new store release ships _(inference)_.
  - The raw session ID is the primary key and the lookup is `where id = ?` [G-sqlx-store].
- **`tower-sessions-rusqlite-store` 0.15.0** (2026-10-04): third-party, on `tokio-rusqlite` 0.8
  and `rusqlite` ^0.40, core ^0.15 [C-deps]. 14k downloads in total.
- **`tower-sessions-redis-store` 0.16.0, `-moka-store` 0.15.0**: last released 2025-01-01 and still
  on core ^0.14 [C-deps]; the stores repo says "we will not be adding additional stores here"
  [G-ts-stores].
- **`tower-sessions-cookie-store` 0.5.2**: all data in a signed or private cookie, listed as not
  persistent in the tower-sessions README [G-ts-readme]. Server-side revocation is not possible
  with it _(inference)_.
- **No user index.** None of the stores can list or delete "all sessions of User X"; the table has
  only `id`, `data` and `expiry_date` [G-sqlx-store]. Sign-out-everywhere, Admin suspension and
  "list my sessions" need either axum-login's auth-hash check (section 4) or a custom
  `SessionStore` with a `user_id` column _(inference)_.

### Plain cookies

`axum-extra` 0.12.6 has `CookieJar`, `SignedCookieJar` and `PrivateCookieJar` behind the `cookie`,
`cookie-signed` and `cookie-private` features, built on `cookie` 0.18 [G-axum-extra-cookie]. They
suit a non-session cookie (for example, a short-lived "TOTP pending" marker); `tower-sessions`
uses `tower-cookies` 0.11 for its own cookie [C-deps].

## 4. axum-login 0.18.0

- **What it is.** "User identification, authentication, and authorization as a `tower` middleware
  for `axum`" on top of `tower-sessions`. The app implements `AuthUser` (`id()`,
  `session_auth_hash()`) and `AuthnBackend` (`authenticate(credentials)`, `get_user(id)`);
  `AuthzBackend` adds permissions; routes are guarded with `login_required!` /
  `permission_required!` or the `require` builder [G-al-readme].
- **Fixation.** `AuthSession::login` calls `session.cycle_id()` with the comment "Session-fixation
  mitigation"; `logout` calls `session.flush()` [G-al-session].
- **Invalidation.** On every request it loads the user with `get_user` and compares the stored
  auth hash with `session_auth_hash()` using `subtle::ConstantTimeEq`; on mismatch it flushes the
  session [G-al-session]. Its SQLite example returns the password hash, "what this means is when
  the user changes their password the auth session becomes invalid" [G-al-example]. That copies the
  password hash into every session row; any per-user secret (a random "session generation") works
  the same way _(inference)_. A suspended User can be logged out by `get_user` returning `None`
  _(inference from the same code path)_.
- **Second factor.** No concept of a half-authenticated session. A password-then-TOTP flow keeps
  a "pending User" key in the plain session and calls `login()` only after the TOTP check
  _(inference)_. The repo's examples are `sqlite`, `permissions`, `oauth2` and `multi-auth`
  [G-al-examples].
- **Version skew.** The published 0.18.0 (2025-07-20) depends on `tower-sessions ^0.14.0`, while
  tower-sessions is at 0.15.0 [C-deps]. `main` already depends on `tower-sessions` 0.15.0 and
  `tower-sessions-sqlx-store` 0.15.0 but still says version 0.18.0 [G-al-cargo]. Using axum-login
  today means tower-sessions 0.14 and `tower-sessions-sqlx-store` 0.14 (or a git dependency)
  _(inference)_. Release cadence: 0.15.x (2024-06), 0.16.0 (2024-09), 0.17.0 (2025-01), 0.18.0
  (2025-07). 1,025 stars, 7 open issues, last push 2026-11-11.
- **Store.** Whatever `SessionStore` sits underneath; users come from the app's own backend.

## 5. CSRF for a cookie-authenticated JSON + SSE API

### tower-http `CsrfLayer` (0.7.0+)

- Added in tower-http 0.7.0 (2026-06-15) behind the `csrf` feature: "porting the cross-origin
  protection scheme introduced in Go 1.25" (PR #699, merged 2026-05-30) [G-th-changelog].
- Rules, quoted from the module docs [G-th-csrf]. A request is allowed if any of these hold:
  1. "The method is `GET`, `HEAD`, or `OPTIONS`."
  2. "The `Origin` header byte-for-byte matches an allow-listed trusted origin."
  3. "`Sec-Fetch-Site` is `same-origin` or `none`."
  4. "Neither `Sec-Fetch-Site` nor `Origin` is present."
  5. "The `Origin`'s authority (host and any port) matches the request's effective host
     byte-for-byte."
- Rejections are `403`, with a `ProtectionError` in the response extensions;
  `with_rejection_response` customizes the body; `add_trusted_origin` and `with_insecure_bypass`
  configure it [G-th-csrf-layer].
- Its own deployment caveat: "Reverse proxies and load balancers that rewrite `Host` … or strip
  `Origin` silently degrade the protection" [G-th-csrf]. nginx does exactly that by default:
  `proxy_set_header Host $proxy_host` [W-nginx]. Registering the public origin with
  `add_trusted_origin`, or forwarding `Host` unchanged, avoids it _(inference)_.
- No state, no token, no client change. tower-http 0.7 needs only `http` 1, `tower-layer` 0.3 and
  `tower` 0.5, so it layers onto axum 0.8 even though axum's own optional `tower-http` dependency
  still says ^0.6.8 [C-deps] _(inference)_.

### Token crates

- **`axum_csrf` 0.12.0** (2026-10-22): a `CsrfToken` extractor that sets a cookie and hands out an
  `authenticity_token` to embed in a form; the handler calls `token.verify(...)`. Its README
  example renders an Askama template [G-axum-csrf]. A cookie-based double-submit variant aimed at
  server-rendered forms.
- **`axum-csrf-sync-pattern` 0.3.2**: synchronizer token in a custom header, the pattern OWASP
  describes for AJAX, but "built for and works with `axum 0.6.x`, `axum-sessions 0.5.x`"
  [G-axum-csrf-sync]. Not usable with axum 0.8 as released.

### What OWASP says for this shape

- "If your software targets only modern browsers, you may rely on Fetch Metadata headers together
  with the fallback options" and "If an API-driven site can't use `<form>` tags, consider using
  custom request headers" [O-csrf]. Fetch Metadata is supported "in all major browsers since
  March 2023"; a fallback to Origin verification "is a mandatory requirement" [O-csrf].
- Fetch Metadata is only sent to potentially trustworthy URLs (`https`, `localhost`) [O-csrf].
- Rejecting the simple content types (`application/x-www-form-urlencoded`, `multipart/form-data`,
  `text/plain`) closes the form-post path for JSON APIs [O-csrf].
- Login CSRF is real: the cheat sheet asks for pre-session tokens or a custom header on the login
  request [O-csrf].
- SameSite is "defense in depth … not as a replacement for a CSRF token"; it may suffice alone
  only if no sibling host on the registrable domain is untrusted, no `GET` changes state, the
  cookie is `Strict` (or `Lax` + `__Host-`), and Origin checks are in place [O-csrf].
- **SSE.** `EventSource` issues `GET`, which `CsrfLayer` lets through. A cross-site page cannot read
  the stream without CORS, and a `SameSite=Strict` cookie is not sent on its cross-site subresource
  request _(inference)_. The SSE handler must not change state.
- **klicker.ing.** Its ADR 0011 already runs this rule in TypeScript: "Every unsafe method is
  rejected with 403 unless `Sec-Fetch-Site` is `same-origin` or `Origin` equals the server's
  origin, whatever the content type" [K1].

## 6. TOTP and recovery codes

- **`totp-rs` 6.0.0** (2026-08-06), MIT, MSRV 1.88. Configurable algorithm, digits, step and skew;
  features `otpauth` (otpauth URLs), `qr` (base64 PNG), `gen_secret` (CSPRNG secret), `serde`,
  `zeroize`; `migration` (default) keeps the 5.7 API compiling during the move to 6.0's `Builder`
  [G-totp-readme]. Releases: 5.6.0 (2024-07), 5.7.0 (2025-04), 5.7.1 (2026-03), 5.7.2 (2026-06),
  6.0.0 (2026-08); 0 open issues; last push 2026-10-30 [C-api][G-repos].
- **Replay is the caller's job.** `check` / `check_current` return the matched step number:
  "a code should only be accepted once … This library does NOT handle this, and it is the caller's
  responsibility to make sure a specific step only gets accepted once" [G-totp]. So the store
  keeps a `last_used_step` per User.
- **Secret size.** Source comment: "It should be at least 128 bits, but 160 are recommended"
  (RFC 4226) [G-totp]. README warning: some authenticator apps accept SHA-256/512 but fall back to
  SHA-1 silently, which makes `check()` fail [G-totp-readme].
- **History.** RUSTSEC-2022-0018 (CVE-2022-29185): versions before 1.1.0 compared tokens in
  non-constant time; fixed with `constant_time_eq` [W-rustsec-totp].
- **`totp-lite` 2.0.1** (2023-10-30) is a minimal "simple, correct TOTP library" with no release
  since [C-api].
- **Recovery codes.** No crate. OWASP lists "providing the user with a number of single-use
  recovery codes when they first setup MFA" as a recovery option [O-mfa]. Pattern: N random codes
  from a CSPRNG (`rand` 0.10.3, `getrandom`), shown once, stored hashed, deleted on use
  _(inference from [O-mfa] and [O-forgot])_.
- **Store.** TOTP secret (must be readable by the server, so encrypted rather than hashed),
  algorithm/digits/step, `last_used_step`, recovery-code hashes. Plain columns on either database.
- OWASP on changing factors: "Require reauthentication with an existing enrolled factor before
  allowing changes" and notify the User out of band [O-mfa].

## 7. Invite-link tokens

No crate exists for this; the building blocks are `rand` / `getrandom` for the token, a fast hash
(`sha2`) for storage and `subtle` (BSD-3-Clause, 2.6.1) for constant-time comparison [C-api].

OWASP's rules for URL tokens [O-forgot]:
- "Generated using a cryptographically secure random number generator", "Long enough to protect
  against brute-force attacks", "Linked to an individual user", "Invalidated after they have been
  used", "Stored in a secure manner".
- "Don't rely on the Host header while creating the reset URLs … The URL should either be
  hard-coded, or validated against a list of trusted domains."
- The landing page sets `Referrer-Policy: no-referrer`; brute-forcing tokens is rate-limited.

OWASP's session-store guidance applies the same reasoning: a CSPRNG secret with at least 128 bits
("prefer 160 bits or more"), a SHA-256 hash in the database, compared in constant time [O-session].
Putting the token in the URL fragment keeps it out of proxy access logs, because fragments are not
sent to the server; the SPA would read it and POST it _(inference)_.

## 8. Rate limiting sign-in

- **`governor` 0.10.4**: GCRA, state in one `AtomicU64` per key, keyed limiters for arbitrary keys
  [G-governor]. In process memory only; a restart forgets all counters _(inference)_.
- **`tower_governor` 0.8.0** wraps it as a tower layer; `axum` and `tonic` features
  [G-tower-governor]. Key extractors:
  - `PeerIpKeyExtractor` (default): behind a proxy "the peer IP address will _always_ be the
    proxy's IP address", so all clients share one bucket.
  - `SmartIpKeyExtractor`: `x-forwarded-for`, `x-real-ip`, `forwarded` in that order, "Only use
    this key extractor if you can ensure these headers are being set by a trusted provider". The
    code takes the **first parseable** `X-Forwarded-For` entry [G-tower-governor-keys], the one
    the client itself can supply.
  - `GlobalKeyExtractor`, or a custom `KeyExtractor`.
  - Maintenance: last commit 2025-08-14, 25 open issues [G-repos].
- **OWASP.** "The counter of failed logins should be associated with the account itself, rather
  than the source IP address"; exponential lockout is an option; lockout must not become a DoS
  against other Users [O-auth]. Generic error messages for wrong user / wrong password / locked
  account, and watch timing differences [O-auth].
- **Store** _(inference)_: an IP-level limit fits in memory (`governor`); a per-account counter
  that survives restarts needs a `failed_attempts` / `locked_until` pair on the User row.

## 9. Trusting `X-Forwarded-Proto` behind a proxy

- **No crate.** Nothing on crates.io surveyed here validates `X-Forwarded-Proto` against a trusted
  proxy. `axum-extra`'s `Scheme` extractor reads `Forwarded`, then `X-Forwarded-Proto`, then the
  URI, and warns that "user agents can set the `X-Forwarded-Proto` header to arbitrary values"; it
  and `Host` are `#[deprecated = "will be removed in the next version"]` [G-axum-scheme], per
  axum issue #3442 "Remove Host and Scheme extractors from axum-extra" [G-axum-3442].
- **Client IP is solved.** `axum-client-ip` 1.x (on `client-ip` 0.2) removed its "leftmost"
  extractor after a "rightmost IP extraction bug" and now offers `RightmostXForwardedFor`,
  `RightmostForwarded`, `XRealIp`, `CfConnectingIp` and others, plus a runtime-configurable
  `ClientIpSource` "when you ship a pre-compiled binary" [G-axum-client-ip]. It does not cover the
  scheme.
- **What the proxies do.**
  - Caddy sets `X-Forwarded-Proto`, and "by default, the proxy will ignore their values from
    incoming requests, to prevent spoofing" unless `trusted_proxies` is set [W-caddy].
  - nginx passes client headers through and only redefines `Host` and `Connection` by default
    [W-nginx]. Without `proxy_set_header X-Forwarded-Proto $scheme;` a client's own header
    reaches the backend _(inference)_.
- **OWASP** on forwarded headers: "Configure the application to trust only known proxy
  connections, and ensure the trusted proxy removes or overwrites client-supplied forwarded
  headers … Do not trust the header merely because it is present" [O-csrf].
- **Pattern** _(inference)_: accept `X-Forwarded-Proto: https` only when the TCP peer (axum
  `ConnectInfo<SocketAddr>`) is in a configured trusted-proxy set, and/or bind the binary to
  loopback or a Unix socket so only the proxy can reach it. A dozen lines of middleware.
- **Cookies and `--insecure`.** "Insecure sites (`http:`) cannot set cookies with the `Secure`
  attribute. The `https:` requirements are ignored when the `Secure` attribute is set by
  localhost" [W-mdn-cookie]. `__Host-` names require `Secure`, no `Domain`, `Path=/` [W-mdn-cookie];
  OWASP recommends `__Host-` for session IDs [O-session]. Under `--insecure` on a non-localhost
  address the session cookie therefore needs `Secure` off and no `__Host-` prefix, and Fetch
  Metadata headers are absent, leaving `CsrfLayer` on its `Origin`/`Host` rule _(inference)_.

## 10. Passkeys later: webauthn-rs 0.6.2

- **Crates.** `webauthn-rs` (high-level, "opinionated" RP API), `webauthn-rs-core` (low-level),
  `webauthn-rs-proto` (JSON/WASM types), all MPL-2.0 [G-webauthn-readme]. Since 0.6.0 it uses
  RustCrypto instead of OpenSSL. Releases: 0.5.0 (2024-04), 0.5.4 (2025-12), 0.5.5 and 0.6.1-dev
  (2026-04-30), 0.6.2 (2026-10-31); MSRV 1.89; 29 open issues; maintained by the Kanidm team
  [C-api][G-repos]. A 60-day coordinated-disclosure policy is in `SECURITY.md` [G-webauthn-sec].
- **Features.** `attestation` (default), `conditional-ui`, `resident-key-support`,
  `danger-allow-state-serialisation`, `danger-credential-internals`,
  `danger-user-presence-only-security-keys`, `workaround-google-passkey-specific-issues` [C-api].
- **State must stay server-side.** The registration/authentication state "*MUST* be persisted on
  the server. If you store this in a cookie … the client can replay a previous authentication
  state"; serialization is disabled by default and `danger-allow-state-serialisation` re-enables
  it for safe cases such as "serialising to a database, or using a cookie 'memory store'"
  [G-webauthn]. The project's own axum tutorial enables that feature and keeps the state in
  `tower-sessions` 0.13 (axum 0.7) [G-webauthn-tutorial].
- **Store.** The app persists each `Passkey` (serde) per User plus a stable user UUID; one row per
  credential works on either database _(inference)_.
- **Not an RP library.** 1Password's `passkey` 0.6.0 is "a passkey client and authenticator"
  [C-api].

## 11. OIDC relying party later: openidconnect 4.1.0

- **openidconnect 4.1.0** (2026-10-31), MIT, MSRV 1.88; on `oauth2` ^5.0.0 (5.0.0, 2025-01-21).
  Default features `reqwest` + `rustls-tls`; `curl` and `ureq` also available [C-api].
- **Coverage.** Code, implicit and hybrid RP flows; standard claims; UserInfo; RSA, HMAC, ECDSA
  P-256/P-384 and Ed25519 ID-token verification; Discovery provider metadata. Not supported:
  aggregated/distributed claims, request objects, `azp` verification, P-521, JWE, WebFinger
  [G-oidc-readme].
- **Maintenance.** 4.0.0 (2025-01), 4.0.1 (2025-07), 4.1.0 (2026-10); 83 open issues [G-repos].
  It still pulls `rand` 0.8, `sha2` 0.10 and `hmac` 0.12 while the RustCrypto password crates are
  on newer lines, so the graph gets two copies of some crypto crates [C-deps] _(inference)_.
- **Store.** The PKCE verifier, `nonce` and CSRF `state` must survive the redirect, i.e. in the
  server-side session; linked identities need an `(issuer, subject) → User` table _(inference)_.
- **`axum-oidc`** (Codeberg, LGPL-3.0-or-later): "A wrapper for the openidconnect crate for axum";
  stable 0.6.0 is from 2025-01-12, and 1.0.0-dev-2 (2026-05-15) depends on `openidconnect` ^4.0 and
  axum 0.8 [C-api][C-deps]. axum-login also ships an `oauth2` example [G-al-examples].
- OWASP's Authentication cheat sheet has sections on OIDC and on "Secure Federated Account Linking"
  [O-auth].

## 12. OWASP guidance that applies

Each line: what OWASP asks, then where the crates stand.

- **Hash.** Argon2id, m=19 MiB, t=2, p=1 [O-pw]. `argon2` 0.6 default.
- **ID entropy.** At least 64 bits from a CSPRNG [O-session]. tower-sessions: 128 bits.
- **Fixation.** Renew the ID on every privilege change, including a password change [O-session].
  `Session::cycle_id()`; axum-login's `login()` calls it.
- **Cookie.** `Secure`, `HttpOnly`, explicit `SameSite` (Strict preferred), `__Host-` prefix
  [O-session]. tower-sessions defaults cover all but the prefix.
- **Timeouts.** Idle **and** absolute, enforced server-side [O-session]. One `Expiry` per session;
  the absolute deadline is app code.
- **Stored IDs.** If store disclosure is in the threat model, keep a hash, not the token
  [O-session]. The sqlx store keeps raw IDs.
- **CSRF.** Fetch Metadata with an Origin fallback; custom header for APIs [O-csrf]. `tower-http`
  `CsrfLayer`.
- **Login CSRF.** Protect the sign-in request too [O-csrf]. The same layer covers it.
- **Lockout.** Per-account counter, not per IP [O-auth]. Not in any crate.
- **Errors.** Generic sign-in errors; mind timing differences [O-auth]. Handler code.
- **MFA changes.** Re-authenticate with an enrolled factor; notify out of band [O-mfa]. Handler
  code.
- **Recovery.** Single-use recovery codes [O-mfa]. No crate; hashed rows.
- **Invite tokens.** CSPRNG, single-use, expiring, stored hashed, URL not built from `Host`
  [O-forgot]. No crate.
- **Logout.** Flush the session and destroy the ID [O-session]. axum-login `logout()`,
  `Session::delete()`.

## Implications for flock

Facts and trade-offs only; no decision.

- **Two stacks to choose between.**
  - **axum-login + tower-sessions** gives `AuthSession`, guards, fixation handling and
    password-change invalidation for free. Today that means tower-sessions 0.14 and a store at
    0.14, or a git dependency, until axum-login cuts a release on 0.15.
  - **tower-sessions alone** (0.15) plus a few hundred lines of flock's own sign-in code avoids the
    version lag. The TOTP step, Admin suspension and "sign out everywhere" are custom anyway,
    because axum-login has no second-factor state and the stores have no user index.
- **The store question reaches into sessions.** `tower-sessions-sqlx-store` serves SQLite and
  PostgreSQL from one crate, so a later switch is a feature flag. The release lags sqlx 0.9 by six
  months; a project on sqlx 0.9 waits for the next store release or writes its own
  `SessionStore`. A custom store (one table: hashed ID, `user_id`, data, created, expires) is
  small, follows OWASP's verifier-hash advice, and gives the per-User index the stock stores lack.
  If the store is rusqlite rather than sqlx, only a third-party store with 14k downloads exists.
- **CSRF is nearly free.** `tower-http`'s `CsrfLayer` implements the same same-origin rule as
  klicker.ing [K1]. With SPA and API on one origin and Electron loading the SPA from the server,
  every legitimate unsafe request is `Sec-Fetch-Site: same-origin`. The one deployment trap is a
  proxy that rewrites `Host` (nginx's default). Configuring the public origin explicitly removes
  it, and the invite-link builder needs the public origin anyway per OWASP.
- **`X-Forwarded-Proto` is flock's own code.** No crate does it safely. The check "HTTPS only, or
  `--insecure`" is only as strong as the proxy's overwrite of that header; Caddy overwrites by
  default, nginx does not. A trusted-proxy setting (peer address allow-list) or loopback-only
  binding closes the gap. `--insecure` also has to turn off `Secure` on the session cookie, or
  browsers off localhost drop it.
- **Rate limiting needs two layers.** `tower_governor` covers per-IP bursts but is unmaintained
  for 15 months and its smart extractor trusts the leftmost `X-Forwarded-For`. Using `governor`
  directly with a key from `axum-client-ip` (rightmost) avoids both. The per-account lockout OWASP
  asks for lives on the User row either way.
- **TOTP and recovery codes are mostly schema.** `totp-rs` covers generation, verification,
  otpauth URLs and QR codes. The store has to hold an encrypted secret, `last_used_step` against
  replay, and hashed recovery codes. That overlaps with "Credential storage for mail accounts"
  ([dyle71/flock#10](https://github.com/dyle71/flock/issues/10)), which also needs a server-side
  encryption key.
- **Later features fit the same session.** webauthn-rs and openidconnect both need short-lived
  ceremony state in a server-side session; tower-sessions provides it, with webauthn-rs's
  `danger-allow-state-serialisation` feature, which its own axum tutorial uses.
- **MSRV floor.** sqlx 0.9 (1.94), webauthn-rs 0.6 (1.89), totp-rs 6 and openidconnect 4.1 (1.88)
  set the toolchain minimum; sqlx is the highest [C-api].
- **Licenses.** Nothing here blocks GPL-3.0. webauthn-rs (MPL-2.0) and axum-oidc (LGPL-3.0) carry
  file-level or library-level copyleft of their own.

## Sources

crates.io and repositories (queried 2026-11-11)

- [C-api] crates.io API `https://crates.io/api/v1/crates/<name>` for every crate named above
  (versions, dates, licenses, features, `rust_version`, downloads, descriptions)
- [C-deps] crates.io API `https://crates.io/api/v1/crates/<name>/<version>/dependencies` for
  tower-sessions 0.15.0, tower-sessions-core 0.15.0, tower-sessions-sqlx-store 0.15.1,
  -redis-store 0.16.0, -moka-store 0.15.0, -rusqlite-store 0.15.0, axum-login 0.18.0, axum 0.8.9,
  axum-extra 0.12.6, tower-http 0.7.1, axum_csrf 0.12.0, tower_governor 0.8.0, axum-client-ip
  1.3.1, tower-cookies 0.11.0, totp-rs 6.0.0, webauthn-rs 0.6.2, openidconnect 4.1.0, argon2
  0.6.0, password-auth 2.0.0, axum-oidc 1.0.0-dev-2, sqlx-sqlite 0.8.6 / 0.9.0
- [G-repos] GitHub REST API `repos/<owner>/<repo>` (pushed_at, archived, open issues, stars) for
  maxcountryman/tower-sessions, tower-sessions-stores, axum-login; RustCrypto/password-hashes;
  AscendingCreations/AxumCSRF; LeoniePhiline/axum-csrf-sync-pattern; yawn/tower-csrf;
  benwis/tower-governor; boinkor-net/governor; imbolc/axum-client-ip, client-ip, tower-cookies;
  constantoine/totp-rs; fosskers/totp-lite; kanidm/webauthn-rs; ramosbugs/openidconnect-rs,
  oauth2-rs; tokio-rs/axum; 1Password/passkey-rs; rwf2/cookie-rs
- [G-axum] `axum/Cargo.toml` on main (`version = "0.8.9"`): https://github.com/tokio-rs/axum
- [G-axum-extra-cookie] `axum-extra/src/extract/cookie/mod.rs` at `axum-extra-v0.12.6`:
  https://github.com/tokio-rs/axum/tree/axum-extra-v0.12.6/axum-extra/src/extract/cookie
- [G-axum-scheme] `axum-extra/src/extract/scheme.rs` and `host.rs` at `axum-extra-v0.12.6`:
  https://github.com/tokio-rs/axum/blob/axum-extra-v0.12.6/axum-extra/src/extract/scheme.rs
- [G-axum-3442] https://github.com/tokio-rs/axum/issues/3442
- [G-argon2-params] `argon2/src/params.rs`:
  https://github.com/RustCrypto/password-hashes/blob/master/argon2/src/params.rs
- [G-password-auth] `password-auth/README.md`:
  https://github.com/RustCrypto/password-hashes/tree/master/password-auth
- [G-ts-readme] https://github.com/maxcountryman/tower-sessions (README)
- [G-ts-service] `tower-sessions/src/service.rs` (`SessionConfig::default`):
  https://github.com/maxcountryman/tower-sessions/blob/main/src/service.rs
- [G-ts-session] `tower-sessions-core/src/session.rs` (`cycle_id`, `Id`, `Expiry`):
  https://github.com/maxcountryman/tower-sessions/blob/main/tower-sessions-core/src/session.rs
- [G-ts-store-trait] `tower-sessions-core/src/session_store.rs` (`ExpiredDeletion`)
- [G-ts-stores] https://github.com/maxcountryman/tower-sessions-stores (README)
- [G-sqlx-store] `sqlx-store/Cargo.toml`, `src/sqlite_store.rs`, `src/postgres_store.rs` on main:
  https://github.com/maxcountryman/tower-sessions-stores/tree/main/sqlx-store
- [G-sqlx-store-readme] `sqlx-store/README.md`, same repository
- [G-axum-sessions] https://github.com/maxcountryman/axum-sessions (README)
- [G-al-readme] https://github.com/maxcountryman/axum-login (README)
- [G-al-session] `axum-login/src/session.rs` (`login`, `logout`, `from_session`):
  https://github.com/maxcountryman/axum-login/blob/main/axum-login/src/session.rs
- [G-al-cargo] `axum-login/Cargo.toml` on main, same repository
- [G-al-example] `examples/sqlite/src/users.rs`, same repository
- [G-al-examples] `examples/` directory listing, same repository
- [G-th-csrf] `tower-http/src/csrf/mod.rs`:
  https://github.com/tower-rs/tower-http/blob/main/tower-http/src/csrf/mod.rs
- [G-th-csrf-layer] `tower-http/src/csrf/layer.rs`, same repository
- [G-th-changelog] `tower-http/CHANGELOG.md` (0.7.0) and PR #699:
  https://github.com/tower-rs/tower-http/pull/699
- [G-tower-csrf] https://github.com/yawn/tower-csrf (README)
- [G-axum-csrf] https://github.com/AscendingCreations/AxumCSRF (README)
- [G-axum-csrf-sync] https://github.com/LeoniePhiline/axum-csrf-sync-pattern (README)
- [G-totp] `src/lib.rs` (`check`, `check_current`, field docs):
  https://github.com/constantoine/totp-rs/blob/master/src/lib.rs
- [G-totp-readme] https://github.com/constantoine/totp-rs (README)
- [G-governor] `governor/README.md`: https://github.com/boinkor-net/governor
- [G-tower-governor] https://github.com/benwis/tower-governor (README)
- [G-tower-governor-keys] `src/key_extractor.rs`, same repository
- [G-axum-client-ip] READMEs: https://github.com/imbolc/axum-client-ip and
  https://github.com/imbolc/client-ip
- [G-webauthn-readme] https://github.com/kanidm/webauthn-rs (README)
- [G-webauthn] `webauthn-rs/src/lib.rs` (module docs on state serialization):
  https://github.com/kanidm/webauthn-rs/blob/master/webauthn-rs/src/lib.rs
- [G-webauthn-tutorial] `tutorial/server/axum/Cargo.toml`, same repository
- [G-webauthn-sec] `SECURITY.md`, same repository
- [G-oidc-readme] https://github.com/ramosbugs/openidconnect-rs (README)

OWASP Cheat Sheet Series (Markdown sources at https://github.com/OWASP/CheatSheetSeries, read
2026-11-11)

- [O-pw] Password Storage:
  https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- [O-session] Session Management (Session ID Entropy, Server-Side Session Token Storage, Cookies,
  Renew the Session ID After Any Privilege Level Change, Automatic Session Expiration):
  https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- [O-csrf] Cross-Site Request Forgery Prevention (Fetch Metadata headers, Employing Custom Request
  Headers, SameSite, Identifying the Target Origin, Login Forms):
  https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- [O-auth] Authentication (Password Strength, Authentication Responses, Account Lockout, OIDC):
  https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- [O-mfa] Multifactor Authentication (Resetting MFA, Changing MFA Factors):
  https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html
- [O-forgot] Forgot Password (General Security Practices, URL Tokens):
  https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html

Other

- [W-fsf] FSF, "Various Licenses and Comments about Them":
  https://www.gnu.org/licenses/license-list.html
- [W-mdn-cookie] MDN `Set-Cookie` (Secure, cookie prefixes), mdn/content
  `files/en-us/web/http/reference/headers/set-cookie/index.md`:
  https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie
- [W-nginx] nginx `proxy_set_header`:
  https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header
- [W-caddy] Caddy `reverse_proxy`, Defaults and `trusted_proxies`:
  https://caddyserver.com/docs/caddyfile/directives/reverse_proxy#defaults
- [W-rustsec-totp] RUSTSEC-2022-0018: https://rustsec.org/advisories/RUSTSEC-2022-0018.html

klicker.ing (read-only, `/home/dyle/Source/headcode.space/klicker.ing/main`)

- [K1] `docs/adr/0011-rest-api-described-by-openapi-no-server-functions.md` (cookies only,
  same-origin rule for unsafe methods)
