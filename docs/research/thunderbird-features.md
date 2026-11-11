# Thunderbird mail feature catalogue

Research ticket: dyle71/flock#2. Companion: `docs/research/kmail-features.md` (KMail side of the
parity matrix).

## Question

Which mail features does Thunderbird ship (accounts, folders, message list, reading, composing,
search, filters, tags, identities, address book, offline, security, settings), and where in the
Thunderbird source tree does each live? Mail only; calendar, chat, RSS and NNTP are out of scope.

## Sources

- Thunderbird source, comm-central checkout at `/home/dyle/Source/other/releases-comm-central`,
  version `162.0a1` per `mail/config/version.txt` (Nightly line, so a few items below sit behind
  Nightly-only or experimental prefs; these are marked).
- User-visible strings: `mail/locales/en-US/messenger/**/*.ftl` and
  `mail/locales/en-US/chrome/messenger/**/*.dtd`. A feature is listed only if it has a UI string
  or a pref and backing code.
- Default prefs: `mailnews/mailnews.js`, `mail/app/profile/all-thunderbird.js`,
  `mail/app/StaticPrefList.yaml`.
- Implementation: `mail/` (front end), `mailnews/` (back end), `rust/` (Exchange clients),
  `mail/extensions/openpgp/` (OpenPGP).

All paths below are relative to the checkout root. Shorthand used in the tables:

| Shorthand | Expands to                                  |
| --------- | ------------------------------------------- |
| `L10N/`   | `mail/locales/en-US/messenger/`             |
| `DTD/`    | `mail/locales/en-US/chrome/messenger/`      |
| `AM/`     | `mailnews/base/prefs/content/` (account settings pages) |
| `PREFS/`  | `mail/components/preferences/` (global settings tab)    |

Feature IDs (`TB-ACC-01` and so on) are stable handles for the parity matrix; they carry no meaning
beyond this file.

Status markers: **exp** = experimental or behind a pref that is off by default; **nightly** =
compiled or enabled on Nightly only; no marker = shipped and on by default.

## Out of scope (one line each)

- **Calendar/tasks**: `calendar/` (Lightning, CalDAV, ICS, Graph calendars); only touches mail via
  "Add to calendar" and invitation handling.
- **Chat**: `chat/` and `mail/components/im/` (XMPP, Matrix, IRC, OTR).
- **RSS/Atom feeds**: `mailnews/extensions/newsblog/` (feed accounts appear as mail folders).
- **NNTP/newsgroups**: `mailnews/news/` (news accounts, subscribe, followup; shares message list,
  filters and search with mail).

## 1. Accounts and protocols

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-ACC-01 | IMAP4rev1 account | Full IMAP client, with capability use of IDLE, CONDSTORE/HIGHESTMODSEQ, COMPRESS=DEFLATE, MOVE, UIDPLUS, SPECIAL-USE/XLIST, LIST-EXTENDED, NAMESPACE, ACL, QUOTA, ID, CLIENTID, ENABLE, UTF8=ACCEPT, LITERAL+, X-GM-EXT (Gmail labels) | `mailnews/imap/src/nsImapProtocol.cpp`, `mailnews/imap/src/nsImapCore.h` (capability flags), `mailnews/imap/src/nsImapIncomingServer.cpp`, `mailnews/imap/src/nsImapMailFolder.cpp` |
| TB-ACC-02 | POP3 account | POP3 client: leave on server, delete after N days, delete from server when deleted locally, fetch headers only, size limit, per-message fetch/delete from server | `mailnews/local/src/Pop3Client.sys.mjs`, `mailnews/local/src/Pop3IncomingServer.sys.mjs`, `L10N/accountSettings.ftl` |
| TB-ACC-03 | Global Inbox (deferred POP) | POP account delivers into another account's Inbox ("Inbox for different account") | `mail.server.default.deferred_to_account` in `mailnews/mailnews.js`, `AM/am-server.js`, `L10N/folderWidgets.ftl` ("Global Inbox") |
| TB-ACC-04 | Local Folders | Built-in "none" server holding local folders, Outbox ("Unsent Messages") and archives | `mailnews/local/src/nsNoIncomingServer.cpp`, `mailnews/local/src/nsLocalMailFolder.cpp` |
| TB-ACC-05 | Microsoft Exchange (EWS) | Native Exchange Web Services account: folder sync, message sync, read/flag/junk state, copy/move, create/delete folders, send | `mailnews/protocols/exchange/src/` (C++ folder/server), `rust/ews_xpcom/src/client/`, `mailnews/protocols/exchange/docs/architecture.md` |
| TB-ACC-06 | Microsoft Graph account | Exchange/Microsoft 365 via Graph API, same operation set as EWS (on by default via `mail.graph.enabled`) | `rust/graph_xpcom/src/client/`, `rust/ms_graph_tb/`, `mail/app/StaticPrefList.yaml` |
| TB-ACC-07 | SMTP outgoing servers | Multiple SMTP servers, per-identity choice, default server, 8BITMIME, SMTPUTF8, SIZE, DSN | `mailnews/compose/src/SmtpClient.sys.mjs`, `mailnews/compose/src/SmtpServer.sys.mjs`, `mailnews/compose/src/OutgoingServerService.sys.mjs`, `AM/am-smtp.js`, `AM/SmtpServerEdit.xhtml` |
| TB-ACC-08 | Automatic account setup | Email + password, lookup via ISPDB, autoconfig on the domain, MX lookup, Exchange AutoDiscover, port/protocol guessing | `mail/components/accountcreation/modules/FindConfig.sys.mjs`, `FetchConfig.sys.mjs`, `GuessConfig.sys.mjs`, `ExchangeAutoDiscover.sys.mjs`, `readFromXML.sys.mjs`, `mail/modules/DNS.sys.mjs` |
| TB-ACC-09 | Manual account configuration | Hand-entered incoming/outgoing host, port, security, auth method, username, with re-test | `mail/components/accountcreation/views/email.mjs`, `content/widgets/email-manual-*-form.mjs`, `modules/ConfigVerifier.sys.mjs` |
| TB-ACC-10 | Account Hub | New unified setup flow for email accounts and address books, including Exchange type choice and "sync accounts" step | `mail/components/accountcreation/content/accountHub.js`, `views/`, `L10N/accountcreation/accountHub.ftl`; toggles in Beta Features (`L10N/preferences/preferences.ftl` `account-hub-*`) |
| TB-ACC-11 | Thundermail provider setup | First-party Thundermail sign-up/sign-in path in Account Hub | `mail/components/accountcreation/modules/Thundermail.sys.mjs`, pref `mail.accounthub.thundermail.enabled` |
| TB-ACC-12 | Authentication methods | Normal/encrypted password (PLAIN, LOGIN, CRAM-MD5), Kerberos/GSSAPI, NTLM, TLS client certificate (EXTERNAL), OAuth2 (XOAUTH2), none | `mailnews/base/src/MailAuthenticator.sys.mjs`, `mailnews/base/src/nsMailAuthModule.cpp`, `kHasAuth*` in `mailnews/imap/src/nsImapCore.h` |
| TB-ACC-13 | OAuth2 | Built-in providers (Google, Microsoft, Yahoo/AOL, Fastmail, Yandex, Mail.ru, Seznam, Comcast, Thundermail), PKCE, external system browser, custom OAuth2 settings per account | `mailnews/base/src/OAuth2.sys.mjs`, `OAuth2Providers.sys.mjs`, `OAuth2Module.sys.mjs`, `OAuth2CustomDetails.sys.mjs`, `L10N/accountSettings.ftl` (custom OAuth fields) |
| TB-ACC-14 | Connection security | None / STARTTLS / SSL/TLS per server; invalid-certificate exception flow during setup | `AM/am-server.js`, `AM/certificate-check.mjs`, `mail/components/accountcreation/modules/ConfigVerifier.sys.mjs` |
| TB-ACC-15 | New-mail checking (biff) | Check at startup, every N minutes, IDLE push, include server in "Get All New Messages", check all folders | `mailnews/base/src/nsMsgBiffManager.cpp`, `mail.server.default.check_time` / `use_idle` / `check_all_folders_for_new` in `mailnews/mailnews.js`, `L10N/accountSettings.ftl` |
| TB-ACC-16 | IMAP server options | IMAP server directory, show only subscribed folders, namespaces (personal / other users / public) with server override, max cached connections, dual-use folders | `AM/am-server-advanced.js`, `L10N/accountSettings.ftl`, `mailnews/imap/src/nsImapNamespace.cpp` |
| TB-ACC-17 | IMAP delete model | Move to Trash / mark as deleted / delete immediately; expunge on folder exit; empty Trash on exit | `mail.server.default.delete_model` in `mailnews/mailnews.js`, `AM/am-server.js`, `L10N/accountSettings.ftl` |
| TB-ACC-18 | Message store format | Per-account mbox (file per folder) or maildir (file per message), plus a converter between them | `mailnews/local/src/nsMsgBrkMBoxStore.cpp`, `nsMsgMaildirStore.cpp`, `mailnews/base/src/mailstoreConverter.sys.mjs`, `AM/converterDialog.xhtml` |
| TB-ACC-19 | Account management | Add/remove accounts (with or without data), set default, reorder in folder pane, account name, account color | `AM/AccountManager.js`, `AM/removeAccount.xhtml`, `mail/modules/AccountColorUtils.sys.mjs`, `L10N/accountManager.ftl`, `L10N/removeAccount.ftl` |
| TB-ACC-20 | Account Central | Landing page for an account root with setup shortcuts | `mailnews/base/content/msgAccountCentral.xhtml`, `L10N/accountCentral.ftl` |
| TB-ACC-21 | Password storage | Saved passwords in login manager, protected by optional Primary Password | `mail/components/preferences/passwordManager.js`, `mail/modules/PrimaryPassword.sys.mjs`, `L10N/preferences/preferences.ftl` (`privacy-passwords-header`) |
| TB-ACC-22 | Export accounts to mobile | QR-code transfer of account settings (optionally passwords) to Thunderbird for Android | `mail/components/preferences/qrExport.mjs`, `mail/modules/QRExport.sys.mjs`, pref `mail.qrexport.enabled` |
| TB-ACC-23 | Profile import | Import accounts, settings, filters, messages, address books from Thunderbird/SeaMonkey profile, Outlook, Apple Mail, or a profile ZIP | `mailnews/import/modules/*ProfileImporter.sys.mjs`, `mailnews/import/src/nsOutlook*.cpp`, `nsAppleMailImport.cpp`, `L10N/aboutImport.ftl` |
| TB-ACC-24 | Profile export | Export the whole profile (accounts, messages, address books, settings) to a ZIP | `mailnews/export/modules/ProfileExporter.sys.mjs`, `L10N/aboutImport.ftl` (export strings) |
| TB-ACC-25 | Account sync (FxA) | Sync accounts, identities, address books, filters, keys across devices; **disabled at build time** ("until we get working servers again") | `mail/components/preferences/sync.js`, `sync.inc.xhtml`, `mail/moz.configure` (`enable_sync` returns False) |
| TB-ACC-26 | Extension-defined accounts | MailExtension API for accounts, folders, messages (read side) | `mail/components/extensions/schemas/accounts.json`, `folders.json`, `messages.json` |

