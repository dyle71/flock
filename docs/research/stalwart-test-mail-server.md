# Stalwart as flock's test and development mail server

Research for [dyle71/flock#18](https://github.com/dyle71/flock/issues/18), part of the wayfinder map
[dyle71/flock#1](https://github.com/dyle71/flock/issues/1). It checks the choice settled in
[dyle71/flock#14](https://github.com/dyle71/flock/issues/14): Stalwart as the main mail server of
the throwaway integration test stack and of the development stack (dev ports 3802 SMTP, 3803 IMAP,
3804 POP3, 3805 web admin). This document states facts and trade-offs; it records no decision.

Researched 2026-11-11. Current release: Stalwart **v1.1.0** (2026-11-09). Source read at tag
`v1.1.0` (commit `e7cde9b3`), documentation read from the `stalwartlabs/website` repository at
commit `392011b2` (2026-11-09), which is the source of stalw.art/docs. Statements marked
_(inference)_ are my own reasoning from the cited facts; no source states them directly. No image
was pulled and no container was run, so nothing here is measured.

## Question

Can Stalwart, run as one container, serve as flock's mail server for the throwaway integration
test stack and for development? Which IMAP extensions does it support, does it serve POP3, SMTP
submission and JMAP, how is it configured headless and reproducibly (accounts, passwords, aliases,
TLS, OAuth), can one message sit in several mailboxes, how fast does it start and reset, and what
are its image, version, cadence and license?

## Summary

- **Protocols.** One binary serves IMAP4rev2/rev1, POP3, SMTP (MTA and submission), JMAP,
  ManageSieve and HTTP. Mail submitted for a hosted domain is delivered into the local mailbox,
  so a test mail sent by one test account arrives in another.
- **IMAP.** After login it advertises IDLE, CONDSTORE, QRESYNC, UIDPLUS, MOVE, SPECIAL-USE,
  CREATE-SPECIAL-USE, LIST-EXTENDED, LIST-STATUS, NAMESPACE, QUOTA, ACL, ENABLE, ESEARCH, SORT,
  THREAD, BINARY, UIDONLY, OBJECTID+, PREVIEW, METADATA and more. SELECT returns
  `PERMANENTFLAGS (... \*)`, so arbitrary keywords work (limit 100 per message, 128 bytes each).
  **COMPRESS=DEFLATE is not implemented**; it is listed as a planned extension, as is NOTIFY.
- **POP3.** CAPA lists `TOP`, `UIDL`, `RESP-CODES`, `PIPELINING`, `EXPIRE NEVER`, `UTF8`, `SASL`,
  `USER`, and `STLS` on listeners with TLS. POP3 serves the Inbox only.
- **Several mailboxes.** Yes. A message is one object with a set of mailbox memberships. IMAP COPY
  inside an account adds a membership instead of a new message, and keywords belong to the message,
  not to the membership. This matches flock's Gmail-label model, and it deviates from the usual
  IMAP expectation that a copy has its own flags.
- **Configuration.** Since v0.16 there is no TOML. A one-line `config.json` names the data store;
  everything else (domains, accounts, listeners, settings) lives in the data store and is changed
  through a JMAP management API, the `stalwart-cli apply` plan format, or the WebUI. Headless setup
  is documented: write `config.json`, start in recovery mode with a recovery admin from the
  environment, apply a plan, restart.
- **Test-hostile defaults.** Default listeners are TLS-only for mail (993, 995, 465, plus 25 with
  STARTTLS); 143, 110 and 587 must be added. Password authentication on cleartext is off by default,
  OAuth bearer and API keys are refused on cleartext always, the WebUI and the spam rules are
  downloaded from GitHub at first start, and auto-ban blocks an address after 100 failed sign-ins
  or 30 malformed unauthenticated commands per day.
- **Image.** `stalwartlabs/stalwart` on Docker Hub and `ghcr.io/stalwartlabs/stalwart`; tags such as
  `v1.1.0`, `v1.1`, `v1`, `latest`, each also as `-alpine`. Compressed size about 124 MB (Debian)
  or 100 MB (Alpine) for amd64.
- **Cadence.** Weekly patch releases through 2026; two releases with breaking changes in the last
  seven months (v0.16.0 rewrote configuration, v1.0.0 changed the database layout).
- **License.** AGPL-3.0-only OR the proprietary SELv2. The same binary runs as the Community
  edition without a license key. Using an unmodified Stalwart image as a test dependency puts no
  obligation on flock's GPL-3.0 code.

## 1. Image, version and release cadence

**Image.** The docs name `stalwartlabs/stalwart` on Docker Hub and `ghcr.io/stalwartlabs/stalwart`
on GHCR, for amd64, arm64, ARMv7 and ARMv6, on Debian slim, with every tag also built on Alpine
(`-alpine` suffix). They recommend the major tag `v1` and advise against `latest` [D-docker]. Docker
Hub on 2026-11-11 lists `latest`, `v1`, `v1.1`, `v1.1.0` (and `-alpine` variants) all pushed
2026-11-09; `v1.1.0` has the manifest-list digest
`sha256:f772aa6ef8cd8018969fa491640f1436b0270eb14e2c219b5f37f1a77020837e` [H-tags].

| Tag | amd64 | arm64 | Base |
|---|---|---|---|
| `v1.1.0` | 123.9 MB | 117.9 MB | Debian trixie-slim |
| `v1.1.0-alpine` | 99.8 MB | 94.8 MB | Alpine |

Sizes are Docker Hub's compressed sizes [H-tags]. The image runs as user `stalwart` (uid 2000),
declares volumes `/etc/stalwart` and `/var/lib/stalwart`, exposes 443, 25, 110, 587, 465, 143, 993,
995, 4190 and 8080, gives the binary `cap_net_bind_service`, includes `curl`, and defines a
health check that probes `https://127.0.0.1:443/healthz/live` (overridable with
`STALWART_HEALTHCHECK_URL`) and then `http://127.0.0.1:8080/healthz/live`. Its default command is
`stalwart --config /etc/stalwart/config.json` [G-dockerfile]. The official image is built with the
`enterprise` feature [G-dockerfile]; without a license key it runs as the Community edition
[D-enterprise].

**CLI image.** `stalwartlabs/cli` (and `ghcr.io/stalwartlabs/cli`), current `1.0.14`, about 4 MB
compressed [D-cli], [H-cli]. The server image does not contain the CLI [G-dockerfile].

**Versions.** From the changelog [G-changelog]:

| Release | Date | Note |
|---|---|---|
| v0.15.0 | 2025-12-16 | |
| v0.16.0 | 2026-04-20 | Breaking: TOML replaced by data-store configuration and a JMAP API |
| v0.16.1 to v0.16.27 | 2026-04-25 to 2026-10-18 | 27 patch releases, about weekly |
| v1.0.0 | 2026-10-26 | Breaking: new database layout, no in-place upgrade from 0.16 |
| v1.0.1 to v1.0.3 | 2026-10-28 to 2026-11-04 | |
| v1.1.0 | 2026-11-09 | Passkeys, Stalwart Keep |

The settings schema was renamed and renumbered in v1.0.0 [G-changelog], and v0.16.0 made all
earlier configuration incompatible [G-upgrading]. Several recent patch releases also ask
administrators to act on upgrade (re-enable encryption at rest in 1.0.2, add new permissions to
default roles in 1.0.3 and 1.1.0) [G-changelog].

**Development model.** Since v1.0.0, the unit, integration, conformance and stress test suites are
kept in a private repository, and new features are developed privately and published as one commit
on release; the source of every release stays public under the AGPL [G-changelog], [B-protect].
Bug reports opened as GitHub issues are closed automatically and redirected to support.stalw.art
[G-issue-3299].

## 2. IMAP

The capability list is built in `crates/imap-proto/src/protocol/capability.rs` [G-capability].
Before authentication the server advertises `IMAP4rev2 IMAP4rev1 ENABLE SASL-IR LITERAL+ ID
UTF8=ACCEPT`, plus `AUTH=` mechanisms or `LOGINDISABLED`, plus `STARTTLS` when TLS is offered. The
full list appears only after authentication, and the tagged `OK` of `AUTHENTICATE` carries it as a
`CAPABILITY` response code [G-authenticate].

| Extension asked for | Status | Notes |
|---|---|---|
| IDLE (RFC 2177) | Yes | Idle timeout default 30 min; no server keepalive during IDLE [G-idle] |
| CONDSTORE, QRESYNC (RFC 7162) | Yes | |
| UIDPLUS (RFC 4315) | Yes | |
| MOVE (RFC 6851) | Yes | |
| COMPRESS=DEFLATE (RFC 4978) | **No** | "Planned Extensions" [D-rfcs]; no code in the IMAP crates |
| SPECIAL-USE, CREATE-SPECIAL-USE (RFC 6154) | Yes | |
| LIST-EXTENDED (RFC 5258), LIST-STATUS (RFC 5819) | Yes | |
| Arbitrary keywords | Yes | `PERMANENTFLAGS` ends in `\*` when the ACL allows [G-select] |
| NAMESPACE (RFC 2342) | Yes | |
| QUOTA (RFC 9208) | Yes | `QUOTA=RES-STORAGE`; `QUOTA=SET` in code, not advertised [G-capability] |
| ACL (RFC 4314) | Yes | With `RIGHTS=texkn` |

Further capabilities after login: `JMAPACCESS`, `CHILDREN`, `MULTIAPPEND`, `BINARY`, `UNSELECT`,
`ESEARCH`, `WITHIN`, `SEARCHRES`, `SORT`, `THREAD=REFERENCES`, `ESORT`, `SORT=DISPLAY`,
`UNAUTHENTICATE`, `STATUS=SIZE`, `OBJECTID+`, `PREVIEW`, `UIDONLY`, `UIDBATCHES`, `MESSAGELIMIT=`,
`SAVELIMIT=` [G-capability], and `METADATA`/`LIST-METADATA` for accounts with the
`imapMetadataGet` permission [G-capability-op], [D-imap]. The docs' RFC list also names RFC 2180,
2683, 5530, 6855, 8457, 9051, 9586, 9590, 9698, 9738, 9979, 10022 and the objectid-bis draft. Its
"Planned Extensions" are IMAP URL, URLAUTH, CATENATE, COMPRESS, I18NLEVEL, NOTIFY, URLFETCH
binary, and Sieve IMAP events [D-rfcs]. A long-open request (#484) also lists REPLACE,
APPENDLIMIT, multimailbox SEARCH and fuzzy SEARCH as missing [G-issue-484].

**Keyword limits.** `maxFlagsPerEmail` default 100, `maxFlagLength` default 128 bytes (max 1024)
[D-ref-email].

**Mailbox names.** New accounts get Inbox, `Deleted Items`, `Junk Mail`, `Drafts` and `Sent Items`
unless `defaultFolders` on the Email singleton says otherwise; folders can have alias names that
resolve to the special-use folder [D-imap].

**Greeting.** Fixed: `Stalwart IMAP4rev2 at your service.` [D-imap].

### Deviations and quirks relevant to a sync engine

- **COPY is a membership, not a copy.** For a COPY or MOVE inside one account, the server adds the
  destination mailbox to the existing message (`new_data.add_mailbox(...)`), and skips messages
  already in the destination [G-copy]. Keywords are stored once per message (`MessageData` holds
  `mailboxes` and `keywords` side by side) [G-messagedata]. So a `STORE +FLAGS` in one mailbox
  shows up in every mailbox the message belongs to _(inference from the data model)_. The docs say
  "JMAP allows a message to be in several mailboxes at once; IMAP clients see each membership as a
  copy" [D-jmap-limits]. A message may belong to at most 100 mailboxes (`maxMailboxesPerEmail`)
  [D-ref-email].
- **No IDLE keepalive.** The IDLE loop only waits for input or the idle timeout [G-idle]; a user
  report about NAT drops was auto-closed [G-issue-3299]. Irrelevant inside one Docker network.
- **Capabilities change at login.** Clients must use the post-login list, as above.
- **Bug channel.** Reported IMAP bugs from 2026 (UIDONLY with Outlook, BODYSTRUCTURE decoding,
  untagged `NO` on LOGIN) appear on GitHub only as auto-closed issues, so their state has to be
  checked on support.stalw.art [G-issues].

## 3. POP3

`CAPA` lists `USER` (when PLAIN is allowed), `SASL <mechanisms>`, `STLS` (when the listener has
TLS), `TOP`, `RESP-CODES`, `PIPELINING`, `EXPIRE NEVER`, `UIDL`, `UTF8` and `IMPLEMENTATION
Stalwart Server` [G-pop3-capa]. The maildrop is the Inbox [G-pop3-mailbox]. A UIDL is the Inbox
UIDVALIDITY followed by the message UID [G-pop3-uidl]. `DELE` marks, `RSET` unmarks, and marked
messages are deleted at `QUIT` [G-pop3-dele]. Supported RFCs: 1939, 2449, 2595, 3206, 5034, 6856
[D-rfcs]. POP3 shares authentication, timeout and limit settings with IMAP (the `Imap` singleton)
[D-imap].

## 4. SMTP submission and local delivery

- **Submission.** Default listener `submissions` on 465 with implicit TLS; 587 with STARTTLS is not
  created by default [D-listener]. AUTH offers `PLAIN`, `LOGIN`, `OAUTHBEARER` and `XOAUTH2` on TLS
  connections to any port except 25. The `saslMechanisms` expression decides this and can be
  changed; the `require` expression (default `local_port != 25`) forces AUTH before `MAIL FROM`
  [D-auth-stage].
- **Sender check.** `mustMatchSender` (default `true`) only lets an authenticated account use its
  own address, its aliases, or addresses of groups it belongs to as envelope sender; anything else
  gets `501 5.5.4` [D-auth-stage]. This is the server-side counterpart of flock's Identities.
- **Local delivery.** The default route `local` stores mail for hosted domains in the recipient's
  mailbox, running Sieve and the spam verdict there [D-routing]. The spam filter runs only for
  unauthenticated sessions by default (`is_empty(authenticated_as)`), so authenticated submissions
  skip it [D-spam]. A test mail submitted by account A to account B therefore lands in B's Inbox
  without external DNS _(inference: local route plus no spam filter for authenticated mail)_.
- **Sent copies.** Submission does not file the message in Sent; the client appends it
  _(inference: no such behavior is documented for SMTP; JMAP has EmailSubmission for this)_.
- **Supported RFCs** include 5321, 4954, 3207, 6152, 1870, 3030, 2920, 3461, 2034, 6531, 8689 and
  LMTP (2033) [D-rfcs].

## 5. JMAP

Stalwart implements JMAP core (RFC 8620), Mail (RFC 8621), WebSocket (RFC 8887), Blob (RFC 9404),
Quotas (RFC 9425), Sieve (RFC 9661), Web Push VAPID (RFC 9749), and the metadata and mail-sharing
drafts [D-rfcs]. The same endpoint `/jmap` also carries the management API (`urn:stalwart:jmap`,
methods such as `x:Account/set`) [D-ref-account], [G-upgrading]. IMAP advertises `JMAPACCESS`
(RFC 9698) [G-capability].

## 6. Headless, reproducible configuration

### How configuration is stored

`config.json` holds only the data store, for example
`{"@type":"RocksDb","path":"/var/lib/stalwart/"}`. Everything else is a JMAP object in that
store [D-declarative], [G-upgrading]. Data store types: RocksDB, SQLite, FoundationDB, PostgreSQL,
MySQL [D-ref-datastore]. Without `config.json` the server starts in **bootstrap mode**: it opens
only plain HTTP on 8080 (or `STALWART_RECOVERY_MODE_PORT`), prints a random admin password to
stderr, and accepts only the `Bootstrap` object (the setup wizard's fields) [D-bootstrap],
[G-registry-build].

### Two documented headless paths

1. **Bootstrap object.** In bootstrap mode, a tool submits the `Bootstrap` object through the JMAP
   API instead of using the wizard; the server writes `config.json` and needs a restart [D-docker],
   [D-bootstrap].
2. **config.json plus plan.** Write `config.json`, start with `STALWART_RECOVERY_MODE=1` and
   `STALWART_RECOVERY_ADMIN=admin:<password>`, run `stalwart-cli apply --file plan.ndjson` against
   `http://<host>:8080`, then restart without the recovery variables. If the plan defines no
   listener, the default listeners are created at that restart [D-declarative]. Plans are NDJSON;
   `upsert` with `matchOn` and `update` of singletons make them idempotent; `stalwart-cli snapshot`
   exports a running server as a plan [D-declarative], [D-plan].

The restart is needed because listeners, stores and caches are read only at startup. Accounts,
domains and roles take effect immediately; most other settings after a `ReloadSettings` action;
certificates after `ReloadTlsCertificates` [D-config].

On a normal start with no objects, the server creates defaults: the WebUI application, MTA
queues, routes and strategies, JMAP settings, default roles, the seven default listeners, the ASN
lookup, and a file log tracer at `/var/log/stalwart` (which writes nothing in a container, so a
plan should define a console tracer) [G-defaults], [D-docker].

### Environment variables

`CONFIG_PATH`, `STALWART_RECOVERY_MODE`, `STALWART_RECOVERY_MODE_PORT`,
`STALWART_RECOVERY_MODE_LOG_LEVEL`, `STALWART_RECOVERY_ADMIN`, `STALWART_UPDATE_ROLE_PERMISSIONS`,
`STALWART_PUBLIC_URL`, `STALWART_HTTPS_PORT`, `STALWART_HOSTNAME`, `STALWART_ROLE`,
`STALWART_PUSH_SHARD`, `RUN_AS_USER`, `RUN_AS_GROUP`, `TOKIO_WORKER_THREADS`, `EXPORT_TYPES`, and in
the image `STALWART_HEALTHCHECK_URL` [D-env], [G-registry-local]. `STALWART_RECOVERY_ADMIN` is
accepted in **every** mode, including normal operation, as a full-access credential that can also
act as master user for any account [D-env], [D-recovery]. For a throwaway test stack that is a
convenient fixed admin; for anything reachable from outside it is a standing backdoor.

### Accounts, passwords, aliases

An account is an `x:Account` object of `@type` `User` with `name` (local part), `domainId`,
`credentials` (a list of `Password`, `AppPassword`, `ApiKey` or `Passkey`) and `aliases` (each with
`name`, `domainId`, `enable`, `description`) [D-ref-account]. Account names are full e-mail
addresses since v0.16; a bare user name gets the default domain appended at login [G-upgrading].
Sign-in with an alias address needs the `authenticateWithAlias` permission, which the default User
role has [D-authn]. Password policy defaults: at least 8 characters and zxcvbn strength `three`,
hashed with argon2id; `minPasswordStrength` can go down to `zero`, and bcrypt, scrypt and pbkdf2 are
alternatives [D-ref-authn]. One password sign-in costs about 26 to 34 ms of CPU [D-requirements].

### TLS and plaintext for tests

- **Listeners.** Defaults: `smtp` 25 (STARTTLS), `submissions` 465, `imaps` 993, `pop3s` 995 (all
  implicit TLS), `sieve` 4190 (STARTTLS), `https` 443, `http` 8080 (plain). No 587, 143 or 110 by
  default [D-listener], [G-defaults]. A listener with `useTls: false` offers no STARTTLS
  [D-listener].
- **Plaintext authentication.** `Imap.allowPlainTextAuth` (default off) controls IMAP, POP3 and
  ManageSieve; with it off, IMAP announces `LOGINDISABLED` and POP3 refuses all logins on
  cleartext. SMTP uses the `saslMechanisms` expression instead [D-imap], [D-authn].
- **Certificates.** With no certificate loaded, the server presents a self-signed certificate for
  `localhost` [D-certs]. A `Certificate` object takes PEM chain and key inline, from an environment
  variable, or from a file path, so a test CA's certificate can be mounted [D-certs].
- **ACME** only runs when a domain asks for it; the wizard path enables it by default, a plan sets
  `certificateManagement` per domain (the docs' example uses `Manual`) [D-declarative].

### OAuth bearer

- `OAUTHBEARER` and `XOAUTH2` are offered on IMAP, POP3, SMTP and ManageSieve **only over TLS**, and
  no setting relaxes this [D-authn].
- They accept OAuth access tokens and **API keys**. An API key is an `ApiKey` credential created
  through the API; its secret (`API_...`) is returned once and works as a bearer token over HTTP
  and through those SASL mechanisms; it has no scopes, so the `mail` scope check passes; default
  limit 5 keys per account [D-apikey], [D-authn]. This gives tests a headless bearer credential.
- Real OAuth tokens come from Stalwart's own authorization server: grants `authorization_code`
  (with PKCE), `refresh_token`, and the device flow; no password or client-credentials grant
  [D-oauth]. Both flows need a user to sign in on `/login` or `/device`, so a headless test would
  have to drive that page _(inference)_. The WebUI only accepts sign-in on the public HTTPS URL, or
  over plain HTTP in recovery or bootstrap mode [D-docker], [D-env].

### Network access the server attempts

At start the server downloads the WebUI bundle from GitHub (60 s timeout, awaited in
`start_services` before connections are served) and schedules a background task to download the
spam filter rule set [D-requirements], [G-services], [G-application], [G-defaults]. It also
downloads ASN lists daily and needs DNS for MX/SPF/DKIM/DMARC lookups [D-requirements]. If the
WebUI download fails, `/admin` returns 404 and the rest works [D-webui]. The WebUI can be loaded
from a `file://` URL instead [D-webui]. In a stack whose network is `internal: true`, the WebUI
fetch fails and may hold start-up for up to the 60 s timeout, depending on how fast the connection
attempt fails _(inference)_.

### Auto-ban

Defaults: ban after 100 authentication failures, 35 failed `RCPT TO`, 150 loitering sessions or 30
scanning events (unknown commands or unparsable input before authentication) per day, per IP, with
no expiry; the `allowed-ip` list exempts addresses [D-autoban]. A test suite that exercises wrong
passwords or malformed commands from one backend container can hit these limits _(inference)_.

## 7. Start-up time and resetting state

- **Start-up time.** Neither the docs nor the changelog give a figure. The image's health check
  allows a 30 s start period [G-dockerfile]. RocksDB is embedded, so there is no separate database
  to wait for; the WebUI download is the only network step on the start path [G-services]
  _(inference)_. A measurement is needed.
- **Health endpoints.** `/healthz/live` and `/healthz/ready`; the latter returns 503 while the data
  store does not answer [D-docker], [G-changelog].
- **Reset options** _(inference, from the configuration model above)_:
  - Fresh container and fresh volume per test run (as klicker.ing's `docker-compose.test.yml` does
    with its own Postgres volume [K-compose]). Costs a full first start plus a configuration
    pass on every run.
  - Provision once into a seed data directory (recovery mode plus `apply`), then copy the seed
    into each run's volume or tmpfs. Start then reads ready configuration; only the restart cost
    remains.
  - Keep one server per run and isolate tests by account: accounts take effect immediately
    [D-config] and can be created and destroyed through `x:Account/set` [D-ref-account]. Cheapest,
    but state left by a failed test stays until the run ends.
- **Account deletion.** `x:Account/set` with `destroy` exists [D-ref-account]; how quickly mail
  data of a destroyed account is purged is not documented in the pages read.

## 8. License

- Every source file is `AGPL-3.0-only OR LicenseRef-SEL`, except Enterprise code blocks marked
  `LicenseRef-SEL` only, and a few Apache-2.0/MIT files [G-license], [G-readme].
- SELv2 requires a paid subscription and a license key; §4.5 allows copying the source for
  development and testing without one. An instance without a valid key runs as the Community
  edition [G-sel], [D-enterprise].
- Stalwart's own page: "The AGPL-3.0 obligations apply only when modified versions of the server
  are themselves redistributed or offered as a network service; running unmodified Stalwart, or
  using it internally, does not trigger any source-disclosure requirement" [W-open-source].
- Stalwart stated in October 2026 that it will not move to a source-available license
  [B-protect].
- For flock _(inference)_: flock's code talks to Stalwart only over network protocols and does
  not link or ship it. A Compose file that names the public image does not redistribute it.
  Running an unmodified image for tests and development therefore puts no license obligation on
  flock. If flock ever distributed a Stalwart image or a modified binary, AGPL terms would apply to
  that artifact. GPL-3.0 §13 permits combining with AGPL-3.0 code in any case.

## Implications for flock

Facts and trade-offs only; no decision.

- **Fit.** Every protocol flock v1 needs (IMAP, POP3, SMTP submission) and the later JMAP are in
  one container, and local delivery closes the send-and-receive loop without external DNS. The IMAP
  extensions flock's sync engine is likely to use (IDLE, CONDSTORE, QRESYNC, UIDPLUS, MOVE,
  SPECIAL-USE, LIST-EXTENDED, keywords) are all present.
- **COMPRESS gap.** COMPRESS=DEFLATE cannot be tested against Stalwart. If flock implements it,
  that code path needs another server (Dovecot, already planned as the second server) or a
  test of its own.
- **Folder model.** Stalwart behaves like Gmail labels: one message in several mailboxes, keywords
  shared across them. That makes it a good fixture for flock's "a message may sit in several
  Folders" rule, and a poor fixture for the classic IMAP case where each copy has its own flags
  and identity. Tests of the classic case need Dovecot. A sync engine tuned only on Stalwart may
  miss the classic case, and the reverse.
- **Tags.** Arbitrary keywords via `PERMANENTFLAGS \*` cover Tags; the 100-per-message and
  128-byte limits are server limits flock may meet in tests.
- **Configuration effort.** No single file configures the server. A reproducible stack needs a
  `config.json`, an NDJSON plan (domain, accounts, aliases, listeners for 143/110/587, plaintext
  auth or a test certificate, console tracer, relaxed password policy, auto-ban allow-list,
  WebUI source), and a start-apply-restart sequence or a pre-built seed. The plan format follows
  the settings schema, which was renamed in v1.0.0; a version bump can break the plan.
- **TLS trade-off.** Plaintext listeners keep tests simple but cannot exercise OAUTHBEARER/XOAUTH2,
  which Stalwart refuses without TLS. A test CA plus a mounted certificate covers both, at the cost
  of trust setup in flock's test client.
- **OAuth testing.** API keys give a headless bearer credential for the SASL path. A full OAuth
  authorization-code flow against Stalwart needs browser-page automation.
- **Isolated network.** The `internal: true` network in the klicker.ing pattern blocks the WebUI
  and spam-rule downloads; serving the WebUI from a local file or disabling it, and not relying on
  the spam filter, avoid start-up delay and noise. Pinning the image by digest, as klicker.ing
  does for its services, guards against the frequent releases.
- **Development stack.** Dev ports 3802 to 3805 map onto container listeners. The WebUI on 3805
  accepts sign-in only on the public HTTPS URL, so it needs `STALWART_PUBLIC_URL` set to that URL
  and a certificate the browser accepts, or recovery mode, which stops mail services [D-docker].
- **Upstream risk.** Fast cadence, two breaking releases in 2026, private test suites, and bug
  reports routed away from GitHub make upstream behavior harder to audit. Pinning a version and
  keeping flock's own protocol tests is the counterweight.
- **Auto-ban.** Negative tests (wrong password, garbage commands) can ban the test client's address
  for the rest of the run unless the stack allow-lists it or raises the rates.
- **License.** Using the unmodified image as a test and dev dependency imposes nothing on flock's
  GPL-3.0 code.

## Sources

Stalwart server source, tag `v1.1.0`, commit `e7cde9b3fac32d2a254ef782efec56f2373ec5d4`,
https://github.com/stalwartlabs/stalwart (paths relative to the repository root)

- [G-readme] `README.md` (License)
- [G-license] `LICENSES/` and SPDX headers across `crates/`
- [G-sel] `LICENSES/LicenseRef-SEL.txt` (§2, §3, §4.5)
- [G-changelog] `CHANGELOG.md` (entries 0.14.1 to 1.1.0)
- [G-upgrading] `UPGRADING/v0_16.md`
- [G-dockerfile] `Dockerfile`
- [G-capability] `crates/imap-proto/src/protocol/capability.rs`
- [G-capability-op] `crates/imap/src/op/capability.rs`
- [G-authenticate] `crates/imap/src/op/authenticate.rs` (CAPABILITY response code)
- [G-select] `crates/imap-proto/src/protocol/select.rs` (PERMANENTFLAGS)
- [G-copy] `crates/imap/src/op/copy_move.rs` (same-account branch)
- [G-messagedata] `crates/email/src/message/messagedata.rs` (`MessageData`)
- [G-idle] `crates/imap/src/op/idle.rs`
- [G-pop3-capa] `crates/pop3/src/protocol/response.rs` (`Response::Capability`)
- [G-pop3-mailbox] `crates/pop3/src/mailbox.rs`
- [G-pop3-uidl] `crates/pop3/src/op/list.rs` (`handle_uidl`)
- [G-pop3-dele] `crates/pop3/src/op/delete.rs`
- [G-registry-local] `crates/store/src/registry/local.rs` (environment variables)
- [G-registry-build] `crates/store/src/build/registry.rs` (bootstrap mode)
- [G-defaults] `crates/common/src/manager/defaults.rs` (`insert_safe_defaults`)
- [G-application] `crates/common/src/manager/application.rs`, `crates/common/src/manager/mod.rs`
  (`fetch_resource`)
- [G-services] `crates/services/src/lib.rs`, `crates/main/src/main.rs` (start-up order)

Stalwart documentation, `stalwartlabs/website` commit `392011b21ab8c0c9489cad580dba98cebbccb880`,
published at https://stalw.art/docs/ (paths under `src/content/docs/docs/`)

- [D-docker] `install/platform/docker.md` (https://stalw.art/docs/install/platform/docker)
- [D-requirements] `install/requirements.md`
- [D-bootstrap] `configuration/bootstrap-mode.md`
- [D-recovery] `configuration/recovery-mode.md`
- [D-declarative] `configuration/declarative-deployments.md`
- [D-env] `configuration/environment-variables.md`
- [D-config] `configuration/index.md` (Applying changes)
- [D-plan] `management/cli/plan-format.md`
- [D-cli] `management/cli/index.md`
- [D-webui] `management/webui/index.md` (Outbound network requirement)
- [D-listener] `server/listener.md` (Default listeners, Implicit TLS and STARTTLS)
- [D-certs] `server/tls/certificates.md`
- [D-autoban] `server/auto-ban.md`
- [D-enterprise] `server/enterprise.md`
- [D-imap] `email/settings/imap.md`
- [D-jmap-limits] `email/settings/jmap.md`
- [D-rfcs] `development/rfcs.md` (https://stalw.art/docs/development/rfcs)
- [D-authn] `auth/authentication/index.md`
- [D-apikey] `auth/authentication/api-key.md`
- [D-oauth] `auth/oauth/authorization.md`
- [D-auth-stage] `mta/inbound/auth.md`
- [D-routing] `mta/outbound/routing.md`
- [D-spam] `spamfilter/index.md`
- [D-ref-account] `ref/object/account.md`
- [D-ref-authn] `ref/object/authentication.md`
- [D-ref-email] `ref/object/email.md`
- [D-ref-datastore] `ref/object/data-store.md`

Other Stalwart sources

- [B-protect] Blog, "Private Tests, Public Code: Protecting Stalwart from AI Clones", 2026-10-16,
  `src/content/docs/blog/protecting-stalwart.mdx`, https://stalw.art/blog/protecting-stalwart/
- [W-open-source] Website page "Community Edition", `src/content/pages/open-source.yml`,
  https://stalw.art/open-source
- [G-issue-484] https://github.com/stalwartlabs/stalwart/issues/484 (missing IMAP extensions)
- [G-issue-3299] https://github.com/stalwartlabs/stalwart/issues/3299 (IDLE keepalive, auto-closed)
- [G-issues] GitHub issue search `imap in:title`, including #3332, #3247 and #3241
- [H-tags] Docker Hub API, `https://hub.docker.com/v2/repositories/stalwartlabs/stalwart/tags`,
  read 2026-11-11
- [H-cli] Docker Hub API, `https://hub.docker.com/v2/repositories/stalwartlabs/cli/tags`, read
  2026-11-11

flock and klicker.ing

- [K-compose] `/home/dyle/Source/headcode.space/klicker.ing/main/docker-compose.test.yml`
  (throwaway stack, internal network, no published ports, digest-pinned images)
