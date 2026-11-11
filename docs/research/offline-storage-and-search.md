# Offline storage, sync and search: how Thunderbird and KMail do it, and flock's options

Research for [#5](https://github.com/dyle71/flock/issues/5), part of the wayfinder map
[#1](https://github.com/dyle71/flock/issues/1). It feeds the storage, sync and search architecture
decision (#11). This document states facts and trade-offs; it records no decision.

Conventions:

- Thunderbird paths are relative to `releases-comm-central` (local checkout at
  `/home/dyle/Source/other/releases-comm-central`). KMail paths are relative to the local KMail
  checkout at `/home/dyle/Source/other/kmail`. Akonadi, kdepim-runtime and akonadi-search are
  cited by their public repositories on `invent.kde.org`.
- Crate versions are the `max_stable_version` reported by the crates.io API on 2026-11-11.
- Statements marked **(estimate)** are back-of-the-envelope numbers derived from the cited inputs,
  not measurements.

## How Thunderbird does it

Thunderbird keeps three separate persistent structures for mail: a **message store** per folder
(raw messages), a **summary database** per folder (headers, flags, threads, offline-operation
queue) and a **global search database** (Gloda). A fourth, **Panorama**, is the in-progress
replacement for the per-folder summaries. Source: comm-central at HEAD `2669a12b`.

### Message stores: pluggable mbox or maildir

- Stores sit behind `nsIMsgPluggableStore` (`mailnews/base/public/nsIMsgPluggableStore.idl`):
  "Each incoming server can have a different message store." A message is located by an opaque
  `storeToken`, "Unique to the folder" and "Constant. Mostly. They may change during a
  compaction": the byte offset for mbox, the file name for maildir.
- mbox is the default: `pref("mail.serverDefaultStoreContractID",
  "@mozilla.org/msgstore/berkeleystore;1")` (`mailnews/mailnews.js:644`).
- mbox (`mailnews/local/src/nsMsgBrkMBoxStore.cpp`) writes byte-exact with reversible mboxrd
  "From " escaping (`mailnews/base/src/MboxMsgOutputStream.h`). The reader has to guess:
  `MboxMsgInputStream.cpp` aims to "Assume well formed mboxrd format, but try to handle other
  variants". **Deleting does not free space**: `DeleteStoreMessages` only patches the Expunged bit
  in `X-Mozilla-Status`; space returns through compaction, triggered by
  `mail.purge_threshold_mb = 500` (`mailnews.js:341`).
- Compaction (`mailnews/base/src/FolderCompactor.cpp`, `mailnews/local/src/MboxCompactor.h`)
  rewrites the mbox into a temp location and renames it into place. Documented failure modes:
  malformed old mboxes ("there are old mboxes in the wild which don't have 'From ' separators"),
  Windows file-locking races (bugs 1959858, 1965686), and it is postponed while offline IMAP
  operations are pending.
- maildir (`nsMsgMaildirStore.cpp`) uses only `tmp/` and `cur/`, one file per message, deletes by
  removing the file and needs no compaction (`supportsCompaction = false`). Flags are not encoded
  in the file name; they live in the summary DB.

### Summary databases: `.msf` files in Mork

- One Mork-format `.msf` per folder (`mailnews/db/msgdb`, `mailnews/db/mork`). It holds the
  envelope columns (subject, sender, recipients, date, size, flags, message-id, references,
  thread ids), arbitrary string properties (keywords, `storeToken`, `gloda-id`, `gloda-dirty`),
  per-thread rows, a folder-info row (counts, `expungedBytes`, `UIDValidity`, `highWaterKey`)
  and, for IMAP, the offline-operations table.
- The summary is a cache that is **thrown away and rebuilt** whenever it looks wrong:
  `nsMsgDatabase::CheckForErrors` deletes the `.msf` ("blow away the db if it's corrupt") on a
  missing folder-info row, an invalid flag, a version mismatch or `forceReparse`; for mbox,
  validity is a comparison of stored folder size and mtime with the file
  (`nsMsgBrkMBoxStore.cpp:313-360`). Rebuild reparses the store and recovers flags from the
  `X-Mozilla-*` headers written into each message.
- Pain points noted in code comments: "mork seems to reuse an old table when we create one",
  "mork isn't really doing NS errors yet", running out of 32-bit keys forces a reparse.

### Panorama: one SQLite database replacing the `.msf` files

- `mailnews/db/panorama/docs/index.md`: "Panorama is the working name of Thunderbird's new global
  message database. All folders and messages will be stored in a single database, instead of one
  database per folder as currently happens." Status: "It is not live and there are currently no
  user-facing parts"; compiled on nightly only, `mail.panorama.enabled` defaults to `false`
  (`mail/app/StaticPrefList.yaml:64`).
- Schema (`panorama.sqlite`, WAL mode, `DatabaseCore.cpp`): `folders`, `folder_properties`,
  `messages` (folder, thread, message-id, date, sender, recipients, subject, flags, tags, ...),
  `message_properties`, virtual-folder tables. No full-text table yet; the plan lists "Replace
  gloda by integrating full-text searching into Panorama."
- Lessons the Panorama docs record:
  - "nsMsgKeys are only 32bit ... 32 bits probably aren't enough for a global database ... we
    should just bite the bullet and go to 64 bits."
  - "IMAP/News won't work": the old design uses the server UID as the local primary key.
    Thunderbird's own ADR (`docs/adr/records/0002-imap-should-not-use-uid-as-message-key.md`,
    Approved) decides that "the database should have responsibility for managing it's own
    primary keys" and the UID becomes a separate field, because UIDs "are unique within a folder
    but they are not globally unique", because "We cannot properly model a message which appears
    in more than one folder" (Gmail `X-GM-MSGID`, RFC 8474 object IDs), and because temporary
    fake keys are needed while a message awaits its UID.
  - Code relies on detached header objects: "Mork supports editing database rows which are not
    attached to a table in the DB. Sqlite does not."

### Gloda: the global search index

- Enabled by default in Thunderbird (`mail/app/profile/all-thunderbird.js:739`). A separate
  SQLite database `global-messages-db.sqlite`, schema version 30
  (`mailnews/db/gloda/modules/GlodaDatastore.sys.mjs`); a corrupt file is deleted and rebuilt;
  schema versions below 26 are wiped rather than migrated.
- Full text uses SQLite **FTS3** with Thunderbird's own tokenizer `mozporter`
  (`USING fts3(tokenize mozporter, ...)`, `GlodaDatastore.sys.mjs:1395`;
  `mailnews/extensions/fts3/fts3_porter.c`: Porter stemming plus "bi-gram indexing" for CJK,
  case and accent folding). Indexed: body, subject, attachment names, author, recipients.
- Indexing is asynchronous and throttled (`IndexMsg.sys.mjs`, `GlodaIndexer.sys.mjs`): event
  driven, but "if we get too many events we start dropping them on the floor and just flag that an
  indexing sweep is required"; time-sliced (32 ms work slices, longer when idle, backs off under
  system load). Per-message index state is kept back in the `.msf` (`gloda-id`, `gloda-dirty`),
  i.e. two databases must agree.
- Coverage limit: "We index the headers of all IMAP messages. We index the bodies of all IMAP
  messages that are offline." Bodies are capped at 20 KB per part (`MAX_SANE_BODY_PART_SIZE`).
- Gloda serves global search, faceting and autocomplete only. Quick filter, Search Messages and
  saved searches run against the `.msf` summaries and read bodies from the local store
  (`mailnews/search/src/nsMsgLocalSearch.cpp`), or send IMAP `SEARCH` online. There are therefore
  two search paths over different data (FTS index vs. summary scan).

### Offline IMAP and sync

- Offline bodies are on by default (`mail.server.default.offline_download` true,
  `autosync_max_age_days -1`, `mailnews.js:618-621`). Autosync (`nsAutoSyncManager.h`) runs at idle
  time, folder priority "INBOX > DRAFTS > SUBFOLDERS > TRASH", "most recent and smallest first",
  2 MB download groups, retries per group.
- **CONDSTORE is off by default**: `pref("mail.server.default.use_condstore", false)`
  (`mailnews.js:585`), commented as "in case client or server has bugs in condstore
  implementation". When on, sync compares HIGHESTMODSEQ and issues `UID FETCH 1:* (CHANGEDSINCE
  n)` (`nsImapProtocol.cpp`). Expunges are still detected heuristically by message count.
  **QRESYNC/VANISHED are not implemented** (no occurrence in `mailnews/imap/src`).
- UIDVALIDITY change: `HandleUidInvalidation` throws away the folder's database and downloaded
  messages and starts over (`nsImapMailFolder.cpp`).
- Offline changes are queued as rows in the `.msf` offline-ops table
  (`nsIMsgOfflineImapOperation.idl`: operations "already performed locally ... waiting for a
  connection"): flag and keyword changes, moves/copies, appends, deletes, one row per message key,
  replayed by `nsImapOfflineSync`. Offline moves need fake keys until the server assigns UIDs.

### Lessons from Thunderbird

1. Per-folder summary files that are silently rebuilt are fragile and make global views (unified
   inbox, cross-account search) a second system. Thunderbird is migrating to one global SQLite
   database after years with Mork.
2. Local primary keys must be the database's own (64-bit), with server UIDs as attributes, and the
   model must allow one message in several folders.
3. A separate search DB that tracks its state in another DB (Gloda + `.msf`) and drops events
   under load needs sweeps to catch up; search coverage then depends on what was downloaded.
4. mbox needs compaction and escaping heuristics; maildir avoids both but moves flags into the DB.
5. CONDSTORE support existed but was disabled by default for fear of server bugs; QRESYNC never
   landed. Incremental sync quality is bounded by server quirks and needs a fallback full-flag
   resync.

## How KMail does it

KMail stores nothing itself. It is a client of **Akonadi**, a per-user server process with a SQL
database, fed by **resources** (one agent process per account type) and indexed by the
**akonadi-search** agent (Xapian). KMail waits for `Akonadi::ServerManager` to reach `Running`
before it does anything (`KMKernel::akonadiStateChanged`, `src/kmkernel.cpp`). Akonadi sources
were read from `invent.kde.org` `master` on 2026-11-11.

### Akonadi: a cache, not a store

- KDE's architecture page: "Akonadi is a cache, not a storage." The server caches items in SQL;
  resources sync between the cache and the real backend
  ([Akonadi architecture](https://community.kde.org/KDE_PIM/Akonadi/Architecture)).
- Database backend is a build-time default plus a runtime config (`General/Driver` in
  `akonadiserverrc`, `src/server/storage/dbconfig.cpp`): MySQL/MariaDB, PostgreSQL or SQLite. The
  compiled default was `"MYSQL"` up to v25.12.0 and is `"SQLITE"` on `master`
  (`CMakeLists.txt`, commit "Switch default database backend to SQLITE", 2026-01-27). The KDE PIM
  2025 sprint notes record the intent to drop the other backends eventually and that the SQLite
  backend "Fails on super large queries", since fixed
  ([Sprints/PIM/2025](https://community.kde.org/Sprints/PIM/2025)). Existing installations keep
  their configured backend; migration is an explicit `akonadi-db-migrator` run. `DbConfigSqlite`
  uses `journal_mode=wal`, `synchronous=1`.
- Schema (`src/server/storage/akonadidb.xml`): `PimItemTable` (collection, mime type, revision,
  `remoteId`, `remoteRevision`, `gid`, `atime`, `dirty`, size), `PartTable` (one row per payload
  part: envelope, header, body..., with `storage` = Internal/External/Foreign),
  `CollectionTable` with an inheritable **cache policy** (`cachePolicyCheckInterval`,
  `cachePolicyCacheTimeout`, `cachePolicySyncOnDemand`, `cachePolicyLocalParts`), flags, tags.
- **Hybrid blob storage**: `mSizeThreshold = 4096` (configurable `General/SizeThreshold`,
  `dbconfig.cpp`); parts larger than that are written as files under
  `~/.local/share/akonadi/file_db_data`, smaller ones inline in the DB
  (`src/server/storage/partstreamer.cpp`, `src/private/externalpartstorage.cpp`).
- The cache policy drives eviction and fetch: `CacheCleaner` drops payload parts older than the
  cache timeout unless they are in `localParts` or the item is `dirty`
  (`src/server/cachecleaner.cpp`); `IntervalCheck` triggers resource syncs (minimum 5 minutes);
  `ItemRetriever` asks the owning resource for missing parts on demand
  (`src/server/storage/itemretriever.cpp`).
- Offline changes: `Akonadi::ChangeRecorder` records change notifications "while an agent is not
  online and replaying the notifications when the agent is online again"
  (`src/core/changerecorder.h`); `ResourceBase` replays them through its scheduler. Items with
  unsynced local changes are marked `dirty`. KMail's offline switch sets the IMAP/POP3 resources
  and the mail dispatcher offline (`KMKernel::setAccountStatus`).

### Maildir resource: double bookkeeping

- For local mail the **maildir on disk is the source of truth** and Akonadi caches it:
  `MaildirResource` watches the directory with `KDirWatch`, sets the root cache policy to
  `localParts = {Envelope}`, `cacheTimeout = 1` minute, `syncOnDemand = true`, and reads bodies
  from disk on request (kdepim-runtime `resources/maildir/maildirresource.cpp`). Envelopes and
  metadata therefore exist twice, and the two copies are reconciled by remote IDs (file names).

### IMAP resource: full offline copy by default, QRESYNC when possible

- `DisconnectedModeEnabled` ("Defines if all the IMAP data should be cached locally all the
  time") defaults to `true` (`resources/imap/imapresource.kcfg`); then the cache policy keeps
  full bodies forever (`cacheTimeout = -1`), otherwise only envelope and header for 60 minutes
  (`retrievecollectionstask.cpp`).
- `RetrieveItemsTask` (`resources/imap/retrieveitemstask.cpp`) uses a QRESYNC `SELECT` when the
  server supports it and UIDVALIDITY, UIDNEXT and HIGHESTMODSEQ are cached ("it is useless to do a
  QRESYNC SELECT if we don't have current UIDValidity, UIDNEXT and HIGHESTMODSEQ cached already"),
  handles `VANISHED` and modified responses, and stores HIGHESTMODSEQ under CONDSTORE. On a
  UIDVALIDITY change: "our local cache is worthless and has to be refetched completely". Without
  CONDSTORE it takes a shortcut that it documents itself: "WARNING: This sacrifices consistency as
  we will not detect flag changes until a new message enters the mailbox."

### Search: akonadi-search on Xapian

- akonadi-search is a fork of the KDE 4 Baloo code on Xapian, unrelated to the KF5 Baloo file
  indexer ([kdepim-users,
  2016-10](https://mail.kde.org/pipermail/kdepim-users/2016-October/011235.html)). The index lives
  outside the Akonadi DB in `~/.local/share/akonadi/search_db/` (separate databases for email,
  contacts, ..., `lib/indexeditems.cpp`).
- An indexing agent receives item notifications and indexes addresses, subject, all raw headers,
  the plain-text body (HTML converted only when no text part exists), status flags, size and date
  (`agent/emailindexer.cpp`). Attachments are not indexed ("FIXME: Handle attachments?").
  Bumping `INDEXING_AGENT_VERSION` deletes the index and reindexes everything (`agent/agent.cpp`).
- KMail searches only the local index (`setRemoteSearchEnabled(false)` in the search dialog) and
  warns about unindexed folders. Its index checker skips IMAP collections whose cache policy does
  not keep `RFC822` bodies (`src/search/checkindexingmanager.cpp:142`), so body search covers only
  what is cached locally.
- A 2017 proposal by the Akonadi maintainer, "Make Indexing Great Again", suggested indexing inside
  the server before items become visible and dropping the separate agent
  ([T7014](https://phabricator.kde.org/T7014)); the current code still uses the agent.

### Documented pain points

- KMail's own troubleshooting guide (`doc/kmail2/troubleshooting.docbook`) covers phantom
  duplicates from `LRCONFLICT` (bug 283682, since 2011, workaround: manual SQL
  `delete from pimitemtable where remoteid is NULL`), items stuck `dirty` (bug 436550), a MySQL
  `binlog_format` fix for "Unable to fetch item", and clean-start steps that kill `mysqld`. It
  still assumes MySQL and describes `file_db_data` as disposable, though the source shows it holds
  every payload part above 4 KiB, including unsynced ones.
- Maintenance tools exist because the cache drifts: `akonadictl fsck` (`StorageJanitor::check`:
  orphaned collections/items/parts/flags, duplicate remote IDs, overlapping external parts, dirty
  objects, search-index consistency) and `akonadictl vacuum`, which needs "a lot of temporary disk
  space" (`src/server/storagejanitor.cpp`). User reports say both often do not fix duplicates
  ([bug 401198](https://bugs.kde.org/show_bug.cgi?id=401198)).
- The 2025 sprint notes Flatpak-isolated Akonadi instances cannot share data.

### Lessons from KMail

1. A generic PIM cache between the client and its data adds a second source of truth for local
   mail (maildir + DB) and a third structure for search (Xapian), each needing reconciliation and
   repair tools.
2. A client/server database (MySQL) as the default for a desktop app caused years of startup and
   corruption support issues; Akonadi has now switched its default to SQLite.
3. A size threshold (4 KiB) splitting payloads between DB and files is workable, but users and
   even KMail's docs misjudge the file directory as disposable.
4. Search coverage again equals local cache coverage; QRESYNC/CONDSTORE are used when present, and
   their absence is papered over with a documented consistency trade-off.

## flock's options

flock's constraints, taken from the ticket: one Rust binary with two runtime modes (local
single-user as an Electron sidecar, which is the default, and a self-hosted multi-user server);
every mailbox synced completely into flock's own store; fully offline; full-text search across all
accounts; unified views. That splits the storage problem into four layers. They can be chosen
largely independently, as Stalwart (a Rust mail server) does: it configures a separate *data
store*, *blob store*, *search store* and *in-memory store*, and each defaults to living inside the
data store
([Stalwart storage overview](https://stalw.art/docs/storage/overview),
[backends](https://stalw.art/docs/storage/backends/)).

1. **Metadata store**: accounts, folders, messages (envelope, flags, keywords, thread links), sync
   state (UIDVALIDITY, UIDNEXT, HIGHESTMODSEQ, POP3 UIDLs), the offline change queue.
2. **Blob store**: raw RFC 5322 messages (and possibly extracted attachments).
3. **Search index**: full-text over headers, decoded body text and possibly attachment text.
4. **Sync engine**: protocol logic writing into 1–3. It is not a storage choice, but its needs (a
   durable outbound queue, per-folder sync cursors, cheap flag updates) constrain the store.

### Protocol facts that shape the store

- IMAP UIDs are only valid with their UIDVALIDITY: "If unique identifiers from an earlier session
  fail to persist in this session, the unique identifier validity value MUST be greater than the
  one used in the earlier session"
  ([RFC 9051 §2.3.1.1](https://www.rfc-editor.org/rfc/rfc9051#section-2.3.1.1)). A store keyed
  only by `(folder, uid)` must be able to drop and re-fetch a whole folder; keying blobs by content
  (or by Message-ID plus a hash) lets a re-fetch skip bodies already present.
- CONDSTORE gives every message a mod-sequence and the mailbox a HIGHESTMODSEQ;
  `FETCH ... (CHANGEDSINCE n)` returns only changed flags. QRESYNC additionally "MUST increment the
  per-mailbox mod-sequence when one or more messages are expunged" and reports them via
  `VANISHED` ([RFC 7162 §3.1, §3.2](https://www.rfc-editor.org/rfc/rfc7162)). Without them, flag
  and expunge detection means re-fetching all UIDs and flags per folder. The store needs a
  per-folder cursor `(uidvalidity, uidnext, highestmodseq)` and a per-message `modseq`.
- POP3 has no folders and no flags; the only stable key is the UIDL, "an arbitrary
  server-determined string, consisting of one to 70 characters ... which uniquely identifies a
  message within a maildrop and which persists across sessions"
  ([RFC 1939, UIDL](https://www.rfc-editor.org/rfc/rfc1939)). For POP3 accounts flock's store is
  the only copy of state such as read flags.
- JMAP (later) addresses bodies by `blobId`, and "the data represented by a blobId is immutable"
  ([RFC 8620 §6](https://www.rfc-editor.org/rfc/rfc8620#section-6)). JMAP sync is state-string
  based (`/changes`), so a store with a monotonic per-account change counter maps directly onto it
  and onto flock's own client-to-server sync towards the SPA.

### Sizing (estimate)

The ticket's reference load is 10 accounts with 500k messages in total. Average message size
varies by mailbox type (attachment-heavy business mail vs. mailing lists); the table is
parametric, not measured.

| Quantity | 25 KB avg | 75 KB avg | Basis |
| --- | --- | --- | --- |
| Raw RFC 5322 blobs | ~12.5 GB | ~37.5 GB | 500k × avg size |
| Metadata rows (envelope, flags, threading; ~1–2 KB/msg) | ~0.5–1 GB | ~0.5–1 GB | estimate |
| FTS index, `detail=full`, over decoded text only (~5 KB/msg → ~2.5 GB of text) | ~1.1 GB | ~1.1 GB | ratio 743/1636 MiB from SQLite's e-mail test |
| same, `detail=column` / `detail=none` | ~0.5 GB / ~0.2 GB | same | 340/1636, 134/1636 MiB |

The ratios come from the FTS5 documentation: "In one test that indexed a large set of emails (1636
MiB on disk), the FTS index was 743 MiB on disk with detail=full, 340 MiB with detail=column and 134
MiB with detail=none" ([SQLite FTS5 §4.6](https://www.sqlite.org/fts5.html)). For scale, the
PostgreSQL docs cite their own mailing-list archive: 461,020 messages, 910,989 unique words,
57,491,343 lexemes ([PostgreSQL text search
limitations](https://www.postgresql.org/docs/current/textsearch-limitations.html)), i.e. the
reference load is about one large public mailing-list archive. The conclusion that survives any
reasonable average: **blobs dominate (tens of GB), metadata and index are each low single-digit
GB.** That is well inside SQLite's limits (default max BLOB 1,000,000,000 bytes; ~17.5 TB max DB at
4 KiB pages, [SQLite limits](https://www.sqlite.org/limits.html)) and PostgreSQL's (1 GB per TOASTed
value, [TOAST](https://www.postgresql.org/docs/current/storage-toast.html)). On a multi-user server
the same numbers multiply by user count.

### Layer 1: metadata database

| | SQLite only | PostgreSQL only | SQLite locally + PostgreSQL on the server |
| --- | --- | --- | --- |
| Local sidecar mode | Embedded, zero-config, one file; "SQLite competes with fopen()" ([whentouse](https://www.sqlite.org/whentouse.html)) | Needs a bundled or downloaded server process, e.g. `postgresql_embedded` 0.22.0 ("PostgreSQL can be bundled with your application, or downloaded"), or `pglite-oxide` 0.5.1; extra process lifecycle, port/socket, upgrades of the data directory across major versions | SQLite |
| Multi-user server | Works for "low writer concurrency"; "only one writer at a time per database file" ([whentouse](https://www.sqlite.org/whentouse.html)). One DB file per user (sharding, which the same page suggests: "using separate database files for different subdomains") removes cross-user write contention and makes per-user export/delete a file operation; no network filesystem ("WAL does not work over a network filesystem", [WAL](https://www.sqlite.org/wal.html)); one host only | Native fit: concurrent writers, row-level security for isolation (policies; "Superusers and roles with the BYPASSRLS attribute always bypass", table owners need `FORCE ROW LEVEL SECURITY`, [RLS](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)), standard backup/replication tooling, horizontal app servers | PostgreSQL |
| Code cost | One dialect | One dialect | Two SQL dialects: two migration sets, two FTS implementations, two test matrices; `sqlx::query!` is checked against one database at compile time; sqlx `Any` driver exists for runtime switching but loses the typed macros |
| Backup | Copy one file with the online backup API or `VACUUM INTO` ("resulting backup database is minimal in size", [VACUUM](https://www.sqlite.org/lang_vacuum.html)); per-user files back up independently | `pg_dump`/base backups + WAL archiving; operators already know it | Both |
| Prior art | Thunderbird's Gloda and Panorama, Akonadi's default on `master`, Stalwart single-node | Akonadi (supported backend), Stalwart cluster | Stalwart supports all of these behind one store trait |

### Layer 2: message blobs

| | In the metadata DB | Maildir (one file per message, per folder) | Content-addressed files (hash → file) | Object store (S3-compatible) |
| --- | --- | --- | --- | --- |
| Atomicity with metadata | Same transaction | Two-phase (write file, then row); crash leaves orphans to sweep | Same as maildir; idempotent writes make orphans harmless and sweepable | Same as files, plus network failure modes |
| Read performance | SQLite: BLOBs "smaller than 100KB" read faster from the DB, larger ones from files ([intern-v-extern-blob](https://www.sqlite.org/intern-v-extern-blob.html)) | One `open()` per message | Same | Network round-trip; needs local cache in server mode |
| Dedup | None by default | None; Gmail labels / IMAP COPY produce copies | Natural: same bytes → same key (across folders and accounts) | Natural if keyed by hash |
| DB size / backup | Large DB file (tens of GB); SQLite backups copy everything; PostgreSQL TOAST compresses with `pglz` by default ("The default is pglz", [config](https://www.postgresql.org/docs/current/runtime-config-client.html)) | Small DB; blobs backed up by rsync/restic; file count = message count (500k inodes) | Same; immutable files make incremental backup trivial | Delegated to the provider |
| Interop | None | Readable by other tools (mutt, notmuch, Dovecot); maildir's selling point is "no locks" and "reliable even over NFS" ([maildir](https://cr.yp.to/proto/maildir.html)) | Opaque | Opaque |
| Multi-user | Per-user DB or RLS | Per-user directory | Per-user directory (dedup only within a user unless deliberately shared) | Per-user prefix/bucket |
| Prior art | Akonadi for parts under its size threshold; Stalwart `Default` blob store | Thunderbird maildir store (opt-in); KMail maildir resource | Akonadi's external payload files are per-part files (not hashed) | Stalwart S3 blob store |

A hybrid is common in the prior art: small payloads inline, large ones as files (Akonadi's
external-part threshold), or headers and decoded text in the DB and the raw message as a file.

### Layer 3: full-text search

| | SQLite FTS5 | PostgreSQL FTS (`tsvector` + GIN) | tantivy |
| --- | --- | --- | --- |
| Ranking | `bm25()` with per-column weights ("k1 and b ... hard-coded at 1.2 and 0.75") | `ts_rank`/`ts_rank_cd`: "do not use any global information"; "Ranking can be expensive since it requires consulting the tsvector of each matching document" ([controls](https://www.postgresql.org/docs/current/textsearch-controls.html)) | BM25 "the same as Lucene" ([README](https://github.com/quickwit-oss/tantivy)) |
| Consistency with metadata | Same transaction (external-content table + triggers, or contentless-delete tables since 3.43.0) | Same transaction | Separate; documents searchable only "after a commit is called on an IndexWriter" and readers reload; must be rebuilt/replayed from the DB after a crash |
| Languages / CJK | `unicode61` (default, no stemming, diacritics removed), `porter` (English only), `trigram` (substring, any script; queries under 3 chars match nothing); custom tokenizers via the C API | Per-language dictionaries/stemmers (`to_tsvector('german', ...)`); limits: lexeme < 2 KB, `tsvector` < 1 MB, positions ≤ 16,383, ≤ 256 positions per lexeme ([limitations](https://www.postgresql.org/docs/current/textsearch-limitations.html)) | Stemming for 17 Latin-script languages; third-party Chinese (`tantivy-jieba` 0.21.0), Japanese/Korean (`lindera` 6.2.0) |
| Features for mail | `highlight()`, `snippet()` (≤ 64 tokens), prefix indexes, column filters, NEAR | Phrase/proximity (`<->`), `ts_headline` | Phrase, fuzzy, facets, fast fields (date, size, flags filters), snippets, multithreaded indexing, mmap, "Tiny startup time (<10ms)" |
| Size control | `detail=full/column/none` (45% / 21% / 8% of corpus in SQLite's e-mail test) | GIN index size depends on vocabulary; no published e-mail ratio found | No e-mail ratio found in first-party docs |
| Maintenance | `automerge`, `crisismerge`, `optimize`, `merge`, `rebuild`, `integrity-check` | Autovacuum, `REINDEX` | `LogMergePolicy with deletes`; data "immutable. To edit a document, the document needs to be deleted and reindexed" |
| Concurrency | Single DB writer (shared with metadata writes) | Concurrent | "Only one process should be able to write tantivy's index at a time" (writer lock file, [directory_lock.rs](https://github.com/quickwit-oss/tantivy/blob/main/src/directory/directory_lock.rs)) |
| Works in both modes | Local yes; server only with SQLite metadata | Server yes; local only with embedded PostgreSQL | Yes, identically, independent of the metadata DB: one index per user directory |
| Prior art | Thunderbird Gloda (SQLite FTS3 + custom tokenizer) | Stalwart (`Default` search store on PostgreSQL) | ParadeDB `pg_search` (PostgreSQL extension built on tantivy, AGPL-3.0, [README](https://github.com/paradedb/paradedb)); Bichon e-mail archiver (listed in tantivy's README); Akonadi uses Xapian, a comparable separate-index design |

Flag/folder changes are frequent in mail (read, flagged, moved). In FTS5 and PostgreSQL those
filters can be joined from the metadata tables at query time. In tantivy, filters stored as fast
fields go stale on every flag change unless the document is deleted and re-added; keeping only
immutable fields (account, date, from/to, subject, body) in tantivy and joining mutable state from
the DB avoids that churn at the cost of a two-step query.

### Rust crate support

| Crate | Version (2026-11-11) | Backends | Model | Notes |
| --- | --- | --- | --- | --- |
| `sqlx` | 0.9.0 | PostgreSQL, MySQL, MariaDB, SQLite | async, SQL strings, compile-time checked `query!` macros, `migrate!` | SQLite is bundled C (`libsqlite3-sys`); "Each SQLite connection has a dedicated thread" (`sqlx-sqlite/src/connection/worker.rs`); `Any` driver picks the backend at runtime |
| `rusqlite` | 0.42.0 | SQLite | sync | Features `bundled`, `backup` (online backup API), `blob` (incremental BLOB I/O as `Read`/`Write`/`Seek`), `hooks`, `session`, `bundled-sqlcipher` (encryption at rest); async via `tokio-rusqlite` 0.8.0 or `deadpool-sqlite` 0.14.0; the C API for custom FTS5 tokenizers is reachable through `libsqlite3-sys` |
| `sea-orm` | 2.0.4 | via sqlx: PostgreSQL, MySQL, SQLite | async ORM, entity codegen, migrations | `sea-orm-sync` for no async runtime; the most direct path to one entity model on two backends |
| `diesel` | 2.3.14 | PostgreSQL, MySQL, SQLite | sync, typed query builder | `diesel-async` 0.9.2: native async PostgreSQL/MySQL, SQLite only through `SyncConnectionWrapper` |
| `tantivy` | 0.27.0 | own index files | sync API, own threads | MIT; pre-1.0: some releases changed the index format and required a reindex ([CHANGELOG](https://github.com/quickwit-oss/tantivy/blob/main/CHANGELOG.md): 0.11, 0.13), later ones keep backward compatibility (0.24 reads 0.21/0.22) |
| `object_store` | 0.14.2 | S3, GCS, Azure, local FS | async | Uniform blob API over local files and object stores |

All listed crates are MIT or MIT/Apache-2.0, SQLite is public domain and PostgreSQL uses the
PostgreSQL License, all compatible with GPL-3.0; ParadeDB's AGPL-3.0 would matter only if flock
shipped it.

## Implications for flock

Facts and trade-offs for the decision ticket (#11). Nothing here is decided.

### What both reference clients converge on

- **One global relational database for metadata, in SQLite.** Thunderbird is replacing per-folder
  Mork files with a single SQLite `panorama.sqlite`; Akonadi has switched its default from MySQL to
  SQLite. Neither ships a client/server database as the desktop default any longer.
- **The database owns its keys.** Thunderbird's ADR 0002 and Panorama docs: local 64-bit keys,
  server UIDs as attributes, one message in several folders. Akonadi already separates its item id
  from `remoteId`. flock needs the same for IMAP (UID + UIDVALIDITY), POP3 (UIDL), JMAP (email id,
  blobId) and Gmail (`X-GM-MSGID`).
- **A per-folder sync cursor** of UIDVALIDITY, UIDNEXT and HIGHESTMODSEQ, a full-refetch path on
  UIDVALIDITY change, and a fallback when CONDSTORE/QRESYNC are missing or buggy. Thunderbird
  ships CONDSTORE disabled; Akonadi skips flag detection without it.
- **A durable local change queue** replayed when online (Thunderbird offline-ops table,
  Akonadi `ChangeRecorder` + `dirty`). flock's "fully offline" goal makes this a first-class table,
  not an afterthought.
- **Search coverage equals local body coverage** in both. flock syncs everything, so it avoids
  that gap by construction, at the price of downloading every body.
- **Separate search stores drift.** Gloda tracks its state in the `.msf`; akonadi-search is a
  separate Xapian index fed by notifications; both need sweeps, reindex-on-version-bump and fsck.
  An index inside the same transaction as the metadata (FTS5, PostgreSQL FTS) cannot drift; an
  external index (tantivy) must be rebuildable from the DB and track a high-water mark.

### Trade-offs per layer

**Metadata DB.**
- *SQLite in both modes* (one file per user on the server): one SQL dialect and one migration set,
  zero-config sidecar, per-user backup/export/delete as a file operation, isolation by file
  boundary instead of RLS. Cost: one host only, no network filesystem, one writer per user DB
  (sync workers and API writes for one user serialize), no standard server-side ops tooling, and
  horizontal scaling of the server mode is ruled out.
- *PostgreSQL in both modes*: one dialect, best server story (concurrency, RLS, replication,
  operator familiarity). Cost: the Electron sidecar must bundle and manage a PostgreSQL server
  (`postgresql_embedded`) or rely on an experimental embedded variant (`pglite-oxide`); larger
  install, port/socket and major-version upgrade handling on users' machines. This is the
  configuration Akonadi just moved away from in its MySQL form.
- *SQLite locally, PostgreSQL on the server*: each mode gets its natural engine. Cost: two
  dialects, two migration sets, two FTS implementations (unless search is external), every
  storage test run twice; `sqlx`'s compile-time checked macros bind to one engine, `sea-orm` hides
  the dialects behind entities, `sqlx::Any` gives up typed macros. Stalwart demonstrates the
  pattern at scale behind a store trait.

**Blobs.**
- *In the DB*: one transaction, one backup unit, simplest consistency. Cost: a multi-GB to
  tens-of-GB database file per user (estimate above), slower reads for large messages per SQLite's
  own measurements, `VACUUM` needing "as much as twice the size of the original database file" in
  free space ([VACUUM](https://www.sqlite.org/lang_vacuum.html)), every DB backup copying all mail.
- *Files (maildir or content-addressed)*: small DB, incremental backup with standard file tools,
  large reads streamed from disk. Cost: two-phase writes with orphan sweeping, about 500k files
  per user at the reference load. Maildir adds interoperability with other tools; content
  addressing adds dedup across folders/accounts (Gmail labels, copies) and immutable files, which
  match JMAP's immutable `blobId`.
- *Hybrid by size* (Akonadi: 4 KiB threshold; SQLite: ~100 KB crossover for reads): small
  messages in the DB, large ones as files. Cost: two code paths.
- *Object store* (`object_store` crate): relevant only to the server mode; local mode would still
  need files.

**Full-text search.**
- *SQLite FTS5*: transactional with metadata, BM25, snippets, `detail` option trading phrase
  search for index size (45/21/8% of corpus); but stemming is English-only out of the box, CJK
  needs `trigram` or a custom C tokenizer (Thunderbird wrote `mozporter` for exactly this), and
  index writes share the single writer with sync.
- *PostgreSQL FTS*: transactional, per-language stemming dictionaries; ranking ignores global
  statistics and must read every matching `tsvector`, and the 1 MB `tsvector` limit needs body
  truncation for very large text. BM25 in PostgreSQL means ParadeDB (AGPL-3.0, itself tantivy).
- *tantivy*: identical in both runtime modes and independent of the metadata DB choice (which
  decouples the hardest SQLite-vs-PostgreSQL difference), BM25, CJK tokenizers available, fast
  fields and facets for unified views. Cost: a second store that must be kept in step (the Gloda and
  akonadi-search lesson), one writer process per index, mutable state (flags, folder) best kept out
  of the index, and reindexing when a 0.x release changes the index format (the changelog records
  forced reindexes at 0.11 and 0.13; 0.24 reads 0.21/0.22 indices).

### Questions for the decision ticket

- Does the server mode need more than one host (horizontal scaling), or is one host per
  installation the target? This alone decides whether SQLite-per-user is viable server-side.
- Is interoperability of the on-disk mail with other tools (maildir) a requirement, or is an
  opaque store acceptable?
- Which languages must search stem or segment in v1 (German, English, CJK)?
- Must the local data be encrypted at rest (SQLCipher via `rusqlite` `bundled-sqlcipher`), and
  does that extend to blob files and the search index?
- What backup and export unit does a user or operator expect: one file, one directory, or a
  database dump?

## Sources

Local source trees (read 2026-11-11):

- Thunderbird comm-central, `/home/dyle/Source/other/releases-comm-central` (HEAD `2669a12b`):
  `mailnews/base/public/nsIMsgPluggableStore.idl`, `mailnews/local/src/nsMsgBrkMBoxStore.cpp`,
  `mailnews/local/src/nsMsgMaildirStore.cpp`, `mailnews/base/src/MboxMsg{Input,Output}Stream.*`,
  `mailnews/base/src/FolderCompactor.cpp`, `mailnews/db/msgdb/src/nsMsgDatabase.cpp`,
  `mailnews/db/panorama/docs/index.md`, `mailnews/db/panorama/src/DatabaseCore.cpp`,
  `docs/adr/records/0002-imap-should-not-use-uid-as-message-key.md`,
  `mailnews/db/gloda/modules/{GlodaDatastore,GlodaIndexer,IndexMsg}.sys.mjs`,
  `mailnews/extensions/fts3/fts3_porter.c`,
  `mailnews/imap/src/{nsImapProtocol,nsImapMailFolder,nsAutoSyncManager,nsImapOfflineSync}.*`,
  `mailnews/search/src/nsMsgLocalSearch.cpp`, `mailnews/mailnews.js`,
  `mail/app/profile/all-thunderbird.js`, `mail/app/StaticPrefList.yaml`.
- KMail, `/home/dyle/Source/other/kmail` (HEAD `98c9118`): `src/kmkernel.cpp`,
  `src/kmmainwidget.cpp`, `src/search/checkindexingmanager.cpp`,
  `doc/kmail2/troubleshooting.docbook`.

KDE (fetched 2026-11-11, `master` unless noted):

- [pim/akonadi](https://invent.kde.org/pim/akonadi): `CMakeLists.txt` (also tag `v25.12.0`),
  `src/server/storage/{akonadidb.xml,dbconfig.cpp,dbconfigsqlite.cpp,partstreamer.cpp,itemretriever.cpp}`,
  `src/private/externalpartstorage.cpp`,
  `src/server/{cachecleaner,intervalcheck,storagejanitor}.cpp`, `src/core/changerecorder.h`,
  `src/agentbase/resourcebase.cpp`.
- [pim/kdepim-runtime](https://invent.kde.org/pim/kdepim-runtime):
  `resources/imap/{retrieveitemstask.cpp,retrievecollectionstask.cpp,imapresource.kcfg}`,
  `resources/maildir/maildirresource.cpp`.
- [pim/akonadi-search](https://invent.kde.org/pim/akonadi-search): `agent/{emailindexer,agent}.cpp`,
  `lib/indexeditems.cpp`.
- [Akonadi architecture](https://community.kde.org/KDE_PIM/Akonadi/Architecture),
  [KDE PIM sprint 2025](https://community.kde.org/Sprints/PIM/2025),
  [T7014](https://phabricator.kde.org/T7014),
  [kdepim-users 2016-10](https://mail.kde.org/pipermail/kdepim-users/2016-October/011235.html),
  [bug 401198](https://bugs.kde.org/show_bug.cgi?id=401198) (user report).

Specifications and technology documentation:

- RFC 9051 (IMAP4rev2), RFC 7162 (CONDSTORE/QRESYNC), RFC 1939 (POP3), RFC 8620 (JMAP core),
  [maildir](https://cr.yp.to/proto/maildir.html).
- SQLite: [FTS5](https://www.sqlite.org/fts5.html),
  [Appropriate uses](https://www.sqlite.org/whentouse.html),
  [Internal vs. external BLOBs](https://www.sqlite.org/intern-v-extern-blob.html),
  [Limits](https://www.sqlite.org/limits.html), [WAL](https://www.sqlite.org/wal.html),
  [VACUUM](https://www.sqlite.org/lang_vacuum.html).
- PostgreSQL (current docs):
  [text search limitations](https://www.postgresql.org/docs/current/textsearch-limitations.html),
  [ranking](https://www.postgresql.org/docs/current/textsearch-controls.html),
  [TOAST](https://www.postgresql.org/docs/current/storage-toast.html),
  [client config](https://www.postgresql.org/docs/current/runtime-config-client.html),
  [row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html).
- tantivy [README](https://github.com/quickwit-oss/tantivy),
  [ARCHITECTURE.md](https://github.com/quickwit-oss/tantivy/blob/main/ARCHITECTURE.md),
  [CHANGELOG](https://github.com/quickwit-oss/tantivy/blob/main/CHANGELOG.md),
  `src/directory/directory_lock.rs`.
- [sqlx README](https://github.com/launchbadge/sqlx) and
  `sqlx-sqlite/src/connection/worker.rs`; [rusqlite README](https://github.com/rusqlite/rusqlite);
  [SeaORM README](https://github.com/SeaQL/sea-orm);
  [diesel_async](https://github.com/weiznich/diesel_async) `src/lib.rs`;
  [ParadeDB README](https://github.com/paradedb/paradedb); crates.io API for versions and
  licenses.
- Stalwart [storage overview](https://stalw.art/docs/storage/overview) and
  [backends](https://stalw.art/docs/storage/backends/).