## 2. Identities

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-ID-01 | Multiple identities per account | Each account has a default identity plus any number of extra ones, each with a label | `mailnews/base/src/nsMsgIdentity.cpp`, `AM/am-identities-list.js`, `AM/am-identity-edit.js`, `DTD/am-identity-edit.dtd` |
| TB-ID-02 | Identity fields | Name, email, reply-to, organization, identity label | `AM/am-main.js`, `DTD/am-main.dtd` |
| TB-ID-03 | Signature | Plain or HTML signature text, or signature from a file (text, HTML, image); placement above/below quote; on replies/forwards; suppress `-- ` separator | `DTD/am-main.dtd`, `DTD/am-addressing.dtd`, `sig_*` prefs in `mailnews/mailnews.js`, `AM/am-addressing.js` |
| TB-ID-04 | Attach vCard | Attach the identity's vCard to outgoing messages | `DTD/am-main.dtd` ("Attach my vCard"), `AM/am-main.js` |
| TB-ID-05 | Per-identity outgoing server | Choose SMTP server per identity | `AM/am-identity-edit.js`, `L10N/accountManager.ftl` |
| TB-ID-06 | Catch-all identity match | "Reply from this identity when delivery headers match" a pattern, checked against Delivered-To / Envelope-To / X-Original-To / To / Cc | `DTD/am-main.dtd`, pref `mail.compose.catchAllHeaders` in `mailnews/mailnews.js` |
| TB-ID-07 | Customize From address | Edit the From address freely in compose (for plus-addressing, aliases) | `L10N/messengercompose/messengercompose.ftl` ("Customize From Address"), `mail/components/compose/content/MsgComposeCommands.js` |
| TB-ID-08 | Copies and folders | Per identity: Fcc to Sent (or other folder), replies in folder of original, auto Cc/Bcc lists, Drafts folder, Templates folder, Archives folder, save confirmation | `AM/am-copies.js`, `DTD/am-copies.dtd`, `L10N/preferences/am-copies.ftl` |
| TB-ID-09 | Composition and addressing | Per identity: compose HTML or plain, auto-quote, reply above/below quote, LDAP directory override for autocomplete | `AM/am-addressing.js`, `DTD/am-addressing.dtd` |
| TB-ID-10 | Archive options | Per identity: single folder, yearly, or monthly archive folders; keep folder structure; recreate inbox folder | `mailnews/base/prefs/content/am-archiveoptions.js`, `L10N/preferences/am-archiveoptions.ftl`, `mail/modules/MessageArchiver.sys.mjs` |
| TB-ID-11 | Per-identity end-to-end encryption | OpenPGP key and S/MIME certs per identity, preferred technology, default encrypt/sign, encrypt subject, attach public key, Autocrypt-style key in headers | `mail/extensions/am-e2e/am-e2e.js`, `AME2E.sys.mjs`, `DTD/am-smime.dtd`, `L10N/openpgp/openpgp.ftl` |
| TB-ID-12 | Return receipt and DSN per identity | Override global return-receipt and delivery-status-notification behavior | `mailnews/extensions/mdn/am-mdn.js`, `DTD/am-mdn.dtd`, `dsn_*` prefs in `mailnews/mailnews.js` |

