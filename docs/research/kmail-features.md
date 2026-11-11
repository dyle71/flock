# KMail mail feature catalogue

Research for dyle71/flock#3, part of the flock v1 specification map (#1).

## Question

Which mail features does KMail ship (accounts/resources, folders, message list,
reading, composing, search, filters, Sieve, tags, identities, address book,
offline, security, settings), and where in `/home/dyle/Source/other/kmail` does
each one live? Where does KMail depend on Akonadi or other KDE PIM components
outside that tree? This catalogue covers mail only. Calendar, contacts, and
notes apps are in scope only where mail features call into them.

## Sources

- **KMail source tree** at `/home/dyle/Source/other/kmail`, commit
  `98c911893b2cc4c793e0d2a6a2723170a9391ee1` (2026-11-11). All paths below are
  relative to that root. The files I relied on most:
  - `src/kmmainwidget.cpp`: main-window actions, including the folder
    properties pages it registers (lines 284–292) and the action IDs.
  - `src/messageactions.cpp`: reply, forward, and message-menu actions.
  - `src/editor/kmcomposerwin.cpp`: composer actions and crypto.
  - `src/kmcommands.h`: the command classes.
  - `src/kmkernel.{h,cpp}`: online/offline state and the D-Bus API.
  - `src/kmlaunchexternalcomponent.cpp`: external apps KMail launches.
  - `src/configuredialog/*.cpp`, `src/identity/identitydialog.cpp`: settings
    and identity tabs.
  - `src/settings/kmail.kcfg`: KMail's own config keys.
  - `agents/*`: the Akonadi agents that ship in this tree.
  - `CMakeLists.txt`, `src/CMakeLists.txt`: external dependencies.
  - `#include` lines across `src/` and `agents/`, which show which external
    library provides each class.
- **KMail Handbook** (DocBook in the tree): `doc/kmail2/getting-started.docbook`,
  `using-kmail.docbook`, `configure.docbook`, `menus.docbook`, `faq.docbook`,
  and the agent manuals `doc/akonadi_archivemail_agent/`,
  `doc/akonadi_followupreminder_agent/`, `doc/akonadi_sendlater_agent/`.
- **What I could not read.** The external KDE PIM repositories (akonadi,
  kdepim-runtime, messagelib, mailcommon, libksieve, libkleo, pimcommon,
  kdepim-addons, and others) are not checked out on this machine, and only a
  few of their headers are installed under `/usr/include/KPim6`. So for the
  rows marked "external", I identified the component from KMail's `#include`
  lines and link targets, plus the handbook. I did not read those components'
  own source.

### How to read the tables

- The **Source** column gives paths inside the KMail tree.
- The **External** column names the component that implements the feature, or
  most of it. The components are:
  - CMake/library names, such as `MailCommon`, `MessageList`, `KSieveUi`, and
    `Akonadi`.
  - Repository names, such as `kdepim-runtime` and `kdepim-addons`.
  - App names, such as `kaddressbook` and `kleopatra`.
- "KMail-only" means the KMail tree contains the whole implementation, apart
  from generic Qt/KF6 code.
- "Plugin" means the feature is a plugin, usually from `kdepim-addons`, that
  the user can switch on or off under Settings → Configure KMail → Plugins
  (`src/configuredialog/configurepluginpage.cpp`).

---

## Accounts

KMail does not talk IMAP, POP3, or SMTP itself. Receiving accounts are
**Akonadi resources**, and sending accounts are **MailTransport transports**.
KMail is a front end that configures and drives them.

| Feature | Description | Source | External |
|---|---|---|---|
| Account wizard | First-run and Settings → Add Account wizard. It autodetects provider settings, can create or import a GnuPG key, and can turn on offline download for IMAP. | `src/util.cpp` (`executeAccountWizard`), `src/kmlaunchexternalcomponent.cpp`; handbook `getting-started.docbook#account-wizard` | external app: `accountwizard` (kmail-account-wizard repo) |
| Receiving accounts (IMAP, POP3, Maildir, mbox, EWS) | Add, modify, remove, or restart a receiving account. The type comes from the Akonadi agent type dialog. | `src/configuredialog/configureaccountpage.cpp` (`AccountsPageReceivingTab`), `src/ui/accountspagereceivingtab.ui`; type IDs in `src/userfeedback/accountinfosource.cpp` | external: Akonadi (`AgentManager`, `AgentTypeDialog`); resources `akonadi_imap_resource`, `akonadi_pop3_resource`, `akonadi_maildir_resource`, `akonadi_mbox_resource`, `akonadi_ews_resource` in **kdepim-runtime** |
| Server settings and autodetection | Host, port, TLS/STARTTLS, nine auth methods (PLAIN, LOGIN, CRAM-MD5, DIGEST-MD5, NTLM, GSSAPI, APOP, Gmail OAuth, and others), and an "Auto Detect" button. | handbook `getting-started.docbook#receiving-mail` | external: kdepim-runtime resource config dialogs |
| Per-account retrieval options | Check on startup, include in manual check, switch offline on shutdown, interval checking. | `src/configuredialog/configureaccountpage.cpp`, `src/kmkernel.cpp` (~line 1695, `OfflineOnShutdown`) | external: Akonadi resource settings over D-Bus |
| POP3 specifics | Leave messages on the server (by age, count, or size), pipelining, destination folder, pre-command. | `src/kmkernel.cpp` (`createPop3SettingsInterface`) | external: `MailCommon` (`pop3settings.h`), kdepim-runtime POP3 resource |
| IMAP specifics | Automatic expunge/compact, server-side subscriptions, local subscriptions. | `src/kmmainwidget.cpp` (`ManageLocalSubscriptions`, ~line 1148) | external: kdepim-runtime IMAP resource, Akonadi `StandardActionManager` |
| Account order | Reorder accounts in the folder tree. | `src/configuredialog/configureaccountpage.cpp` | external: `MailCommon::AccountConfigOrderDialog` |
| Sending accounts (SMTP, sendmail, EWS) | Create, configure, and pick a default outgoing transport. Supports authentication and auto-detecting server settings. | `src/configuredialog/configureaccountpage.cpp` (`AccountsPageSendingTab`) | external: `MailTransport` (kmailtransport: `TransportManagementWidgetNg`, `TransportManager`) |
| Outbox and send policy | Send-now or send-later default; flush the outbox never, on manual checks, or on all checks. | `src/configuredialog/configureaccountpage.cpp`; kcfg `SendOnCheck` | external: `MailTransport`, `akonadi_maildispatcher_agent` (kdepim-runtime) |
| Check mail / Check mail in | Fetch from all accounts or from one account. | `src/kmmainwidget.cpp` actions `check_mail`, `check_mail_in`; `src/widgets/kactionmenuaccount.cpp` | external: Akonadi resources |
| Unified mailboxes | Virtual Inbox, Sent, and Drafts that merge every account, plus user-defined unified boxes. Off by default. | `agents/unifiedmailboxagent/*` | depends on Akonadi agent framework |
| Plasma Activities scoping | Show accounts, identities, transports, and LDAP servers per Plasma Activity. | `src/activities/*`; kcfg `PlasmaActivitySupport` | external: `PimCommonActivities`, Plasma Activities |
| Import from other clients | Import wizard (other mail clients), plus manual import of messages and mbox files. | `src/kmlaunchexternalcomponent.cpp` (`slotImportWizard`, `slotImport`); `src/scripts/mh2kmail` | external app: `akonadiimportwizard` (akonadi-import-wizard) |
| Backup/restore settings and local mail | Export and restore KMail data and settings. | `src/kmlaunchexternalcomponent.cpp` (`slotExportData`) | external app: `pimdataexporter` (pim-data-exporter) |