## 3. Folders

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-FLD-01 | Special folders | Inbox, Drafts, Sent, Templates, Archives, Trash, Spam/Junk, Outbox (Unsent Messages), Virtual; IMAP SPECIAL-USE mapping; localized names | `mailnews/base/public/nsMsgFolderFlags.idl`, `mailnews/base/src/nsMsgDBFolder.cpp`, pref `mail.useLocalizedFolderNames` |
| TB-FLD-02 | Folder CRUD | New folder/subfolder (incl. "folders only / messages only" on restricted servers), rename, delete, move by drag and drop, copy | `mailnews/base/content/newFolderDialog.js`, `renameFolderDialog.js`, `mail/base/content/modules/FolderCommands.mjs`, `L10N/new-folder-dialog.ftl` |
| TB-FLD-03 | Folder pane modes | All Folders, Unified Folders (smart mailboxes across accounts), Unread, Favorite, Recent, Tags; compact mode; multiple modes at once | `mail/base/content/about3Pane.js` (`folderPaneModes`), `mail/modules/SmartMailboxUtils.sys.mjs`, `L10N/about3Pane.ftl` |
| TB-FLD-04 | Folder pane options | Show/hide header, Get Messages / New Message buttons, folder size, total count, full path, hide Local Folders | `L10N/about3Pane.ftl`, `mail/modules/FolderPaneUtils.sys.mjs` |
| TB-FLD-05 | Favorite folders | Mark folder as favorite, shown in Favorite mode | `DTD/messenger.dtd` ("Favorite Folder"), `mail/base/content/about3Pane.js` |
| TB-FLD-06 | Folder color and icon | Custom icon color per folder, reset | `mail/modules/FolderTreeProperties.sys.mjs`, `L10N/folderprops.ftl` ("Icon Color") |
| TB-FLD-07 | Manual folder order | Reorder folders (move up/down, drag), reset order | `mail/base/content/modules/FolderReorderListener.mjs`, `L10N/about3Pane.ftl` |
| TB-FLD-08 | Folder properties | Name, type, location, message count, size on disk, include in global search, check for new mail, offline sync, repair | `mailnews/base/content/folderProps.js`, `L10N/folderprops.ftl` |
| TB-FLD-09 | IMAP sharing and quota | Folder ACL display and "Privileges…", per-folder quota bar; quota warning in status bar | `L10N/folderprops.ftl`, `kACLCapability`/`kQuotaCapability` in `mailnews/imap/src/nsImapCore.h`, `L10N/messenger.ftl` ("IMAP quota") |
| TB-FLD-10 | Subscribe | IMAP folder subscription dialog | `mailnews/base/content/subscribe.js`, `mailnews/base/src/nsSubscribableServer.cpp`, `L10N/subscribe.ftl` |
| TB-FLD-11 | Saved searches | Virtual folders over one or more folders with search terms; online search option; edit properties | `mailnews/base/content/virtualFolderProperties.js`, `mailnews/base/src/VirtualFolderWrapper.sys.mjs`, `mailnews/base/src/nsMsgXFVirtualFolderDBView.cpp`, `L10N/virtualFolderProperties.ftl` |
| TB-FLD-12 | Compact folders | Remove deleted messages from storage, manual or automatic with threshold, optional prompt | `mailnews/base/src/FolderCompactor.cpp`, `mail/base/content/compactFoldersDialog.js`, `L10N/compactFoldersDialog.ftl` |
| TB-FLD-13 | Repair folder | Rebuild a damaged folder index (.msf) | `L10N/folderprops.ftl` ("Repair Folder"), `mailnews/base/content/folderProps.js`, `mailnews/base/src/MboxRepair.sys.mjs` |
| TB-FLD-14 | Retention policy | Per folder or account: delete messages older than N days or keep only the newest N; always keep starred | `mailnews/base/content/RetentionSettingsUI.mjs`, `L10N/retention.ftl`, `mailnews/base/src/nsMsgPurgeService.cpp` |
| TB-FLD-15 | Empty Trash / empty Spam | Empty special folders, with confirmation | `L10N/messenger.ftl`, `mailnews/base/src/nsMsgDBFolder.cpp` |
| TB-FLD-16 | Mark folder / all folders read | Mark all in folder, all folders of account | `DTD/messenger.dtd` ("Mark All Folders Read") |
| TB-FLD-17 | Recent destinations | "Move/Copy to <recent>" and "Move Again"; MRU or alphabetical, configurable count | `L10N/messenger.ftl` ("Move Again", "Recent Destinations"), `L10N/preferences/preferences.ftl` (`recent-folders-*`) |
| TB-FLD-18 | Per-folder view settings | Sort, threading and columns per folder, apply to folder and children or to all | `L10N/about3Pane.ftl` (apply-columns/apply-view), `L10N/preferences/preferences.ftl` (`apply-view-settings-*`) |

## 4. Message list

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-LST-01 | Three-pane layout | Folder pane, message list, message pane; Classic, Wide and Vertical layouts; toggle panes | `mail/base/content/about3Pane.xhtml`, `about3Pane.js`, `DTD/messenger.dtd` ("Classic View", "Wide View", "Vertical View") |
| TB-LST-02 | Card view and table view | Cards (2 or 3 rows, default) or table rows; horizontal scroll in table | `mail/base/content/widgets/treeview/thread-card.mjs`, `thread-row.mjs`, pref `mail.threadpane.listview`, `L10N/preferences/preferences.ftl` (Appearance) |
| TB-LST-03 | Columns | Select, thread, star, attachment, subject, from, recipient, correspondents, spam status, date, received, status, size, tags, account, priority, unread count, total count, location, order received, delete; custom columns via extensions | `mail/base/content/modules/ThreadPaneColumns.mjs` |
| TB-LST-04 | Sorting | Sort by any column, ascending/descending, default sort for new folders | `mailnews/base/src/nsMsgDBView.cpp`, `L10N/about3Pane.ftl`, `L10N/preferences/preferences.ftl` (`default-sort-*`) |
| TB-LST-05 | Threading | Threaded, unthreaded, grouped by sort; expand/collapse all; threads with unread; watched threads with unread | `mailnews/base/src/nsMsgThreadedDBView.cpp`, `nsMsgGroupView.cpp`, `nsMsgSpecialViews.cpp`, `DTD/messenger.dtd` |
| TB-LST-06 | Watch / ignore thread | Watch thread, ignore thread, ignore subthread, show ignored threads | `DTD/messenger.dtd`, `mailnews/base/src/nsMsgDBView.cpp` |
| TB-LST-07 | Quick Filter Bar | Instant filter by text (sender, recipients, subject, body), unread, starred, contact, tags (any/all), attachment; sticky across folders | `mail/base/content/quickFilterBar.js`, `mail/modules/QuickFilterManager.sys.mjs`, `L10N/about3Pane.ftl` |
| TB-LST-08 | Message views | Predefined views (All, Unread, Tags, People I Know, Recent Mail, Last 5 Days, Not Spam, Has Attachments) plus custom views | `mailnews/extensions/mailviews/`, `mail/extensions/mailviews/`, `mail/modules/MailViewManager.sys.mjs`, `L10N/mailViews.ftl` |
| TB-LST-09 | Marking | Read/unread, thread read, read by date, all read, star (flag), spam/not spam; automatic mark-as-read immediately or after N seconds | `DTD/messenger.dtd`, `DTD/markByDate.dtd`, `mailnews/base/content/markByDate.js`, `L10N/preferences/preferences.ftl` (`auto-mark-as-read`) |
| TB-LST-10 | Status indicators | Replied, forwarded, redirected, new, attachments, priority, tags; unread/total per thread | `L10N/about3Pane.ftl` ("Message replied…"), `mail/base/content/widgets/treeview/thread-row.mjs` |
| TB-LST-11 | Navigation | Next/previous, next/previous unread (across folders), go back/forward, open in containing folder | `mail/base/content/msgViewNavigation.js`, `DTD/messenger.dtd` |
| TB-LST-12 | Delete, undelete, undo | Delete to Trash or permanently (Shift), undelete (IMAP mark-as-deleted model), undo/redo move, copy, delete, mark-all-read | `mailnews/base/src/nsMsgTransactionService.cpp`, `mailnews/imap/src/nsImapUndoTxn.cpp`, `mailnews/local/src/nsLocalUndoTxn.cpp`, `DTD/messenger.dtd` |
| TB-LST-13 | Move/copy | Drag and drop, Move To / Copy To menus, move to another account | `mailnews/base/src/nsMsgCopyService.cpp`, `mailnews/base/src/nsImapMoveCoalescer.cpp` |
| TB-LST-14 | Archive | One-key archive into identity's archive hierarchy | `mail/modules/MessageArchiver.sys.mjs` |
| TB-LST-15 | Multi-message summary | Selecting several messages or a collapsed thread shows a summary instead of one message | `mail/base/content/multimessageview.js`, `mail/modules/SummaryFrameManager.sys.mjs`, `L10N/multimessageview.ftl` |
| TB-LST-16 | Conversation view | Gloda-based conversation view (**exp**, **nightly**, `mail.thread.conversation.enabled` false) | `mail/base/content/widgets/conversation-view/conversation-view.mjs`, `mail/modules/ConversationOpener.sys.mjs`, `mail/app/profile/all-thunderbird.js` |
| TB-LST-17 | Address display format | Full name and email, email only, name only; show only display name for known contacts | `mail/modules/DisplayNameUtils.sys.mjs`, `L10N/preferences/preferences.ftl` (`address-display-*`) |
| TB-LST-18 | Tabs and windows | Open message in tab, new window, existing window; recently closed tabs; session restore | `mail/base/content/tabmail.js`, `mail/base/content/messageWindow.js`, `mail/modules/SessionStoreManager.sys.mjs` |
| TB-LST-19 | Panorama message database | New single global SQLite message DB and live views (**exp**, `mail.panorama.enabled` false; Nightly build only) | `mailnews/db/panorama/`, `mailnews/db/panorama/docs/index.md` |
| TB-LST-20 | Legacy per-folder DB | Mork `.msf` per folder holding headers and flags (current default) | `mailnews/db/msgdb/`, `mailnews/db/mork/` |

## 5. Reading

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-RD-01 | Message header pane | From, To, Cc, Bcc, Reply-To, Subject, Date, Tags, Newsgroups, Organization, User-Agent, Message-ID, References, In-Reply-To, List-* headers, Archived-At; show all recipients; normal/all headers | `mail/base/content/msgHdrView.js`, `msgHdrView.inc.xhtml`, `mail/base/content/widgets/header-fields.js`, `L10N/messageheader/headerFields.ftl` |
| TB-RD-02 | Header display settings | Show sender's profile picture, larger picture, large subject, always show full sender address, hide labels column | `L10N/messenger.ftl` ("Message Header Settings") |
| TB-RD-03 | Body display modes | Original HTML, Simple HTML, Plain Text, All Body Parts | `DTD/messenger.dtd` ("Message Body As"), `mailnews/mime/src/mimethsa.cpp`, `mimethpl.cpp`, `mimetpla.cpp` |
| TB-RD-04 | MIME rendering | Full MIME parser: multipart/alternative/related/mixed/digest, message/rfc822, enriched/rich text, inline images, charset handling, format=flowed | `mailnews/mime/src/` (libmime), `mailnews/mime/src/mimeParser.sys.mjs`, `jsmime.sys.mjs` |
| TB-RD-05 | Repair text encoding | Re-detect charset of a mis-labeled message | `L10N/menubar.ftl` ("Repair Text Encoding") |
| TB-RD-06 | Remote content blocking | Block remote images/styles by default; allow per message, per sender or per site; exceptions list | `mailnews/base/src/nsMsgContentPolicy.cpp`, `mail/components/preferences/permissions.js`, `L10N/preferences/preferences.ftl` (`remote-content-*`) |
| TB-RD-07 | Scam (phishing) detection | Warn on likely scam messages and mismatched link text; report scam | `mail/modules/PhishingDetector.sys.mjs`, `L10N/preferences/preferences.ftl` (`scam-detection-*`) |
| TB-RD-08 | Attachments pane | List, open, save, save all, detach, detach all, delete, delete all; show inline; open containing folder | `mail/base/content/msgHdrView.js`, `mail/modules/AttachmentInfo.sys.mjs`, `mail/base/content/widgets/attachment-list.mjs`, `DTD/msgHdrViewOverlay.dtd` |
| TB-RD-09 | Attachment handling actions | Per content type: preview in app (PDF), open with app, save, always ask; download folder | `mail/components/preferences/applicationManager.js`, `L10N/preferences/preferences.ftl` (`applications-*`) |
| TB-RD-10 | Dark message mode | Darken message content in dark theme, per-message toggle | `mail/base/content/modules/DarkReader.mjs`, prefs `mail.dark-reader.*` |
| TB-RD-11 | Display options | Fixed/variable width, wrap long lines, quoted text style/size/color, emoticons as graphics, underline links, fonts, zoom | `L10N/preferences/preferences.ftl` (`display-*`, `quoted-text-*`), `mail/base/content/viewZoomOverlay.js`, `mail/components/preferences/fonts.js` |
| TB-RD-12 | Mailing list header actions | Reply to list, List-Unsubscribe/Subscribe/Archive links in header | `mail/base/content/msgHdrView.js` (`list-unsubscribe`), `L10N/messageheader/headerFields.ftl` |
| TB-RD-13 | Return receipts (MDN) | Honor read-receipt requests: never, always, ask; rules for outside domain / not in To/Cc | `mailnews/extensions/mdn/MDNService.sys.mjs`, `nsMsgMdnGenerator.cpp`, `mail/components/preferences/receipts.js`, `L10N/preferences/receipts.ftl` |
| TB-RD-14 | View source | Show raw RFC 5322 source | `mail/base/content/viewSource.js`, `L10N/viewSource.ftl` |
| TB-RD-15 | Find in message | In-message find bar, find again | `mail/base/content/widgets/lazy-findbar.mjs`, `DTD/messenger.dtd` ("Find in This Message…") |
| TB-RD-16 | Print | Print message(s) with preview | `mail/base/content/printUtils.js` |
| TB-RD-17 | Save message | Save as `.eml`, HTML or text; save multiple | `mail/modules/MessageSaver.sys.mjs` |
| TB-RD-18 | Open `.eml` files | Open a message file from disk in a window | `mail/components/MessengerContentHandler.sys.mjs`, `DTD/messenger.dtd` ("Saved Message…") |
| TB-RD-19 | Header context actions | Add sender to address book, edit contact, compose to, copy address, create filter from message | `mail/base/content/msgHdrView.js`, `mail/base/content/editContactPanel.js`, `DTD/messenger.dtd` |
| TB-RD-20 | Caret browsing | F7 caret browsing in message pane | `L10N/messenger.ftl` ("Caret Browsing") |
| TB-RD-21 | Link handling | Links open in default browser; copy link | `mail/modules/LinkHelper.sys.mjs` |
| TB-RD-22 | Calendar invitations in mail | Inline iMIP invitation bar (accept/decline) | `calendar/` (out of scope; noted for parity) |