## Folders

| Feature | Description | Source | External |
|---|---|---|---|
| Folder tree | Folder tree in long or short layout, with a folder quick-search field and tooltips (total, unread, size). | `src/kmmainwidget.cpp`, `src/configuredialog/configureappearancepage.cpp` (Layout tab); kcfg `EnableFolderQuickSearch`, `ToolTipDisplayPolicy` | external: `MailCommon::FolderTreeWidget`/`FolderTreeView` |
| Special folders | Local Folders (Maildir) with inbox, outbox, sent-mail, trash, drafts, and templates; per-identity sent/drafts/templates. | handbook `using-kmail.docbook#folders` | external: Akonadi-Mime `SpecialMailCollections`, kdepim-runtime Maildir resource |
| Favorite folders | Pinned folder pane, shown as icons or a list, reorderable. | `src/kmmainwidget.cpp` (`add_favorite_folder`) | external: `MailCommon::FavoriteCollectionWidget`, Akonadi `FavoriteCollectionsModel` |
| Folder CRUD and drag-and-drop | Add, rename, delete, copy, cut, and paste folders; move or copy prompt on drop. | `src/kmmainwidget.cpp`, `src/job/removecollectionjob.cpp`; kcfg `EnableFolderDnD`, `ShowPopupAfterDnD` | external: Akonadi `StandardMailActionManager` |
| Folder properties: General | Name, notify on new mail, keep replies in folder, hide from folder selection, sender identity, content type, shared unread state. | registered in `src/kmmainwidget.cpp:285` | external: `MailCommon::CollectionGeneralPage` |
| Folder properties: View | Custom icons, sender/receiver column, per-folder aggregation and theme, default HTML/plain format. | `src/collectionpage/collectionviewpage.cpp` | external: `MessageList` (aggregation and theme combos) |
| Folder properties: Retrieval | Sync interval, full messages or bodies on demand, local cache retention. | (Akonadi's built-in page) | external: Akonadi `CollectionPropertiesDialog` cache-policy page |
| Folder properties: Templates | Per-folder reply, forward, and new-message templates. | `src/collectionpage/collectiontemplatespage.cpp` | external: `TemplateParser` |
| Folder properties: Expiry | Auto-expire read or unread mail after N days, by moving or deleting. | registered in `src/kmmainwidget.cpp:289`; kcfg `AutoExpiring`, `ExcludeImportantMailFromExpiry` | external: `MailCommon::CollectionExpiryPage` |
| Folder properties: Mailing list | Mark a folder as a list; detect or edit List-Post, Subscribe, Unsubscribe, Archive, and Help; invoke the handler. | `src/collectionpage/collectionmailinglistpage.cpp` | external: `MessageCore::MailingList` |
| Folder properties: Shortcut | Keyboard shortcut that jumps to the folder. | `src/collectionpage/collectionshortcutpage.cpp`, `src/foldershortcutactionmanager.cpp` | KMail-only |
| Folder properties: Maintenance | Type, size, and counts; toggle full-text indexing. | `src/kmmainwidget.cpp:286` | external: Akonadi `CollectionMaintenancePage` |
| Folder properties: Quota (IMAP) | Shows quota usage. Colors folders near the quota threshold. | `src/collectionpage/collectionquotapage.cpp`, `collectionquotawidget.cpp`; kcfg `CloseToQuotaThreshold` | external: Akonadi `CollectionQuotaAttribute` |
| Folder properties: Access Control (IMAP ACL) | Per-user rights, simplified to None, Read, Append, Write, or All. | `src/kmmainwidget.cpp:284` | external: `PimCommon::CollectionAclPage` (pimcommon) |
| Folder actions | Mark all as read (optionally recursive), move all to trash, expire, empty all trash, remove duplicates (optionally recursive), clear the Akonadi cache, update folder. | `src/kmmainwidget.cpp`, `src/job/markallmessagesasreadinfolderandsubfolderjob.cpp`, `src/job/removeduplicatemessageinfolderandsubfolderjob.cpp`, `src/job/clearcachejobinfolderandsubfolderjob.cpp` | external: Akonadi-Mime `RemoveDuplicatesJob`, `MarkAsCommand`; Akonadi `ClearCacheFoldersJob` |
| Jump to folder / folder switcher | "Jump to Folder" dialog; Ctrl+Tab-style history of recently used folders. | `src/kmmainwidget.cpp` (`jump_to_folder`), `src/historyswitchfolder/*` | external: `MailCommon::FolderSelectionDialog` |
| Archive folder (one-off) | Export a folder tree to zip, tar, tar.bz2, or tar.gz, optionally deleting it afterwards. | `src/dialog/archivefolderdialog.cpp` | external: `MailCommon::BackupJob` |
| Automatic archiving | Scheduled archive jobs per folder, with a retention count. | `agents/archivemailagent/*`; manual `doc/akonadi_archivemail_agent/` | runs as an Akonadi agent |
| Per-message archive ("Folder Archive") | One-click "Archive" moves a message into a per-account archive folder: a single folder, or one per month or per year. | `src/folderarchive/*`; action `archive_message` in `src/messageactions.cpp` | KMail-only (uses Akonadi) |
| Undo/redo moves | Undo stack for message moves. | `src/kmundoredomanager.{h,cpp}` | KMail-only |
| Startup folder, empty trash on exit | Open a chosen folder at startup; empty the local trash on quit. | kcfg `StartSpecificFolderAtStartup`, `StartupFolder`, `EmptyTrashOnExit` | KMail-only |

## Message list

| Feature | Description | Source | External |
|---|---|---|---|
| Message list view | Threaded or flat list with multi-select and sort columns. | `src/kmmainwidget.cpp`, `src/widgets/collectionpane.cpp` | external: `MessageList::Pane`/`View`/`StorageModel` (messagelib) |
| Aggregations | Eight presets (Activity by Date flat or threaded, Current Activity flat or threaded, Flat Date, Senders/Receivers, Standard Mailing List, Thread Starters). Users can define their own grouping, threading, and expand policies. | `src/configuredialog/configureappearancepage.cpp` (Message List tab); handbook `configure.docbook#configure-appearance-message-list` | external: `MessageList` (`AggregationComboBox`, `AggregationConfigButton`) |
| Themes | Classic, Smart, and Smart with Clickable Status, plus custom column/content themes. | same | external: `MessageList` (`ThemeComboBox`, `ThemeConfigButton`) |
| Per-folder view overrides | Aggregation and theme per folder. | `src/collectionpage/collectionviewpage.cpp` | external: `MessageList` |
| Tabs | Open folders in message-list tabs. | `src/kmmainwidget.cpp` (`slotCreateNewTab`), `src/widgets/collectionpane.cpp` | external: `MessageList::Pane` |
| Quick search / filter bar | Text search over subject, from, body, and so on, plus status and tag filters. | `src/kmmainwidget.cpp` (`focus_to_quickseach`) | external: `MessageList` |
| Thread operations | Expand or collapse a thread or all threads; mark a thread read, unread, important, or action item; watch or ignore a thread; move a thread to trash. | `src/kmmainwidget.cpp` (`thread_*`, `expand_*`, `collapse_*`, `move_thread_to_trash`) | external: Akonadi-Mime `MessageStatus` |
| Navigation | Next/previous message, unread message, or unread folder; "next unread text" (Space); configurable loop behavior and what to select on folder entry. | `src/kmmainwidget.cpp` (`go_*`); `src/configuredialog/configuremiscpage.cpp`; kcfg `LoopOnGotoUnread`, `ActionEnterFolder` | KMail-only |
| Date display formats | Standard, localized, smart, or custom. | `src/configuredialog/configureappearancepage.cpp` | external: `MessageCore::DateFormatter` |
| Snooze | Hide messages until a chosen time, then bring them back. | `src/snooze/*` (menu, date dialog, `SnoozeFilterProxyModel`), `agents/snoozeagent/*` | external: `MailCommon::SnoozeAttribute`; Akonadi agent |
| Fonts per status | Separate fonts for unread, important, and action-item rows. | `src/configuredialog/configureappearancepage.cpp` (Fonts tab) | KMail-only |
| Mark-as-read delay | Mark the selected message read after N seconds. | `src/configuredialog/configuremiscpage.cpp` | external: `MessageViewer::MarkMessageReadHandler` |

## Reading

| Feature | Description | Source | External |
|---|---|---|---|
| Preview pane and separate reader window | Preview pane below or beside the list, or hidden; a standalone reader window. | `src/kmreaderwin.cpp`, `src/kmreadermainwin.cpp`; kcfg `readerWindowMode` | external: `MessageViewer::Viewer` (messagelib) on QtWebEngine (`WebEngineViewer`) |
| MIME rendering | Multipart, HTML, plain text, inline images, and attachment parsing. | (consumer only) | external: `MimeTreeParser`, `MessageViewer`, KF6 `KMime` |
| HTML vs. plain text | Global "Prefer HTML". Per folder and per contact ("show HTML when mail comes from this contact"). HTML status bar. | `src/kmreaderwin.cpp`, `src/kmmainwidget.cpp` (`display_format_message`), `src/widgets/displaymessageformatactionmenu.cpp` | external: `MessageViewer` |
| External references (remote content) | Blocked by default. Load per message, per folder, or per contact; per-domain allow list. | `src/kmmainwidget.cpp` (`prefer_html_external_refs`) | external: `MessageViewer::RemoteContentConfigureDialog`/`RemoteContentMenu` |
| Header styles | Brief, Fancy, Long, Standards, and Grantlee themes; downloadable themes. | `src/configuredialog/configurepluginpage.cpp` | Plugin: kdepim-addons header-style plugins via `MessageViewer::HeaderStylePlugin` |
| Attachment display | Show attachments as icons, smart, inline, hidden, or in the header only. Save attachments, delete attachments, open with an external app. | `src/kmcommands.h` (`KMSaveAttachmentsCommand`, `KMDeleteAttachmentsCommand`), `src/kmmainwidget.cpp` (`file_save_attachments`, `delete_all_attachments`) | external: `MessageViewer` |
| View source / message structure | Raw source viewer and MIME tree viewer (Ctrl+Alt+D). | handbook `menus.docbook` | external: `MessageViewer` |
| Find in message, zoom, fixed font | In-message search, zoom, fixed-width font toggle, emoticon toggle. | `src/kmreaderwin.cpp` (`find_in_messages`), `src/widgets/zoomlabelwidget.cpp` | external: `MessageViewer`, `WebEngineViewer` |
| Quote handling | Quote colors per level, collapsible quotes, smaller font for quotes. | `src/configuredialog/configureappearancepage.cpp` (General and Colors tabs) | external: `MessageViewer::MessageViewerSettings`/`CSSHelper` |
| Character encoding override | Fallback and override encodings. | handbook `configure.docbook#configure-appearance-general` | external: `MessageViewer` |
| Print, print preview, export to PDF | Print a message or export it to PDF, with print options (selected text only, backgrounds, crypto details). | `src/kmcommands.h` (`KMPrintCommand`), `src/messageactions.cpp` (`file_export_pdf`), `src/configuredialog/configuremiscpage.cpp` (Printing tab) | external: `MessageViewer::PrintingSettings` |
| Save as / open .eml and .mbox | Save messages to files; open a message file. | `src/kmcommands.h` (`KMSaveMsgCommand`, `KMOpenMsgCommand`), `src/job/saveasfilejob.cpp` | KMail-only |
| Link and image context actions | Compose to, reply to, or forward to an address; add to address book; bookmark a link; save or open or share an image. | `src/kmreaderwin.cpp` | external: `PimCommon` purpose menu; Akonadi contacts |
| Restore closed viewer | Reopen recently closed reader windows. | `src/historyclosedreader/*`; actions `restore_closed_messageviewer` | KMail-only |
| Mailing-list menu | Post, help, subscribe, unsubscribe, and archive links taken from List-* headers. | `src/messageactions.cpp` (`mailing_list`) | external: `MessageCore::MailingList` |
| Calendar invitations | Render iMIP invitations inline with accept/decline. Settings under Misc → Invitations. | `src/configuredialog/configuremiscpage.cpp` (`MiscPageInviteTab`) | external: `MessageViewer::InvitationSettings`; kdepim-addons body-part formatter + akonadi-calendar |
| TNEF (winmail.dat) | Standalone TNEF viewer. | `ktnef/` | external: KTNEF library |
| Viewer plugins | Create event, todo, or note from a mail; translator; expand short URL; external script; Gravatar; DKIM. | `src/configuredialog/configurepluginpage.cpp`; handbook `configure.docbook#configure-plugins-misc` | Plugin: kdepim-addons via `MessageViewer::ViewerPluginManager` |
| Text-to-speech on messages | Read the message aloud. | `src/configuredialog/configureaccessibilitypage.cpp`; link `KF6::TextEditTextToSpeech` | external: ktextaddons |

## Composing

| Feature | Description | Source | External |
|---|---|---|---|
| Composer window | Separate window with identity, from, to, cc, bcc, reply-to, FCC, transport, dictionary, and subject. Users choose which fields are visible. | `src/editor/kmcomposerwin.cpp` (`show_identity`, `show_transport`, `show_fcc`, `show_dictionary`, `show_all_fields`), `src/editor/composer.cpp` | external: `MessageComposer::ComposerViewBase` (messagelib) |
| Recipients editor and autocompletion | Multi-line recipients editor; completion from contacts, recent addresses, LDAP, and addresses indexed from mail; "Select Recipients" dialog. | `src/editor/kmcomposerwin.cpp` (`select_recipients`) | external: `MessageComposer::RecipientsEditor`, `PimCommon::AddresseeLineEdit`, `RecentAddresses`, akonadi-contacts, akonadi-search, KLDAP |
| Reply family | Reply, reply to all, reply to author, reply to list, reply without quote, reply with a custom template, quote only the selection, smart quoting. | `src/messageactions.cpp`, `src/kmcommands.h` (`KMReplyCommand`), `src/job/createreplymessagejob.cpp` | external: `MessageComposer::MessageFactoryNG`, `TemplateParser` |
| Forward family | Forward inline or as attachment, redirect (bounce), forward with a custom template. | `src/messageactions.cpp`, `src/kmcommands.h` (`KMForwardCommand`, `KMForwardAttachedCommand`, `KMRedirectCommand`), `src/job/createforwardmessagejob.cpp`; kcfg `ForwardingInlineByDefault` | external: `MessageComposer`, `MailCommon::RedirectDialog` |
| Edit as new / send again / use template | Reopen a sent or draft message; create a new message from a template. | `src/messageactions.cpp` (`editasnew`, `send_again`, `use_template`), `src/kmcommands.h` (`KMEditMessageCommand`, `KMResendMessageCommand`, `KMUseTemplateCommand`) | KMail-only |
| Plain-text and HTML (rich text) editing | HTML toggle, rich-text toolbar, inline images, "reply in the original format". | `src/editor/kmcomposereditorng.cpp`, `src/editor/kmcomposerwin.cpp` (`html`); kcfg `UseHtmlMarkup` | external: `MessageComposer::RichTextComposerNg`, `KPIMTextEdit` |
| External editor | Write the body in an external program. | kcfg `ExternalEditor`, `UseExternalEditor`; `src/editor/warningwidgets/externaleditorwarning.cpp` | external: `KPIMTextEdit::RichTextExternalComposer` |
| Templates | Global (new, reply, reply-all, forward, quote string), per identity, and per folder, plus custom templates with shortcuts and a substitution language. | `src/configuredialog/configurecomposerpage.cpp` (Standard and Custom Templates tabs), `src/collectionpage/collectiontemplatespage.cpp`, `src/editor/kmcomposerupdatetemplatejob.cpp`; kcfg group `GlobalTemplates` | external: `TemplateParser` (messagelib) |
| Signatures | Per-identity signature: text, file, or command output; plain or HTML. Append, prepend, or insert at the cursor; separator; strip the signature on reply. | `src/editor/kmcomposerwin.cpp` (`append_signature`, `prepend_signature`, `insert_signature_at_cursor_position`), `src/configuredialog/configurecomposerpage.cpp` | external: `KIdentityManagementCore::Signature`, `MessageComposer::SignatureController` |
| Attachments | Attach a file or directory, clipboard text, own vCard, vCards, or a public key. Attachment properties include "suggest inline". Paste as attachment. | `src/editor/attachment/attachmentcontroller.cpp`, `attachmentview.cpp`; handbook `menus.docbook#composer-attach-menu` | external: `MessageComposer::AttachmentControllerBase`/`AttachmentModel` |
| Forgotten-attachment warning | Warn when the body mentions attachment keywords but nothing is attached. | `src/editor/warningwidgets/attachmentmissingwarning.cpp`; kcfg `ShowForgottenAttachmentWarning`, `AttachmentKeywords` | KMail-only |
| Auto-resize images | Shrink attached images according to rules (size, format, recipients, filename). | `src/configuredialog/configurecomposerpage.cpp` (`ComposerPageAutoImageResizeTab`) | external: `MessageComposer::ImageScalingWidget` |
| Spell check | As-you-type and on-demand checking, per-identity dictionary, check before send. | `src/editor/kmcomposerwin.cpp` (`options_auto_spellchecking`, `setup_spellchecker`), `src/dialog/spellcheckerconfigdialog.cpp`; kcfg `AutoSpellChecking`, `CheckSpellingBeforeSend` | external: KF6 `Sonnet` |
| Autocorrection | Typographic quotes, replacement lists, exceptions. | `src/configuredialog/configurecomposerpage.cpp` (`ComposerPageAutoCorrectionTab`), `src/editor/subjectlineeditwithautocorrection.cpp` | external: ktextaddons `TextAutoCorrection*` |
| Text snippets | Snippet manager with variables, inserted from a side panel. | `src/editor/kmcomposerwin.cpp` (`snippets`); kcfg `ShowSnippetManager` | external: `MailCommon::SnippetsManager`, `MessageComposer::ConvertSnippetVariablesJob` |
| Editing helpers | Paste as quotation, add or remove quote characters, clean spaces, change case, insert special character, insert email, zoom, word wrap, fixed font. | `src/editor/kmcomposerwin.cpp` (`paste_att`, `clean_spaces`, `wordwrap`, `toggle_fixedfont`) | Plugin: kdepim-addons composer plugins (change case, insert email, special char, short URL, quick text, share text, zoom) |
| Grammar check / translator | LanguageTool and Grammalecte grammar checks; translator sidebar. | `src/editor/plugininterface/kmailplugingrammareditormanagerinterface.cpp` | Plugin: kdepim-addons; ktextaddons `TextTranslator` |
| Speech-to-text dictation | Dictate message text. | `src/configuredialog/configureaccessibilitypage.cpp`; kcfg `EnableSpeechToText` | external: ktextaddons `TextSpeechToText` |
| Drafts, autosave, templates folder | Save as draft, autosave at an interval, save as template, save as file, insert a text file. | `src/job/savedraftjob.cpp`, `src/editor/kmcomposerwin.cpp` (`save_in_drafts`, `save_in_templates`, `save_as_file`, `insert_file`); kcfg `AutosaveInterval` | external: `MessageComposer` |
| Send / send via / queue | Send now, send through a chosen transport, queue in the outbox, confirm before send, send-shortcut validation. | `src/editor/kmcomposerwin.cpp` (`send_default`, `send_default_via`, `send_alternative`), `src/editor/validatesendmailshortcut.cpp`; kcfg `ConfirmBeforeSend` | external: `MessageComposer::AkonadiSender`, `MailTransport` |
| Undo send | Hold the message for N seconds (10–50; default 10) with an undo button. | `src/undosend/*`; kcfg group `UndoSend` | KMail-only |
| Send later (scheduled send) | Schedule delivery for a later time, with optional recurrence. "Delayed Messages" lists scheduled mail. | `agents/sendlateragent/*`, `src/messageactions.cpp` (`message_delayed`); manual `doc/akonadi_sendlater_agent/` | external: `MessageComposer::SendLaterDialog`/`SendLaterInfo`; Akonadi agent |
| Follow-up reminder | Remind me if nobody replies by a given date; creates a to-do. | `agents/followupreminderagent/*`, `src/job/createfollowupreminderonexistingmessagejob.cpp`, composer `follow_up_mail` | external: `MessageComposer::FollowupReminder*`, KF6 `KCalendarCore` (to-do) |
| Message options | Urgent priority, request read receipt (MDN), request delivery status (DSN). | `src/editor/kmcomposerwin.cpp` (`urgent`, `options_request_mdn`, `options_request_delivery_confirmation`); kcfg `RequestMDN` | external: `MessageComposer` |
| Custom headers, Message-ID suffix, subject prefixes | Add or override MIME headers; recognized Re:/Fwd: prefixes; Outlook-compatible attachment naming; maximum attachment size. | `src/editor/custommimeheader/*`, `src/configuredialog/configurecomposerpage.cpp` (Headers, Subject, Attachments tabs) | external: `MessageComposer::MessageComposerSettings` |
| Distribution lists | Save the current recipients as a contact group; expand groups and aliases. | `src/editor/kmcomposerwin.cpp` (`save_distribution_list`) | external: akonadi-contacts (`ContactGroupExpandJob`), `MessageComposer::AliasesExpandJob` |
| Pre-send checks | Too many recipients; mismatch between identity and folder; recipient addresses that look like phishing; duplicate recipients; confirm address allow/deny lists; automatically add contacts. | `src/editor/warningwidgets/toomanyrecipientswarning.cpp`, `incorrectidentityfolderwarning.cpp`, `src/editor/potentialphishingemail/*`; kcfg `TooManyRecipients`, `RecipientThreshold` | Plugin: kdepim-addons "Check Before Send", "Confirm Address", "Automatic Add Contacts" via `MessageComposer::PluginEditorCheckBeforeSend*` |
| Mail-to / D-Bus compose | Open a composer from a `mailto:` link, the command line, or D-Bus; hidden composer for programmatic sends. | `src/kmkernel.h` (D-Bus `org.kde.kmail.kmail`), `src/job/opencomposerjob.cpp`, `opencomposerhiddenjob.cpp`, `src/kmail_options.h` | KMail-only |
| Mail merge | Agent scaffold only. `MailMergeManager` methods are TODO stubs, and no main-window action calls it. | `agents/mailmergeagent/*` | — (not a working feature) |

## Search

| Feature | Description | Source | External |
|---|---|---|---|
| Find Messages dialog | Multi-rule search across chosen folders, including subfolders. Results open as a saved virtual "search folder" that can be edited later. | `src/searchdialog/searchwindowdialog.cpp`, `kmailsearchpatternedit.cpp`, `kmsearchmessagemodel.cpp`, `src/ui/searchwindow.ui` | external: `MailCommon::SearchPattern`/`SearchPatternEdit`; Akonadi `SearchCreateJob`, `PersistentSearchAttribute`, `akonadi_search_resource` |
| Full-text index | Xapian index of mail and contacts. KMail warns about folders that are not indexed. | `src/search/checkindexingmanager.cpp`, `checkindexingjob.cpp`, `src/searchdialog/incompleteindexdialog.cpp`; kcfg `CheckCollectionsIndexing` | external: **akonadi-search** (`akonadi_indexing_agent`, `AkonadiSearchPIM`) |
| Quick search bar | Instant filter of the current list by text, status, or tag. | (see Message list) | external: `MessageList` |
| Folder name filter | Live filter of the folder tree. | kcfg `EnableFolderQuickSearch` | external: `MailCommon` |

## Filters

| Feature | Description | Source | External |
|---|---|---|---|
| Client-side filter engine | Ordered filters. Each can apply to incoming mail (all accounts or chosen ones), sent mail, before sending, or manual runs, and can stop processing on match. | `agents/mailfilteragent/*` (runs as `akonadi_mailfilter_agent`), `agents/mailfilteragent/filtermanager.cpp` | external: `MailCommon::MailFilter`/`FilterManager`/`FilterAction`/`SearchPattern` |
| Filter dialog | Create, reorder, copy, and rename filters; add a filter to the Apply Filter menu with a shortcut or toolbar icon. | `src/kmmainwidget.cpp` (`filter`, `create_filter`) | external: `MailCommon::KMFilterDialog` |
| Criteria | Complete message, body, any header, all recipients, size, age, status, tag, encrypted, any named header. Operators include contains, equals, and regex, among others. | handbook `using-kmail.docbook#filter-criteria` | external: `MailCommon::SearchPattern` |
| Actions | Move or copy to folder, set identity, transport, or Reply-To; mark as; add tag; fake MDN; forward or redirect; confirm delivery; execute command; pipe through; remove, add, or rewrite header; play sound; add to address book; delete; unset status; encrypt or decrypt. | handbook `using-kmail.docbook#filter-action` | external: `MailCommon::FilterAction*` |
| Quick filter creation | "Filter on Subject, From, To, CC, or Mailing List" from the selected message. | `src/kmmainwidget.cpp` (`subject_filter`, `from_filter`, `to_filter`, `cc_filter`), `src/kmcommands.h` (`KMMailingListFilterCommand`) | external: `MailCommon` |
| Apply filters manually | Apply one filter or all filters to the selection, a folder, or a folder recursively. | `src/kmmainwidget.cpp` (`apply_filters*`), `src/kmcommands.h` (`KMFilterActionCommand`, `KMMetaFilterActionCommand`) | external: `MailCommon` |
| Filter log | Log viewer with configurable detail level and size. | `agents/mailfilteragent/filterlogdialog.cpp`, `src/kmmainwidget.cpp` (`filter_log_viewer`) | external: `MailCommon::FilterLog` |
| Filter import/export | Import filters from other clients and export them. | `agents/mailfilteragent/filtermanager.cpp` | external: `MailCommon::FilterImporterExporter` |
| Anti-spam / anti-virus wizards | Generate filters that call external scanners (SpamAssassin, Bogofilter, ClamAV, and others) through "pipe through". There is no built-in classifier. | `src/avscripts/kmail_*.sh`; handbook `using-kmail.docbook#the-anti-spam-wizard` | Plugin: kdepim-addons "Antispam"/"Antivirus" |
| Spam/ham classification | "Classify as spam / not spam" message actions, driven by wizard-configured tools. | handbook `menus.docbook#reader-message-menu` | Plugin: kdepim-addons |
| **Sieve** (server-side filters) | Manage scripts on ManageSieve servers: script editor, debug, and server discovery through IMAP account settings. | `src/sieveimapinterface/kmailsieveimapinstanceinterface.cpp`, `kmsieveimappasswordprovider.cpp` (QtKeychain); `src/kmmainwidget.cpp` (`sieveFilters`, `tools_debug_sieve`) | external: **libksieve** (`KSieveUi::ManageSieveScriptsDialog`, `SieveDebugDialog`, `KSieveCore::SieveImapInstance`) |
| Out-of-office (vacation) | Edit the Sieve vacation script across accounts. An indicator appears when a vacation script is active. | `src/widgets/vacationscriptindicatorwidget.cpp`, `src/kmmainwidget.cpp` (`tools_edit_vacation`); handbook `configure.docbook#configure-non-gui-options` (`CheckOutOfOfficeOnStartup`) | external: libksieve (`KSieveUi::MultiImapVacationDialog`/`VacationManager`, `KSieveCore::MultiImapVacationManager`) |

## Tags and flags

| Feature | Description | Source | External |
|---|---|---|---|
| Message tags | User tags with text color, background, font, icon, shortcut, and optional toolbar button. Applied from the menu, toolbar, or a dialog. | `src/tag/tagactionmanager.cpp`, `tagselectdialog.cpp`, `tagmonitormanager.cpp`; `src/configuredialog/configureappearancepage.cpp` (`AppearancePageMessageTagTab`); `src/kmcommands.h` (`KMSetTagCommand`) | external: Akonadi `Tag`/`TagAttribute`; `MailCommon::Tag`/`TagWidget`/`AddTagDialog` |
| Status flags | Read/unread, important, action item (to-do), watched/ignored thread, replied/forwarded, spam/ham, and others. | `src/kmcommands.h` (`KMSetStatusCommand`), `src/messageactions.cpp` (`set_status`) | external: Akonadi-Mime `MessageStatus`/`MessageFlags` |
| Local flags option | Allow local-only flags on read-only folders. | kcfg `AllowLocalFlags` | KMail-only |
| Create to-do from mail | Turn a message into a to-do (task attribute). | `src/job/createtaskjob.cpp`, `src/attributes/taskattribute.cpp` | external: KF6 `KCalendarCore` |
| Notes on messages | "Add Note…" attaches a note to a message. | handbook `menus.docbook#reader-message-menu` | Plugin (kdepim-addons "Create Note") |

## Identities

| Feature | Description | Source | External |
|---|---|---|---|
| Identity management | Create (empty, from system settings, or duplicate), rename, set default, remove. | `src/identity/identityngpage.cpp`, `identitytreengwidget.cpp`, `newidentitydialog.cpp`, `src/ui/identityngpage.ui` | external: `KIdentityManagementCore` (`IdentityManager`, `IdentityTreeModel`) |
| General tab | Name, organization, email address, aliases. | `src/identity/identitydialog.cpp` | external: `KIdentityManagementCore::Identity` |
| Cryptography tab | OpenPGP signing and encryption keys, S/MIME certificates, preferred format, auto-sign, auto-encrypt, Autocrypt toggle. | `src/identity/identitydialog.cpp` (~line 565, Autocrypt) | external: `Libkleo` (`KeySelectionCombo`, `KeyRequester`) |
| Advanced tab | Reply-To, CC, BCC, dictionary, sent/drafts/templates folders, outgoing transport, attach my vCard, autocorrection language, default domain, spam folder expiry. | `src/identity/identitydialog.cpp`, `identityfolderrequester.cpp`, `identityaddvcarddialog.cpp`, `identityeditvcarddialog.cpp`, `identityexpirespamfolderdialog.cpp` | external: `MailTransport::TransportComboBox`, `MailCommon::FolderRequester` |
| Templates tab | Per-identity templates. | `src/identity/identitydialog.cpp` | external: `TemplateParser::TemplatesConfiguration` |
| Signature tab | Signature editor. | `src/identity/identitydialog.cpp` | external: `KIdentityManagementWidgets` |
| Picture tab | X-Face or Face header (from a file, the address book, or raw input). | `src/identity/xfaceconfigurator.cpp`, `encodedimagepicker.cpp`, `src/ui/xfaceconfigurator.ui` | external: `MessageViewer::KXFace` |
| Activities tab | Limit an identity to Plasma Activities. | `src/identity/identitydialog.cpp`, `src/activities/identityactivities.cpp` | external: `PimCommonActivities` |
| Identity selection | Folder default identity; identity chosen from recipients on reply; warning on mismatch. | `src/editor/warningwidgets/incorrectidentityfolderwarning.cpp` | external: `MailCommon::CollectionGeneralPage` |

## Address book

KMail has no address book of its own. Contacts live in Akonadi, and the full
editor is the separate KAddressBook app.

| Feature | Description | Source | External |
|---|---|---|---|
| Open address book | Launch the address book app. | `src/kmlaunchexternalcomponent.cpp` (`slotRunAddressBook`), `src/kmmainwidget.cpp` (`addressbook`) | external app: **kaddressbook** |
| New contact / add sender | New contact, add an address to a new or existing contact, edit a contact, open the contact in the address book. | `src/job/createnewcontactjob.cpp`, `addemailtoexistingcontactjob.cpp`, `src/dialog/addemailtoexistingcontactdialog.cpp`, `src/kmreaderwin.cpp` (`add_addr_book`, `add_to_existing_contact`, `edit_contact`) | external: akonadi-contacts (`ContactEditorDialog`, `AddEmailAddressJob`), KF6 `KContacts` |
| Recipient completion | Completion from Akonadi contacts, recent addresses, LDAP, and addresses indexed from mail. | `src/configuredialog/configurecomposerpage.cpp` (completion options); kcfg `CompletionMode` | external: `PimCommon::AddresseeLineEdit`, `RecentAddresses`, `CompletionConfigureDialog`; akonadi-search |
| LDAP directories | Configure LDAP servers for lookup and completion. | `src/configuredialog/configureaccountpage.cpp` (`LdapCompetionTab`), `src/activities/ldapactivities.cpp` | external: KLDAP (`KLDAPWidgets::LdapConfigureWidgetNg`, `KLDAPCore::LdapClientSearch`) |
| Per-contact display preferences | HTML and remote content allowed per sender, stored on the contact. | `src/kmreaderwin.cpp` | external: akonadi-contacts |
| Sender avatars | Gravatar images in headers. | `src/configuredialog/configurepluginpage.cpp` | external: libgravatar (`KPim6::Gravatar`) |
| Auto-add recipients | Add new recipients to an address book on send. | — | Plugin: kdepim-addons "Automatic Add Contacts" |

## Offline and sync

| Feature | Description | Source | External |
|---|---|---|---|
| Work offline / online | Global toggle that takes every network resource offline. Asks at startup if KMail was offline. | `src/kmkernel.cpp` (`setAccountStatus`, `isOffline`, ~lines 720–860), `src/kmmainwidget.cpp` (~line 2484); kcfg `NetworkState` | external: Akonadi `AgentInstance::setIsOnline`; `PimCommon::NetworkManager` (network detection) |
| Switch offline on shutdown | Per-account option. Defaults to on for POP3. | `src/kmkernel.cpp` (~line 1695) | external: Akonadi |
| Local cache / offline IMAP | Cache policy per folder: full messages or bodies on demand, retention, sync interval. "Download all messages for offline use" in the wizard. | (Retrieval tab, see Folders) | external: Akonadi cache policy; kdepim-runtime IMAP resource (disconnected mode) |
| Outbox queue | Messages sent while offline wait in the outbox. Send queued, send queued via a transport. | `src/kmmainwidget.cpp` (`send_queued`, `send_queued_via`), `src/widgets/kactionmenutransport.cpp` | external: `MailTransport`, `akonadi_maildispatcher_agent` (kdepim-runtime) |
| Background job control | D-Bus `pauseBackgroundJobs`, `stopNetworkJobs`, `resumeNetworkJobs`. | `src/kmkernel.h` | KMail-only (drives Akonadi) |
| Change notification | KMail reacts to store changes made by other clients and agents. | `src/kmkernel.cpp` | external: Akonadi `Monitor`/`ChangeRecorder`, `MailCommon::FolderCollectionMonitor` |
| New-mail notifications | Desktop notifications with per-folder opt-out, notification history, system-tray unread count, taskbar badge. | `src/kmsystemtray.cpp`, `src/unityservicemanager.cpp`, `src/dialog/kmknotify.cpp`, `src/kmlaunchexternalcomponent.cpp` (`slotShowNotificationHistory`), `src/dbusinterface/org.freedesktop.Akonadi.NewMailNotifier.xml`; kcfg `SystemTrayEnabled`, `ShowUnreadInTaskbar` | external: `akonadi_newmailnotifier_agent` (kdepim-runtime), KF6 Notifications/StatusNotifierItem |

## Security and privacy

| Feature | Description | Source | External |
|---|---|---|---|
| OpenPGP and S/MIME sign/encrypt | Toggle sign and encrypt per message, auto-sign or auto-encrypt when possible, show keys for approval, encrypt to self, crypto state indicator. | `src/editor/kmcomposerwin.cpp` (`sign_message`, `encrypt_message`, `options_select_crypto`), `src/editor/encryptionstate.cpp`, `src/editor/widgets/cryptostateindicatorwidget.cpp`, `src/configuredialog/configuresecuritypage.cpp` (`SecurityPageEncryptionTab`) | external: **libkleo** (`KeyResolverCore`, `KeyCache`), **gpgme/GnuPG** (`Gpgmepp`), `MessageComposer` crypto jobs |
| Autocrypt | Use Autocrypt and gossip keys for recipients; per-identity toggle. | `src/editor/kmcomposerwin.cpp` (~lines 3660–3790), `src/identity/identitydialog.cpp` | external: `MessageCore::AutocryptStorage` |
| Decrypt and verify on read | Optional automatic decryption, signature status display, automatic key/certificate import. "Copy decrypted" to a folder. | `src/configuredialog/configuresecuritypage.cpp` (Reading tab), `src/kmcommands.h` (`KMCopyDecryptedCommand`) | external: `MessageViewer`, `MimeTreeParser`, libkleo |
| Key and certificate warnings | Warn on unsigned or unencrypted sends, on recipients missing from the certificate, on expiring keys; near-expiry banner in the composer. | `src/editor/warningwidgets/nearexpirywarning.cpp`, `src/configuredialog/configuresecuritypage.cpp` | external: `Libkleo::ExpiryChecker` |
| Encrypted drafts / encrypted sent copies | Always encrypt drafts; store sent messages encrypted. | kcfg `AlwaysEncryptDrafts`, `CryptoStoreEncrypted` | external: `MessageComposer` |
| S/MIME validation | CRL and OCSP settings, HTTP/LDAP fetch policy, proxy. | `src/configuredialog/configuresecuritypage.cpp` (`SecurityPageSMimeTab`), `src/ui/smimeconfiguration.ui` | external: libkleo / gpgconf (GnuPG) |
| Certificate manager | Launch the certificate manager. | `src/kmlaunchexternalcomponent.cpp` (`slotStartCertManager`) | external app: **kleopatra** |
| Key generation in the wizard | Create or publish an OpenPGP key during account setup. | handbook `getting-started.docbook#account-wizard` | external app: accountwizard; `Libkleo::OpenPGPCertificateCreationDialog` |
| HTML and remote content safety | Plain text by default; external references blocked; HTML status bar. | `src/configuredialog/configuresecuritypage.cpp` (`SecurityPageGeneralTab`) | external: `MessageViewer` |
| Scam / phishing detection | Scam heuristics in the viewer; Google Safe Browsing URL check; tracking-URL scan. | handbook `configure.docbook#configure-security-reading` | external: `MessageViewer`, `WebEngineViewer::CheckPhishingUrlCache` |
| Phishing recipient warning | Warns when a recipient's display name hides a different address. | `src/editor/potentialphishingemail/*` | KMail-only |
| DKIM verification | Verify DKIM signatures and show their status. | `src/kmmainwidget.cpp`, `src/kmreaderwin.cpp` (`MessageViewer::DKIMViewerMenu`/`DKIMWidgetInfo`) | external: `MessageViewer` DKIM; Plugin: kdepim-addons "DKIM Checker" |
| Read receipts (MDN) policy | Ignore, ask, deny, or always send; quote nothing, the full message, or headers only; never answer encrypted mail. | `src/configuredialog/configuresecuritypage.cpp` (`SecurityPageMDNTab`), `src/ui/securitypagemdntab.ui` | external: `MessageViewer::MDNWarningWidget`, `MailCommon::MDNWarningJob`, KMime `MDN` |
| Ad blocking / Do Not Track in the viewer | Block remote trackers in HTML mail. | `src/configuredialog/configurepluginpage.cpp` | Plugin: kdepim-addons (WebEngineViewer URL interceptors) |
| Credential storage | Account and Sieve passwords live in the system keychain (KWallet/Secret Service). | `src/sieveimapinterface/kmsieveimappasswordprovider.cpp`; link `qt6keychain` | external: QtKeychain; resources in kdepim-runtime store their own secrets |
| Anti-virus scripts | Shell wrappers for ClamAV, F-Prot, Sophos, and AntiVir that the anti-virus wizard calls. | `src/avscripts/*` | Plugin: kdepim-addons "Antivirus" |

## Settings and customization

| Feature | Description | Source | External |
|---|---|---|---|
| Configure dialog | Seven pages, each also available as a KCM: Accounts, Appearance, Composer, Security, Misc, Plugins, Accessibility. | `src/configuredialog/configuredialog.cpp`, `kcm_kmail_*.cpp`, `src/kmail_config_*.json` | KF6 KCMUtils |
| Accounts page | Identities, Receiving, Sending, and LDAP tabs. | `src/configuredialog/configureaccountpage.cpp` | see Accounts |
| Appearance page | General (tray, quotes, encodings), Fonts, Colors, Layout (folder list long or short, preview position, favorites), Message List, Message Tags. | `src/configuredialog/configureappearancepage.cpp` | `MessageList`, `MessageViewer` |
| Composer page | General, Standard Templates, Custom Templates, Subject, Headers, Attachments, Autocorrection, Auto Resize Image. | `src/configuredialog/configurecomposerpage.cpp` | `TemplateParser`, `MessageComposer`, ktextaddons |
| Security page | Reading, MDN, Encryption, S/MIME Validation. | `src/configuredialog/configuresecuritypage.cpp`, `src/ui/securitypage*.ui` | libkleo, `MessageViewer` |
| Misc page | Folders (behavior), Invitations, Printing, User Feedback. | `src/configuredialog/configuremiscpage.cpp`, `src/ui/miscpagemaintab.ui` | `MessageViewer` |
| Plugins page | Enable and configure plugins: Akonadi agents, check-before-send, composer, grammar, header styles, viewer, and tools. | `src/configuredialog/configurepluginpage.cpp`, `src/configuredialog/configureplugins/configurepluginslistwidget.cpp`, `src/plugininterface/*`, `src/editor/plugininterface/*` | external: **kdepim-addons** plugins, `PimCommon::ConfigurePluginsWidget` |
| Accessibility page | Text-to-speech and speech-to-text settings. | `src/configuredialog/configureaccessibilitypage.cpp` | external: ktextaddons |
| Hidden options | `SendMDNsWithEmptySender`, `CloseDespiteSystemTray`, `CheckOutOfOfficeOnStartup`, `OutOfOfficeDomain`, and others. | handbook `configure.docbook#configure-non-gui-options` | — |
| Config schema | KMail's own keys. Most viewer, composer, and list settings live in each library's own kcfg. | `src/settings/kmail.kcfg`, `src/settings/kmailsettings.cpp` | messagelib/mailcommon kcfg files |
| Toolbars, shortcuts, menubar | Standard KXmlGui customization, color schemes, and show or hide the menubar for each window. | `src/kmmainwin.rc`, `src/kmcomposerui.rc`, `src/kmreadermainwin.rc`; kcfg `ShowMenuBar`, `ComposerShowMenuBar` | KF6 XmlGui |
| Reset settings | Assistant that cleans up stale settings. | `kmail-refresh-settings/` | — |

## Other

| Feature | Description | Source | External |
|---|---|---|---|
| Kontact integration | KMail as a Kontact part, plus a summary widget. | `src/kmail_part.cpp`, `src/kontactplugin/kmail/*`, `src/kontactplugin/summary/*` | external: KontactInterface, Kontact app |
| D-Bus API | `org.kde.kmail.kmail`: check mail, compose, select folder, list accounts, open the reader, and more (26 scriptable methods). | `src/kmkernel.h`, `src/mailserviceimpl.cpp` | KMail-only |
| Command line | `mailto:`, attachments, subject, body, check, view a message file. | `src/kmail_options.h`, `src/commandlineinfo.cpp` | KMail-only |
| What's New / usage telemetry | Release notes dialog; opt-in KUserFeedback data. | `src/kmmainwidget.cpp` (`whatsnew`), `src/userfeedback/*` | KUserFeedback, ktextaddons |
| Calendar invitations (sending side) | Options for invitation replies (Outlook-compatible, delete after reply, and others). | `src/configuredialog/configuremiscpage.cpp` | external: `MessageViewer::InvitationSettings`, akonadi-calendar |
| Share text / images | Share through the KDE Purpose framework. | `src/kmcommands.h` (`KMShareImageCommand`), `agents/mailfilteragent/mailfilterpurposemenuwidget.cpp` | external: `PimCommon::PurposeMenuWidget` |
| Expire Trash plugin, Folder Settings plugin, Expert plugin, Log Activities | Tool plugins. | `src/configuredialog/configurepluginpage.cpp` | Plugin: kdepim-addons |

---

## External components KMail depends on (mail scope)

| Component (repo) | What KMail uses it for |
|---|---|
| **Akonadi** (`akonadi`: AkonadiCore/Widgets/AgentBase) | All mail storage, caching, change notification, collections (folders), items (messages), tags, the agent/resource framework, persistent search collections, and folder property dialogs. The KMail tree has no storage layer of its own. |
| **kdepim-runtime** | Mail resources (IMAP, POP3, Maildir, mbox, EWS, Gmail through IMAP), `akonadi_maildispatcher_agent` (outbox), `akonadi_newmailnotifier_agent`. |
| **akonadi-mime** | Mail-specific Akonadi helpers: `MessageStatus`/`MessageFlags`, `SpecialMailCollections`, `StandardMailActionManager`, `RemoveDuplicatesJob`, `MarkAsCommand`, MDN and sent-behavior attributes. |
| **akonadi-search** | Xapian full-text indexing (`akonadi_indexing_agent`) and the address-completion index. |
| **akonadi-contacts** | Contact editor, contact search and selection, contact-group expansion. |
| **messagelib** | `MessageList` (list, threading, themes, quick search); `MessageViewer` (reader, header styles, remote content, MDN, DKIM, invitations UI); `MessageComposer` (composer core, crypto composition, send later, follow-up, plugins); `MessageCore`; `TemplateParser`; `MimeTreeParser`; `WebEngineViewer` (QtWebEngine view, phishing URL check). |
| **mailcommon** | Folder tree and favorites, the filter engine, dialog, and actions, search patterns, folder General and Expiry pages, tags UI, snippets, backup/archive job, POP3 settings bridge, redirect dialog. |
| **libksieve** | Sieve script management, editor, debugger, out-of-office/vacation. |
| **libkleo** + **gpgme/GnuPG** | OpenPGP and S/MIME key handling, key resolution, expiry checks, S/MIME validation config. |
| **kidentitymanagement** | Identities and signatures. |
| **kmailtransport** | SMTP, sendmail, and EWS transports and their management UI. |
| **kldap** | LDAP lookup and configuration. |
| **pimcommon** | IMAP ACL page, address line edit, recent addresses, network manager, plugin framework, Activities support. |
| **libkdepim**, **libgravatar**, **kpimtextedit**, **kontactinterface** | Progress UI, avatars, rich-text editor base, Kontact embedding. |
| **ktextaddons** | Autocorrection, translator, text-to-speech, speech-to-text, custom editors, What's New. |
| **kdepim-addons** | Every switchable plugin: header styles, check-before-send, confirm address, auto-add contacts, grammar, composer helpers, viewer tools (DKIM, Gravatar, translator, create event/todo/note), anti-spam and anti-virus wizards, ad block, invitation and other body-part formatters. |
| KF6: KMime, KContacts, KCalendarCore, Sonnet, KIO, XmlGui, Notifications, and others; QtKeychain | MIME parsing and composition, vCard, iCal to-dos, spell check, notifications, secret storage. |
| External apps launched | `accountwizard`, `kaddressbook`, `kleopatra`, `akonadiimportwizard`, `pimdataexporter`, Kontact. |

## Notes and caveats

- **The handbook lags behind the code in places.** It documents a Composer →
  Character Set tab, but `src/configuredialog/configurecomposerpage.cpp` no
  longer contains one: no "charset" matches, and the tabs are General,
  Standard Templates, Custom Templates, Subject, Headers, Attachments,
  Autocorrection, and Auto Resize Image. It also describes a "Configure
  Folder Archive Agent" menu entry, but per-message archiving is now
  in-process under `src/folderarchive/`. Where the two disagree, trust the
  code.
- **The mail merge agent is a stub.** `agents/mailmergeagent` is built, but
  `MailMergeManager` is all TODO and nothing in the main window calls it. Do
  not count it as a shipped feature.
- **Much of KMail's behavior lives outside this tree.** The tree is about
  front-end glue, Akonadi agents, and settings. The biggest feature areas sit
  in messagelib (list, viewer, composer) and mailcommon (filters, folder
  tree). For the parity matrix, the feature names above are reliable because
  both the handbook and the KMail-side wiring confirm them. Finer detail on
  those rows, such as the exact filter operators or the import formats,
  should be checked against those repos if it matters.
- **Spam handling is external by design.** The handbook says KMail "does not
  have a built-in spam detection solution". The wizards only generate filters
  that call external tools.