## 6. Composing

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-CMP-01 | Compose window | New, reply, reply all, reply to list, reply to sender only, forward (inline / as attachment), redirect (bounce), edit as new, edit draft, smart reply | `mail/components/compose/content/MsgComposeCommands.js`, `mailnews/compose/src/nsMsgCompose.cpp`, `DTD/messenger.dtd` |
| TB-CMP-02 | Addressing widget | To, Cc, Bcc, Reply-To, Followup-To pills; drag pills between fields; expand list; invalid/unknown-address hints; "too many public recipients" Bcc nudge | `mail/components/compose/content/addressingWidgetOverlay.js`, `mail/components/compose/content/composeWidgets.mjs`, `L10N/messengercompose/messengercompose.ftl` |
| TB-CMP-03 | Recipient autocomplete | Local address books, collected addresses, LDAP directories, "my domain" fallback | `mailnews/addrbook/src/AbAutoCompleteSearch.sys.mjs`, `AbLDAPAutoCompleteSearch.sys.mjs`, `AbAutoCompleteMyDomain.sys.mjs` |
| TB-CMP-04 | Rich text (HTML) editor | Fonts, sizes, colors, paragraph styles, headings, lists, indent, alignment, links, anchors, images, tables, horizontal line, special characters, emoticons, page colors, insert HTML, table of contents | `mail/components/compose/content/editor.js`, `ComposerCommands.js`, `mail/components/compose/content/dialogs/Ed*.js`, `DTD/messengercompose/editorOverlay.dtd` |
| TB-CMP-05 | Math insertion | Insert LaTeX math rendered as MathML | `mail/components/compose/texzilla/TeXZilla.js`, `mail/components/compose/content/dialogs/EdInsertMath.js` |
| TB-CMP-06 | Plain text compose | Plain text mode, rewrap, paste as quotation, paste without formatting, line wrap | `DTD/messengercompose/messengercompose.dtd`, pref `mailnews.wraplength` |
| TB-CMP-07 | Sending format | Automatic, HTML only, plain only, both (multipart/alternative); per-message override | `L10N/preferences/preferences.ftl` (`compose-format-*`), `L10N/messengercompose/messengercompose.ftl` ("Sending format") |
| TB-CMP-08 | Quoting | Auto-quote on reply, reply above/below, quote original, attribution line | `mailnews/compose/src/nsMsgQuote.cpp`, `mail/modules/QuoteSanitizer.sys.mjs`, `AM/am-addressing.js` |
| TB-CMP-09 | Attachments | Attach files, web page, message, personal vCard, OpenPGP public key; rename, reorder, remove; attachment pane | `mail/components/compose/content/MsgComposeCommands.js`, `mailnews/compose/src/nsMsgAttachment.cpp`, `L10N/messengercompose/messengercompose.ftl` |
| TB-CMP-10 | Attachment reminder | Warn on send if body mentions attachment keywords but none attached; keyword list editable | `mail/modules/AttachmentChecker.worker.js`, `mail/components/preferences/attachmentReminder.js`, `L10N/preferences/attachment-reminder.ftl` |
| TB-CMP-11 | Filelink (cloud attachments) | Upload large attachments to a cloud provider and insert a link; size threshold; providers via extensions | `mail/components/cloudfile/cloudFileAccounts.sys.mjs`, `mail/components/compose/content/cloudAttachmentLinkManager.js`, `mail/components/compose/content/bigFileObserver.js`, `mail/components/extensions/schemas/cloudFile.json` |
| TB-CMP-12 | Link previews | Offer to embed a preview card when pasting a URL | `L10N/preferences/preferences.ftl` (`add-link-previews`), `L10N/messengercompose/messengercompose.ftl` |
| TB-CMP-13 | Spell checking | Inline and pre-send spell check, multiple dictionaries, subject spell check, personal dictionary | `mail/components/compose/content/dialogs/EdSpellCheck.js`, `mail/components/compose/content/subjectSpellCheckMenu.mjs`, `EdDictionary.js` |
| TB-CMP-14 | Drafts and autosave | Save draft, autosave every N minutes | `L10N/preferences/preferences.ftl` (`autosave-draft-label`), `mailnews/compose/src/nsMsgCompose.cpp` |
| TB-CMP-15 | Templates | Save as template, new message from template, edit template, reply-with-template filter action | `DTD/messenger.dtd`, `mailnews/base/src/TemplateUtils.sys.mjs` |
| TB-CMP-16 | Send later / Outbox | Queue in Outbox ("Unsent Messages"), send unsent messages, automatic send when back online | `mailnews/compose/src/nsMsgSendLater.cpp`, pref `mail.default_sendlater_uri`, `mail/components/activity/modules/sendLater.sys.mjs` |
| TB-CMP-17 | Message options | Priority, return receipt request, delivery status notification, character encoding, custom headers | `DTD/messengercompose/messengercompose.dtd`, `L10N/customHeaders.ftl` |
| TB-CMP-18 | Send confirmations | Confirm keyboard-shortcut send, empty subject reminder, no recipients | `L10N/preferences/preferences.ftl` (`warn-on-send-accel-key`), `L10N/messengercompose/messengercompose.ftl` |
| TB-CMP-19 | Composition defaults | Default font, size, text and background color, paragraph spacing, forward inline vs attachment | `mail/components/preferences/compose.js`, `L10N/preferences/preferences.ftl` (Compose tab) |
| TB-CMP-20 | Contacts sidebar | Address book sidebar in compose for picking recipients | `mail/components/addrbook/content/abContactsPanel.js` |
| TB-CMP-21 | MIME construction | Build and encode outgoing MIME, 8-bit/QP/base64 selection | `mailnews/compose/src/MimeMessage.sys.mjs`, `MimeEncoder.sys.mjs`, `MessageSend.sys.mjs` |
| TB-CMP-22 | `mailto:` handling | Register as handler; open compose from `mailto:` links | `mailnews/compose/src/MailtoProtocolHandler.sys.mjs` |
| TB-CMP-23 | Windows Simple MAPI | "Send to mail recipient" from other Windows apps | `mailnews/mapi/` |

## 7. Search

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-SRCH-01 | Global search (Gloda) | Indexed full-text search across all accounts with SQLite FTS3 and Porter stemming/CJK tokenizer | `mailnews/db/gloda/modules/`, `mailnews/extensions/fts3/`, `L10N/preferences/preferences.ftl` (`enable-global-search-label`) |
| TB-SRCH-02 | Faceted results | Results tab with facets (people, tags, folders, attachment types, dates timeline); plain-list option | `mail/base/content/glodaFacetView.js`, `glodaFacetTab.js`, `mail/base/content/widgets/glodaFacet.js`, `L10N/glodaFacetView.ftl` |
| TB-SRCH-03 | Search bar autocomplete | Unified toolbar search box with Gloda suggestions | `mail/components/unifiedtoolbar/content/items/global-search-bar.mjs`, `mail/base/content/widgets/gloda-autocomplete-input.js` |
| TB-SRCH-04 | Advanced search dialog | Term-based search over chosen folders and subfolders, server-side search, save as search folder, act on results | `mail/base/content/searchDialog.js`, `L10N/searchDialog.ftl` |
| TB-SRCH-05 | Search terms | Subject, From, To, Cc, To-or-Cc, any address, body, date, age in days, priority, status, tags, size, attachment status, spam status/score/origin, custom headers, address book membership | `mailnews/search/src/nsMsgSearchTerm.cpp`, `L10N/searchWidgets.ftl` |
| TB-SRCH-06 | Server-side search | IMAP SEARCH delegation and local/offline search | `mailnews/search/src/nsMsgImapSearch.cpp`, `nsMsgLocalSearch.cpp`, `nsMsgSearchSession.cpp` |
| TB-SRCH-07 | Quick filter body search | Quick Filter Bar text search incl. body, continue search across all folders | `mail/modules/QuickFilterManager.sys.mjs`, `L10N/about3Pane.ftl` |
| TB-SRCH-08 | Desktop search integration | Expose messages to Windows Search or macOS Spotlight | `mail/components/search/WinSearchIntegration.sys.mjs`, `SpotlightIntegration.sys.mjs`, `mail/components/search/mdimporter/` |
| TB-SRCH-09 | Search settings | Settings search box and diacritic matching | `mail/components/preferences/findInPage.js`, pref `mail.matchdiacritics` |

## 8. Filters and spam

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-FLT-01 | Message filters | Per-account ordered filter list; enable/disable, new/edit/copy/delete, move to top/bottom | `mailnews/search/src/nsMsgFilterList.cpp`, `nsMsgFilter.cpp`, `mail/base/content/FilterListDialog.js`, `L10N/filterEditor.ftl` |
| TB-FLT-02 | Filter triggers | Getting new mail (before or after spam classification), manually run, after sending, archiving, periodically | `L10N/filterEditor.ftl`, `mailnews/search/src/PeriodicFilterManager.sys.mjs` |
| TB-FLT-03 | Filter conditions | Same term set as search, match all / any / all messages, custom header terms | `mailnews/search/src/nsMsgSearchTerm.cpp`, `L10N/searchWidgets.ftl`, `L10N/customHeaders.ftl` |
| TB-FLT-04 | Filter actions | Move, copy, forward, reply with template, mark read/unread, star, set priority, tag, set spam status, delete, delete from / fetch from POP server, ignore thread/subthread, watch thread, stop filter execution, custom (extension) actions | `mailnews/search/src/nsMsgFilterService.cpp`, `L10N/searchWidgets.ftl` |
| TB-FLT-05 | Run filters on demand | Run on folder, selected messages, or chosen folder from the filter dialog | `DTD/messenger.dtd` ("Run Filters on Folder"), `L10N/filterEditor.ftl` |
| TB-FLT-06 | Filter log | Log of filter actions and errors | `L10N/filterEditor.ftl` ("Filter Log") |
| TB-FLT-07 | Create filter from message | Prefill filter from selected message or address | `DTD/messenger.dtd` ("Create Filter From Message…") |
| TB-FLT-08 | OpenPGP filter actions | Decrypt permanently / create decrypted copy via filter | `mail/extensions/openpgp/content/modules/filters.sys.mjs`, `L10N/messenger.ftl` ("Create Decrypted Copy In") |
| TB-FLT-09 | Adaptive spam filter | Bayesian classifier trained by marking spam / not spam, reset training data | `mailnews/extensions/bayesian-spam-filter/nsBayesianFilter.cpp`, `mailnews/base/content/junkCommands.js`, `L10N/preferences/preferences.ftl` (`reset-spam-button`) |
| TB-FLT-10 | Spam settings per account | Enable detection, trust address-book senders (allowlist), trust server headers (SpamAssassin, Bogofilter, DSPAM, POPFile, SpamPal, SpamCatcher), move spam to folder, auto-delete spam older than N days | `AM/am-junk.js`, `L10N/preferences/am-spam.ftl`, `mailnews/search/src/*.sfd`, `mailnews/base/src/nsSpamSettings.cpp` |
| TB-FLT-11 | Global spam handling | On mark as spam: move to Spam or delete; mark read when marked manually or auto; spam log | `L10N/preferences/preferences.ftl` (`spam-*`), `mailnews/base/content/junkLog.js`, `L10N/preferences/spamLog.ftl` |
| TB-FLT-12 | Antivirus quarantine | Write incoming mail to a temp file first so antivirus can quarantine it | `mailnews/base/src/nsQuarantinedOutputStream.cpp`, `L10N/preferences/preferences.ftl` (`antivirus-isolate-label`) |

## 9. Tags

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-TAG-01 | Tag service | User-defined tags (name, color, IMAP keyword), five defaults (Important, Work, Personal, To Do, Later) | `mailnews/base/src/nsMsgTagService.cpp`, `mail/modules/TagUtils.sys.mjs`, `L10N/messenger.ftl` |
| TB-TAG-02 | Tag management | New, edit, delete, reorder tags in settings; New Tag from message menu | `mail/base/content/newTagDialog.js`, `mail/components/preferences/general.js`, `L10N/preferences/new-tag.ftl` |
| TB-TAG-03 | Tag messages | Toggle tags via menu, number keys 1–9, remove all tags (0); colored display in list and header | `L10N/messenger.ftl` ("Remove All Tags", "Manage Tags…"), `mail/base/content/widgets/treeview/thread-card-tags.mjs` |
| TB-TAG-04 | Tags as IMAP keywords | Tags synced to server as IMAP keywords/flags | `mailnews/imap/src/nsImapMailFolder.cpp`, search term "IMAP Flag" in `L10N/searchWidgets.ftl` |
| TB-TAG-05 | Tag-based navigation | Tags folder-pane mode, tag quick filter (any/all/untagged), tag search term, tag filter action | `mail/base/content/about3Pane.js` (`tags` mode), `L10N/about3Pane.ftl` |
| TB-TAG-06 | Star (flag) | Single "starred" flag separate from tags, synced as `\Flagged` | `mail/base/content/modules/ThreadPaneColumns.mjs` (`flaggedCol`) |

## 10. Address book

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-AB-01 | Address book tab | Books list, contact list (list or table layout), contact detail/editor, search, sort | `mail/components/addrbook/content/aboutAddressBook.js`, `L10N/addressbook/aboutAddressBook.ftl` |
| TB-AB-02 | Local address books | SQLite-backed books; Personal and Collected Addresses by default; create/rename/delete | `mailnews/addrbook/modules/SQLiteDirectory.sys.mjs`, `AddrBookDirectory.sys.mjs`, `AddrBookManager.sys.mjs` |
| TB-AB-03 | CardDAV | Two-way sync with CardDAV servers, discovery, OAuth (Google, Fastmail) | `mailnews/addrbook/modules/CardDAVDirectory.sys.mjs`, `CardDAVUtils.sys.mjs`, `L10N/addressbook/abCardDAVDialog.ftl` |
| TB-AB-04 | LDAP directories | Read-only LDAP lookup and autocomplete; offline replication | `mailnews/addrbook/modules/LDAPDirectory.sys.mjs`, `LDAPReplicationService.sys.mjs`, `LDAPClient.sys.mjs` |
| TB-AB-05 | OS address books | macOS Contacts and Outlook address book as directories | `mailnews/addrbook/src/nsAbOSXDirectory.mm`, `nsAbOutlookDirectory.cpp` |
| TB-AB-06 | vCard 4 contacts | Name parts, nickname, multiple emails/phones/addresses/URLs, IM, organization, title, role, birthday/anniversary, time zone, notes, custom fields, photo | `mail/components/addrbook/content/vcard-edit/`, `mailnews/addrbook/modules/VCardUtils.sys.mjs`, `L10N/addressbook/vcard.ftl` |
| TB-AB-07 | Mailing lists | Local named recipient lists | `mailnews/addrbook/modules/AddrBookMailingList.sys.mjs`, `mail/components/addrbook/content/abMailListDialog.xhtml` |
| TB-AB-08 | Collected addresses | Auto-save outgoing recipients to a chosen book | `mailnews/compose/src/AddressCollector.sys.mjs`, `L10N/preferences/preferences.ftl` (`collect-outgoing-label`) |
| TB-AB-09 | Quick add / edit contact | Star-like popup in header to add or edit a sender | `mail/base/content/editContactPanel.js` |
| TB-AB-10 | Advanced address search | Term-based contact search across books | `mail/components/addrbook/content/abSearchDialog.js` |
| TB-AB-11 | Import / export contacts | Import CSV/TSV, LDIF, vCard, Mork `.mab`, SQLite; export CSV, LDIF, vCard | `mailnews/import/modules/AddrBookFileImporter.sys.mjs`, `mailnews/addrbook/modules/AddrBookUtils.sys.mjs`, `mailnews/addrbook/src/nsAbLDIFService.cpp` |
| TB-AB-12 | Print contacts | Print selected contacts | `L10N/addressbook/aboutAddressBook.ftl` ("Print…") |
| TB-AB-13 | Contact avatars | Contact photo shown in message list cards and header | `mail/base/content/widgets/treeview/contact-avatar.mjs`, `L10N/contact-avatar.ftl` |

## 11. Offline and storage

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-OFF-01 | Work offline | Manual offline toggle, startup prompt, auto-detect network state | `mail/base/content/mail-offline.js`, `mailnews/extensions/offline-startup/OfflineStartup.sys.mjs`, `mailnews/base/src/nsMsgOfflineManager.cpp` |
| TB-OFF-02 | IMAP offline store | Keep full copies of IMAP messages locally; per account and per folder | `mailnews/imap/src/nsImapOfflineSync.cpp`, `AM/am-offline.js`, `mailnews/base/content/msgSelectOfflineFolders.js` |
| TB-OFF-03 | Auto-sync | Background download of messages for offline, age and size limits | `mailnews/imap/src/nsAutoSyncManager.cpp`, `nsAutoSyncState.cpp`, `mail/components/activity/modules/autosync.sys.mjs`, `DTD/am-offline.dtd` |
| TB-OFF-04 | Download/Sync Now | Manual sync of offline folders and send unsent messages | `mailnews/base/content/msgSynchronize.js`, `DTD/msgSynchronize.dtd` |
| TB-OFF-05 | Offline operations queue | Moves, deletes, flag changes made offline replayed on reconnect | `mailnews/imap/src/nsImapOfflineSync.cpp`, `mailnews/db/msgdb/` (offline ops) |
| TB-OFF-06 | Exchange offline storage | Offline copies for EWS/Graph folders | `mailnews/protocols/common/src/OfflineStorage.cpp`, `mailnews/protocols/exchange/src/ExchangeFetchMsgsToOffline.cpp` |
| TB-OFF-07 | Disk cache | Disk cache for non-offline IMAP messages, size limit, clear on exit | pref `mail.imap.use_disk_cache2`, `L10N/preferences/preferences.ftl` (`temporary-storage-legend`) |
| TB-OFF-08 | Activity Manager | Log of sends, downloads, moves, indexing, autosync | `mail/components/activity/`, `L10N/activity.ftl` |

## 12. Security and privacy

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-SEC-01 | OpenPGP (built in) | Sign/encrypt/decrypt PGP/MIME and inline via RNP; no external GnuPG needed | `mail/extensions/openpgp/content/modules/RNP.sys.mjs`, `mimeEncrypt.sys.mjs`, `mimeDecrypt.sys.mjs`, `mimeVerify.sys.mjs`, `mail/extensions/openpgp/README.md` |
| TB-SEC-02 | OpenPGP key manager | Generate (RSA, ECC, PQC subkey), import from file/clipboard/URL, export, backup secret keys, revoke, change expiry, view photo ID, key acceptance levels | `mail/extensions/openpgp/content/ui/enigmailKeyManager.js`, `keyWizard.js`, `keyDetailsDlg.js`, `changeExpiryDlg.js`, `L10N/openpgp/openpgp.ftl` |
| TB-SEC-03 | Key discovery | Keyservers (configurable list), Web Key Directory (WKD), keys collected from incoming messages | `mail/extensions/openpgp/content/modules/keyserver.sys.mjs`, `wkdLookup.sys.mjs`, `CollectedKeysDB.sys.mjs`, `L10N/preferences/preferences.ftl` (`email-e2ee-key-servers-*`) |
| TB-SEC-04 | Smartcard via GnuPG | Use secret keys held in external GnuPG (e.g. smartcards) through GPGME | `mail/extensions/openpgp/content/modules/GPGME.sys.mjs`, `GPGMELib.sys.mjs` |
| TB-SEC-05 | Key Assistant | Resolve missing/unaccepted recipient keys from compose | `mail/extensions/openpgp/content/ui/keyAssistant.js`, `L10N/openpgp/keyAssistant.ftl` |
| TB-SEC-06 | S/MIME | Sign/encrypt/decrypt CMS; per-identity certs; fetch recipient certs from LDAP; certificate manager; PKCS#11 security devices | `mailnews/extensions/smime/`, `mailnews/mime/src/mimecms.cpp`, `mimemcms.cpp`, `mail/extensions/smime/`, `L10N/smime/smime.ftl` |
| TB-SEC-07 | Message security status | Signature/encryption status in header with details popup | `mail/base/content/msgSecurityPane.js`, `L10N/openpgp/msgReadStatus.ftl` |
| TB-SEC-08 | Encrypt subject | Protected headers for encrypted subject | `L10N/messengercompose/messengercompose.ftl` ("Encrypt Subject") |
| TB-SEC-09 | Automatic encryption | Turn encryption on when all recipients have accepted keys/certs; turn off and notify when not | `L10N/preferences/preferences.ftl` (`email-e2ee-automatism*`) |
| TB-SEC-10 | Primary Password | Encrypts saved passwords and OpenPGP secret-key passphrases | `mail/modules/PrimaryPassword.sys.mjs`, `mail/extensions/openpgp/content/modules/masterpass.sys.mjs` |
| TB-SEC-11 | Client certificates and OCSP | Choose client cert automatically or ask; OCSP checking; certificate manager | `L10N/preferences/preferences.ftl` (`certificate-*`, `ocsp-check-label`) |
| TB-SEC-12 | Secure DNS (DoH) | DNS over HTTPS modes and provider | `L10N/preferences/preferences.ftl` (`preferences-doh-*`) |
| TB-SEC-13 | Web content privacy | Cookies, history, Global Privacy Control for web content shown in tabs | `mail/components/preferences/privacy.js`, `cookies.js` |
| TB-SEC-14 | Clear recent history | Sanitize history, cookies, cache | `mail/base/content/sanitize.js`, `DTD/sanitize.dtd` |
| TB-SEC-15 | Remote content and scam | See TB-RD-06, TB-RD-07 | as listed there |

## 13. Notifications and OS integration

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-NOT-01 | New mail alert | Custom alert or system notification, with customizable fields (sender, subject, preview) | `mailnews/base/content/newmailalert.js`, `mailnews/base/src/MailNotificationManager.sys.mjs`, `mail/components/preferences/notifications.js` |
| TB-NOT-02 | Sounds | Default system sound or custom sound on new mail | `mail/modules/NotificationSounds.sys.mjs`, `L10N/preferences/preferences.ftl` (`mail-play-sound-label`) |
| TB-NOT-03 | Tray and badges | Tray icon for unread, minimize/close/start to tray, unread badge on dock/taskbar | `mailnews/base/src/SystemTrayBadgeManager.sys.mjs`, `MessengerLinuxTrayIntegration.sys.mjs`, `WinUnreadBadge.sys.mjs`, `mail/base/content/closeToTray.mjs`, `mail/components/preferences/dockoptions.js` |
| TB-NOT-04 | Default mail client | Check/set as default mail, news, feed client | `mail/components/shell/`, `mail/base/content/systemIntegrationDialog.js` |
| TB-NOT-05 | In-app notifications | Server-pushed product notices inside the app | `mail/components/inappnotifications/` |

## 14. Settings, UI and extensibility

| ID | Feature | Description | Source |
| -- | ------- | ----------- | ------ |
| TB-SET-01 | Settings tab | Panes General, Appearance, Composition, Privacy & Security, Chat, Calendar, Sync, Export for Mobile, Beta Features; searchable | `mail/components/preferences/preferences.js`, `*.inc.xhtml`, `L10N/preferences/preferences.ftl` |
| TB-SET-02 | Account Settings | Per account: Server Settings, Copies & Folders, Composition & Addressing, Spam Settings, Synchronization & Storage, End-to-End Encryption, Return Receipts; Outgoing Server list | `AM/AccountManager.js`, `AM/am-*.js`, `L10N/accountSettings.ftl` |
| TB-SET-03 | Appearance | Accent color, message list style and default sort/threading, density (compact/default/relaxed), font size | `mail/components/preferences/appearance.mjs`, `mail/modules/UIDensity.sys.mjs`, `mail/modules/UIFontSize.sys.mjs` |
| TB-SET-04 | Language and date formatting | UI language packs, app vs regional date locale | `mail/components/preferences/messengerLanguages.js`, `L10N/preferences/languages.ftl` |
| TB-SET-05 | Network | Proxy settings, offline settings, cache size | `mail/components/preferences/connection.js`, `offline.js` |
| TB-SET-06 | Unified toolbar | Per-space customizable toolbar (get messages, write, search, quick filter, view picker, spam, delete, compact, add-ons) | `mail/components/unifiedtoolbar/` |
| TB-SET-07 | Spaces toolbar | Left bar switching Mail, Address Book, Calendar, Tasks, Chat, Settings, extension spaces | `mail/base/content/spacesToolbar.js`, `mail/components/extensions/schemas/spaces.json` |
| TB-SET-08 | Keyboard shortcuts | Shortcut overview and customizable shortcuts | `mail/modules/ShortcutsManager.sys.mjs`, `mail/components/customizableshortcuts/`, `L10N/customizableshortcuts/customizableShortcuts.ftl` |
| TB-SET-09 | Themes | Light/dark/system themes and add-on themes | `mail/themes/`, `mail/components/extensions/schemas/theme.json` |
| TB-SET-10 | MailExtensions | WebExtension API for mail (accounts, folders, messages, compose, mailTabs, messageDisplay, addressBook, cloudFile, menus, identities, spaces) | `mail/components/extensions/schemas/*.json` |
| TB-SET-11 | Enterprise policies | Admin policies (`policies.json`), about:policies | `mail/components/enterprisepolicies/`, `mail/components/enterprisepolicies/documentation/policies.yaml` |
| TB-SET-12 | Config editor | about:config advanced pref editor | `L10N/preferences/preferences.ftl` (`advanced-settings-button`) |
| TB-SET-13 | Troubleshooting | Troubleshoot mode (safe mode), Troubleshooting Information page, profile downgrade guard | `mail/base/content/troubleshootMode.js`, `mail/components/about-support/`, `mail/base/content/profileDowngrade.js` |
| TB-SET-14 | Updates and telemetry | App updates, data collection opt-out, crash reports | `L10N/preferences/preferences.ftl` (`update-*`, `collection-*`) |
| TB-SET-15 | Migration | Profile/pref migrations between versions | `mail/modules/MailMigrator.sys.mjs` |

## Notes for the parity matrix

- Thunderbird's back-end split is protocol-per-directory (`mailnews/imap`, `mailnews/local` for POP3
  and local folders, `mailnews/compose` for SMTP, `mailnews/protocols/exchange` + `rust/` for EWS and
  Graph). Folder, message DB, search, filters and MIME are shared across protocols in
  `mailnews/base`, `mailnews/db`, `mailnews/search` and `mailnews/mime`.
- Exchange support is new code (C++ shell + Rust clients) and the only part already written in
  Rust; it is the closest precedent for flock's Rust backend.
- Nightly-only or experimental items in this catalogue: conversation view (TB-LST-16), Panorama DB
  (TB-LST-19), Account Hub variants (TB-ACC-10), FxA sync (TB-ACC-25, compiled out).
- Thunderbird has no JMAP and no Sieve/ManageSieve support in this tree: a case-insensitive search
  for `jmap` and `sieve` under `mail/`, `mailnews/` and `rust/` finds only a passing mention of
  JMAP in `mailnews/base/docs/folder_storage.md` and unrelated uses of the word "sieve". Server-side
  filtering is limited to trusting spam headers.
