# OAuth2 client registration for Gmail and Microsoft 365 mail access

Research for [dyle71/flock#19](https://github.com/dyle71/flock/issues/19), part of the wayfinder
map [dyle71/flock#1](https://github.com/dyle71/flock/issues/1). Raised in
[dyle71/flock#10](https://github.com/dyle71/flock/issues/10); builds on the "OAuth2 / XOAUTH2"
section of [`rust-mail-crates.md`](rust-mail-crates.md).

Researched 2026-11-11. Google and Microsoft facts come from their developer documentation and
help center pages, read on that date; the "last updated" date a page shows is given in Sources.
Thunderbird facts come from the local comm-central checkout
(`/home/dyle/Source/other/releases-comm-central`, commit `2669a12bd1`, 2026-11-11). KDE facts
come from the local KMail checkout (`/home/dyle/Source/other/kmail`, commit `98c911893`) and from
the `master` branches of `kdepim-runtime`, `kmailtransport` and `libkgapi` on KDE's GitHub mirror.
Statements marked _(inference)_ are my own reasoning from the cited facts. No source states them
directly.

This file names where other projects keep their client IDs and secrets, but does not reproduce
any of the values.

## Question

What does flock need to sign in to Gmail/Google Workspace and Microsoft 365/Outlook.com mail over
IMAP, SMTP and POP3 with OAuth2, as a GPL-3.0 open-source client, in Local mode (desktop app,
Electron SPA on `flock://app`, Rust sidecar does the network I/O) and in Server mode (self-hosted,
multi-user, operator-chosen HTTPS origin behind a reverse proxy)? The backend syncs in the
background, so it needs long-lived refresh tokens.

Specifically: Google's classification of `https://mail.google.com/`, app verification and the
annual security assessment (CASA), the limits of unverified and testing-mode apps, internal
Workspace apps, and which client type fits a desktop app versus a web server. Microsoft's public
and confidential clients, multi-tenant registration, publisher verification, the Exchange scopes,
and consent for personal versus work accounts. How Thunderbird and KMail ship their client IDs,
whether a fork may reuse them, and which registration models are realistic.

## Summary

- **Gmail over IMAP/SMTP/POP3 needs a restricted scope.** `https://mail.google.com/` is the only
  scope for those protocols and Google lists it as restricted, "includes any usage of IMAP,
  SMTP, and POP3 protocols" [GH-restricted]. Google's FAQ adds that IMAP use may be judged a
  minimum-scope violation, in which case it asks for a move to the Gmail API [GH-faq].
- **A public Google app with that scope needs restricted-scope verification and an annual
  CASA assessment** whenever it "accesses or has the capability to access Google user data from
  or through a server" [GI-restricted]. Google charges nothing; the assessor does, at a price
  agreed between developer and assessor [GH-faq].
- **Without verification, Google caps an app at 100 users.** An unverified app in production
  shows an "unverified app" screen and is limited to 100 new users [GH-unverified]. An app in
  *Testing* is limited to 100 listed test users and its refresh tokens expire after **7 days**
  [GH-audience][GI-oauth2]. An *Internal* app (Workspace org only) has neither the screen nor
  the cap, but works only for that organization's users [GH-exempt].
- **Google's API Terms say "Developer credentials may not be embedded in open source
  projects"** [G-terms], while Google's own installed-app docs say a desktop client's secret "is
  obviously not treated as a secret" [GI-oauth2]. Thunderbird and KDE both embed theirs.
- **Desktop app clients use a loopback redirect with PKCE.** Google's "Desktop app" type is a
  public client that uses `http://127.0.0.1:<port>`. Custom URI schemes "are no longer
  supported", so `flock://` cannot be a Google redirect target [GI-native]. Microsoft's
  "Mobile and desktop" platform takes `http://localhost` and ignores the port
  [MS-reply][MS-desktop].
- **A Server-mode instance is a web client at an origin nobody knows in advance.** Google wants
  an HTTPS redirect on a public-suffix domain [GI-webserver]. Microsoft allows at most 100
  redirect URIs for apps that also sign in personal accounts [MS-reply]. One project-registered
  web client cannot list every operator's origin _(inference)_.
- **Microsoft is the easier provider.** Delegated scopes `IMAP.AccessAsUser.All`,
  `POP.AccessAsUser.All`, `SMTP.Send` and `offline_access` cover Microsoft 365 and Outlook.com
  [MS-imap]. There is no assessment. Apps must be registered in a directory since June 2024
  [MS-breaking]. Publisher verification needs a Microsoft partner account and a non-
  `onmicrosoft.com` domain. Without it, users in tenants with risk-based step-up consent cannot
  consent to multitenant apps registered after November 2020 [MS-pubver]. Refresh tokens last 90
  days and renew on use; only SPA-type redirects get 24 hours [MS-refresh].
- **Thunderbird** hard-codes one client per provider in `OAuth2Providers.sys.mjs`, including a
  Google client secret, under the comment "Don't copy these values for your own application -
  register one for yourself!" [TB-providers]. Since 2025 it lets extensions register their own
  providers and, since August 2026, override built-in hostnames, for example to bring an
  organization's own Microsoft client [TB-ext].
- **KDE** hard-codes one Google client (ID and secret) and one Microsoft client ID in
  `kdepim-runtime` and `kmailtransport`. The same IDs serve IMAP and SMTP
  [KDE-imap-gmail][KDE-smtp-gmail][KDE-imap-outlook][KDE-smtp-outlook].

## 1. Google: the scope and its classification

- **The scope.** "The scope for IMAP, POP, and SMTP access is `https://mail.google.com/`"
  [GI-xoauth2]. Google's restricted-scopes list names it with the note "(includes any usage of
  IMAP, SMTP, and POP3 protocols)" [GH-restricted]. The Gmail scopes page describes it as "Read,
  compose, send, and permanently delete all your email from Gmail" and lists it under
  "Restricted scopes" [GI-scopes].
- **Minimum-scope policy.** The verification FAQ answers "What if my app is using IMAP or SMTP?
  Do I need to submit for verification?" with "Yes". It goes on: "If your usage of IMAP/SMTP is
  deemed to violate the minimum scope policy within the verification process, you will need to
  migrate to using the Gmail API." The scope "should only be requested if your application also
  needs to immediately and permanently delete threads and messages, bypassing Trash". An
  SMTP-only app must move to the Gmail API with `gmail.send` [GH-faq].
- **Approved use case.** The Gmail API policy lists "Built-in and web email clients that allow
  users to compose, send, read, and process email via a user interface" as the first approved use
  case for Gmail scopes [GI-gmailpolicy].
- **Device flow.** Not available for this scope (see [`rust-mail-crates.md`](rust-mail-crates.md),
  "OAuth2 / XOAUTH2"). Gmail needs the authorization-code flow.

## 2. Google: verification, CASA and the exceptions

### What a public app goes through

- **Brand verification**, then **restricted-scope verification**: verified domain, homepage and
  privacy policy on that domain, a YouTube demo video of the consent flow and every scope's use,
  and a written reason why narrower scopes are insufficient [GI-restricted][GH-requirements].
  Brand verification "typically takes 2-3 business days"; the restricted-scope process "can
  potentially take several weeks" [GI-restricted].
- **Security assessment (CASA).** "Every app that requests access to Google users' restricted data
  and has the ability to access data from or through a third-party server must go through a
  security assessment from Google-empanelled security assessors." Apps must be "reverified for
  compliance and complete a security assessment at least every 12 months after your assessor's
  Letter of Assessment (LOA) approval date" [GI-restricted]. The help center restates it as an
  annual assessment that ends in a "Letter of Validation" [GH-casa][GH-annual].
- **CASA** is built on OWASP ASVS, with assurance levels AL1 and AL2 assigned by "user count,
  requested scopes, and other application-specific signals" [GH-casa][ADA-casa]. The Tier 2
  self-scan process is marked deprecated in favor of a "lab verified CASA scan" [ADA-tier2].
- **Cost.** "Google does not charge the developer any fees for security assessment. … The cost
  for such a service is agreed on between the developer and the assessor" [GH-faq].
- **Data-handling rules for restricted scopes** include keeping "tokens such as OAuth access and
  refresh tokens, encrypted at rest" and keys "stored in a hardware security module or
  equivalent-strength key management system" [GI-gmailpolicy]. Both bear on the credential
  storage question of [dyle71/flock#10](https://github.com/dyle71/flock/issues/10).
- **Local-only apps.** The assessment trigger is phrased around server access ("from or through a
  server"). No page I read says outright that a client which keeps tokens and mail only on the
  user's device is exempt. Google's own verification team "will then determine the application
  type" when unsure [GI-restricted]. _(inference)_ Whether flock's Local mode alone would escape
  CASA is Google's call at review time.

### What an app can do without verification

| Mode | Who can authorize | Warning screen | Refresh token | Source |
| --- | --- | --- | --- | --- |
| External, *Testing* | Up to 100 test users listed in the console | Tester warning | **Expires in 7 days** for any scope beyond name, email and profile | [GH-audience][GI-oauth2] |
| External, *In production*, unverified | Any Google Account, **100 new users in total** | "Unverified app" screen | Normal lifetime | [GH-unverified][GH-audience] |
| *Internal* (Workspace or Cloud Identity org) | Users of the owning organization only | None | Normal lifetime | [GH-exempt][GH-audience] |
| External, verified | Any Google Account | None | Normal lifetime | [GI-restricted] |

- **Exceptions to verification** listed by Google: personal use ("fewer than 100 users"),
  development/testing/staging projects, service-owned data, internal use, and Workspace
  admin-trusted or Marketplace-installed apps [GH-exempt][GI-restricted]. The older help page
  also names "OAuth-based plugins … such as SMTP for WordPress" as not needing verification
  [GH-unverified].
- **Internal apps** require the Cloud project to be owned by the organization. For "high-risk
  Gmail and Drive scopes" the organization's admins may still have to allow internal apps to
  access restricted Workspace services [GH-audience][GH-exempt]. The User Data Policy says of the
  restricted-scope requirements: "If your app is only used by users within your own domain, then
  these requirements do not apply" [G-udp].
- **Workspace admins** can restrict API access to "only OAuth client IDs that are trusted by the
  organization". Google recommends publishing the client ID in help documents so admins can
  allowlist it [GI-workspace]. Accounts in Advanced Protection "may block most non-Google apps"
  [GH-audience].

## 3. Google: refresh tokens

- **Offline access** needs `access_type=offline` on the authorization request [GI-webserver].
- **A refresh token stops working when** the user revokes access, it "has not been used for six
  months", "the user changed passwords and the refresh token contains Gmail scopes", the
  per-account limit is exceeded, time-based access expires, or an admin restricts the service
  [GI-oauth2].
- **Testing mode:** "A Google Cloud Platform project with an OAuth consent screen configured for an
  external user type and a publishing status of 'Testing' is issued a refresh token expiring in 7
  days, unless the only OAuth scopes requested are a subset of name, email address, and user
  profile" [GI-oauth2].
- **Count limit:** "a limit of 100 refresh tokens per Google Account per OAuth 2.0 client ID. If the
  limit is reached, creating a new refresh token automatically invalidates the oldest refresh token
  without warning" [GI-oauth2].
- **Workspace controls.** Web session length settings for Gmail "typically [do] not invalidate
  OAuth refresh tokens granted to third-party applications". App access control can revoke them
  [GI-workspace].

## 4. Google: client types and redirect URIs

### Desktop app (installed application)

- **Type.** "Select the **Desktop app** application type." "Desktop apps are public clients and
  cannot keep secrets confidential." "Always use PKCE" [GI-native]. `code_challenge_method` takes
  `S256` (recommended) or `plain` [GI-native].
- **Client secret.** The console still issues one. In the token request it is marked "Optional"
  [GI-native]. The overview says the secret is embedded in the source code and "in this context,
  the client secret is obviously not treated as a secret" [GI-oauth2].
- **Redirect.** Loopback: `http://127.0.0.1:port` or `http://[::1]:port`, listening "on a random
  available port". `localhost` is allowed but "may cause issues with client firewalls". The
  loopback option is "DEPRECATED for Android, Chrome app and iOS OAuth client types", not for
  desktop [GI-native].
- **Custom schemes.** "Custom URI schemes are no longer supported due to the risk of app
  impersonation" [GI-native]. A `flock://` redirect is therefore not an option for Google
  _(inference from the quote)_.
- **Out-of-band copy/paste** "is no longer supported" [GI-native].
- **Embedded webviews** are disallowed by Google's OAuth policies (`disallowed_useragent`)
  [GI-native]. The sign-in has to open in the system browser, not inside Electron
  _(inference: an Electron `BrowserWindow` is an embedded user-agent)_.
- **Standard.** RFC 8252 §8.5: a secret "statically included as part of an app distributed to
  multiple users" is not confidential, and a server that still requires one "MUST treat the client
  as a public client" [RFC8252].

### Web application

- **Confidential.** The flow "is designed for applications that can securely store confidential
  information". The `client_secret.json` must be kept out of public locations and, for code shared
  "on GitHub", outside the source tree [GI-webserver].
- **Redirect URI rules.** "Redirect URIs must use the HTTPS scheme, not plain HTTP" (localhost
  exempt), hosts "cannot be raw IP addresses", and "Host TLDs … must belong to the public suffix
  list" [GI-webserver]. Every redirect URI has to be registered on the client; verification also
  requires verifying ownership of the authorized domains [GI-restricted].
- **Changes after verification.** "If you add any new redirect URLs or JavaScript origins, or if
  you change your product name after verification, you have to go through verification again"
  [GH-unverified].

## 5. Google: API Terms on credentials

Google APIs Terms of Service, Section 4b "Confidential Matters" (last modified 2021-11-09):
"Developer credentials (such as passwords, keys, and client IDs) are intended to be used by you
and identify your API Client. You will keep your credentials confidential and make reasonable
efforts to prevent and discourage other API Clients from using your credentials. Developer
credentials may not be embedded in open source projects." Section 3c adds "You will not
misrepresent or mask either your identity or your API Client's identity" [G-terms].

This conflicts with the installed-app guidance in §4, which tells developers to embed the ID and
secret in the app [GI-oauth2]. Both Thunderbird and KDE embed theirs in public source (§10, §11).

## 6. Microsoft: registration and client types

- **Public versus confidential.** Public clients "run on devices, such as desktop … They can't be
  trusted to safely keep application secrets … they can't have client secrets." Confidential
  clients "run on servers, such as web apps" and "can adequately hold configuration-time secrets"
  [MS-clients].
- **Token requests.** `client_secret` is "required for confidential web apps". "Don't use the
  application secret in a native app or single page app" [MS-authcode].
- **PKCE** "is now recommended for all application types, both public and confidential clients,
  and required … for single page apps"; `S256` and `plain` are supported [MS-authcode].
- **Tenant path.** `/common`, `/organizations`, `/consumers` or a tenant ID [MS-authcode].
- **Audiences.** Single tenant; "Accounts in any Microsoft Entra directory" (work and school);
  "Accounts in any Microsoft Entra directory and personal Microsoft accounts (such as Skype, Xbox,
  Outlook.com)"; or personal accounts only [MS-tenancy][MS-register]. Reaching both Microsoft 365
  and Outlook.com needs the third.
- **Registration needs a directory.** "Starting June 2024, all applications must be registered in a
  directory." A personal-account user can create one by joining the Microsoft 365 Developer Program
  or signing up for Azure [MS-breaking]. The quickstart lists "An Azure account that has an active
  subscription" ("Create an account for free") as a prerequisite [MS-register].
- **Desktop registration.** Platform "Mobile and desktop applications"; for system browsers "use
  the exact value: `http://localhost`"; for "Node.js Electron apps:
  `msal{Your_Application/Client_Id}://auth`". "Allow public client flows" is a separate switch
  for device code and similar flows [MS-desktop].

## 7. Microsoft: redirect URIs

- **Scheme.** "Redirect URIs must begin with the scheme https, with exceptions for some localhost
  redirect URIs." HTTP is "supported only for localhost URIs" [MS-reply].
- **Loopback ports.** "The port component … is ignored for the purposes of matching a localhost
  redirect URI." `[::1]` "isn't currently supported". Microsoft prefers `127.0.0.1` over
  `localhost`, but an `http://127.0.0.1` redirect can only be added through the application
  manifest, not the portal text box [MS-reply].
- **Limits.** At most 100 redirect URIs for apps that sign in personal and work accounts, 256
  characters each. Wildcards are "unsupported in app registrations configured to sign in personal
  Microsoft accounts" [MS-reply].

## 8. Microsoft: scopes, consent and verification

- **Scopes.** `https://outlook.office.com/IMAP.AccessAsUser.All`, `…/POP.AccessAsUser.All` and
  `…/SMTP.Send`; "In addition, you can request for offline_access scope … your app can receive
  refresh tokens". OAuth for these protocols "is available for both Microsoft 365 … and
  Outlook.com users" [MS-imap].
- **Admin consent** is described only for the app-only path (client credentials with
  `IMAP.AccessAsApp` and friends), which also needs a service principal registered in Exchange
  through PowerShell [MS-imap]. The delegated scopes above are what Thunderbird and KDE request
  for interactive sign-in [TB-providers][KDE-imap-outlook].
- **Tenant user-consent policy.** "By default, all users are allowed to consent to applications
  for permissions that don't require administrator consent. For example, by default, a user can
  consent to allow an app to access their mailbox." Microsoft recommends that admins allow user
  consent "only for applications that have been published by a verified publisher". The built-in
  policy `microsoft-user-default-low` does that [MS-consent].
- **Publisher verification.** It needs a Microsoft AI Cloud Partner Program account that has
  passed verification, an app registered with a work or school account in an Entra tenant, and a
  publisher domain that is not `*.onmicrosoft.com`. "No charges are associated" with it
  [MS-pubver]. "Beginning November 2020, if risk-based step-up consent is enabled, users can't
  consent to most newly registered multitenant apps that aren't publisher verified." This applies
  to apps registered after 2020-11-08 that request more than basic sign-in permissions "from users
  in tenants that aren't the tenant where the app is registered" [MS-pubver].
- **Exchange Online switches.** SMTP AUTH can be off tenant-wide or per mailbox, and "If security
  defaults is enabled in your organization, SMTP AUTH is already disabled in Exchange Online"
  [MS-smtpauth]. A valid token does not help when the protocol is disabled for the mailbox
  _(inference)_.

## 9. Microsoft: refresh tokens

"The default lifetime for the refresh tokens are as follows: 24 hours for single-page
applications. … 90 days for all other scenarios. Refresh tokens replace themselves with a fresh
token upon every use." Tokens sent to a redirect URI of type `spa` expire after 24 hours, and so
do the refresh tokens that descend from them [MS-refresh]. A password change by the user revokes
"password-based" tokens; a confidential client's token "stays alive" in that case [MS-refresh].

_(inference)_ A backend that refreshes at least every 90 days keeps a Microsoft grant alive
indefinitely, as long as no admin or user revokes it. The token exchange must happen in the
sidecar or server, not in the SPA, or the 24-hour SPA rule applies.

## 10. Thunderbird

All in `mailnews/base/src/` unless noted.

- **Where the IDs live.** `OAuth2Providers.sys.mjs` hard-codes a `kIssuers` map (line 240) with
  one `clientId` per issuer and, for some, a `clientSecret`. The comment above it (line 235) reads:
  "For the moment these details are hard-coded, since dynamic client registration is not yet
  supported. Don't copy these values for your own application - register one for yourself!"
  [TB-providers]
- **Google** (`accounts.google.com`, line 242): client ID **and client secret** in the source,
  `usePKCE: true`, `useExternalBrowser: true`, no `redirectionEndpoint` and no `schemeRedirect`.
  Scopes: `https://mail.google.com/` for IMAP, POP3 and SMTP, plus CardDAV and CalDAV scopes
  requested together at mail setup (lines 8–21) [TB-providers].
- **Microsoft** (`login.microsoftonline.com`, line 327): client ID only, no secret, `/common`
  endpoints, `useExternalBrowser: true`. Scopes are the three Exchange scopes plus
  `offline_access` (lines 30–35). A pref `mail.microsoft.useM365Sandbox` swaps in a sandbox tenant
  for prototyping (line 660) [TB-providers].
- **Redirect handling.** With no `redirectionEndpoint`, the redirect defaults to
  `http://127.0.0.1` (`OAuth2.sys.mjs` lines 112–113).
  `ExternalRequest.startLoopbackRedirectListener()` opens a server socket on a random port and
  rewrites the redirect URI with it (from line 841).
  PKCE uses `S256` (line 222). The client secret, when present, is sent in the token request
  (line 425ff) [TB-oauth2]. The default switched from `localhost` to `127.0.0.1` in Bug 2045029
  [TB-log].
- **Prefs.** `mailnews.oauth.useExternalBrowser` is `true` (system browser instead of a
  Thunderbird window). `mailnews.oauth.useSchemeRedirect` is `true` outside ESR and lets providers
  that support it use a `net.thunderbird://` redirect; neither Google nor Microsoft has one
  configured (`mailnews/mailnews.js` lines 1068–1073) [TB-prefs][TB-providers].
- **Bring your own client.** Since Bug 1967370 (2025-05-22) an extension API registers OAuth
  providers at run time. The schema says "You will need to have a client ID registered with the
  provider to use this API" and requires a loopback redirect with the external browser. Since Bug
  2055286 (2026-08-07) extension providers may override built-in hostnames, and the schema gives
  `["office365.com", "outlook.com"]` as the example for overriding Microsoft
  (`mail/components/extensions/schemas/oauth_provider.json`) [TB-ext][TB-log].

## 11. KMail (KDE PIM)

The KMail tree itself has no OAuth code. Its only mention of Gmail authentication is the
"Gmail" option for IMAP accounts in the handbook
(`doc/kmail2/getting-started.docbook` lines 471–481) [KM-doc]. The code lives in other KDE
repositories:

- **Gmail, IMAP:** `kdepim-runtime/resources/imap/gmailpasswordrequester.cpp` defines
  `GOOGLE_API_KEY` (the client ID) and `GOOGLE_API_SECRET` as string macros and passes both to
  libkgapi's `AccountManager` with `KGAPI2::Account::mailScopeUrl()` [KDE-imap-gmail].
- **Gmail, SMTP:** `kmailtransport/src/plugins/smtp/gmailpasswordrequester.cpp` holds `apiKey` and
  `apiSecret` as static strings. They are the same values as the IMAP resource's [KDE-smtp-gmail].
- **libkgapi** runs the browser flow with `QDesktopServices::openUrl`, a local listener on
  `QHostAddress::LocalHost`, and redirect `http://127.0.0.1:<port>`. It sends `client_secret` in
  the token request. I found no `code_challenge` in its auth jobs
  (`src/core/private/fullauthenticationjob.cpp`, `newtokensfetchjob.cpp`) [KDE-kgapi].
- **Microsoft:** `kdepim-runtime/resources/imap/outlookpasswordrequester.cpp` (added 2024-01-30)
  and `kmailtransport/src/plugins/smtp/outlookpasswordrequester.cpp` hold the same `clientId`, with
  `tenantId` `common`, and request `IMAP.AccessAsUser.All` or `SMTP.Send` plus `offline_access`.
  Tokens go to the system keychain through QtKeychain [KDE-imap-outlook][KDE-smtp-outlook].
- **kmailtransport's `OutlookOAuthTokenRequester`** implements PKCE with `S256`, listens on
  `QHostAddress::LocalHost`, and uses `http://localhost:<port>` as the redirect; there is no
  client secret [KDE-outlook-req].
- These files are LGPL-2.0-or-later, so KDE publishes its Google secret in public source under a
  free license, as Mozilla does under MPL-2.0.

## 12. Reusing another project's client IDs

- **Google forbids it.** The API Terms require each developer to "make reasonable efforts to
  prevent and discourage other API Clients from using your credentials" and forbid masking "your
  API Client's identity" [G-terms]. A fork that ships Thunderbird's or KDE's Google client would
  show Thunderbird's or KDE's name on the consent screen. It would also sit inside their 100-token
  per-account limit and their verification scope [GI-oauth2][GI-restricted] _(inference)_.
- **Microsoft forbids it.** "Access Credentials are non-transferable and non-assignable. You must
  keep your Access Credentials confidential, and You may not share your Access Credentials with
  anyone else." Access Credentials are defined as "security keys, secrets, tokens, and other
  credentials" [MS-terms]. Whether a bare public client ID counts as an Access Credential is not
  spelled out. The Microsoft consent screen would still show the other project's app name
  _(inference)_.
- **The owners ask you not to.** Mozilla's source says "Don't copy these values … register one for
  yourself!" [TB-providers].
- **Operational risk.** Either owner can rotate, restrict or delete its client at any time.
  Thunderbird already swaps endpoints by pref and lets extensions override providers, which shows
  its client set is not a stable public interface [TB-providers][TB-ext] _(inference)_.

## 13. How the two runtime modes fit the providers

| | Local mode (sidecar on the user's machine) | Server mode (operator's HTTPS origin) |
| --- | --- | --- |
| Google client type | Desktop app (public) [GI-native] | Web application (confidential) [GI-webserver] |
| Google redirect | `http://127.0.0.1:<random port>` served by the sidecar; system browser | `https://<operator origin>/<path>`, registered on the client |
| Google device flow | Not available for the Gmail scope | Not available for the Gmail scope |
| Google CASA trigger | Not obviously triggered; decided at review _(inference)_ | Triggered: data flows "from or through a server" [GI-restricted] |
| Microsoft client type | Public, "Mobile and desktop" [MS-desktop] | Confidential web, or public with device code |
| Microsoft redirect | `http://localhost` (port ignored), or `msal<client id>://auth` for Electron [MS-reply][MS-desktop] | `https://<operator origin>/<path>`, up to 100 per registration [MS-reply] |
| Microsoft device flow | Possible [MS-imap] | Possible [MS-imap]; no redirect URI needed |
| Refresh token holder | Sidecar; Microsoft 90 days rolling, Google until idle 6 months | Server; same lifetimes |

The redirect must reach the component that holds the PKCE verifier and does the token exchange:
the sidecar in Local mode, the server in Server mode. The SPA on `flock://app` cannot be a
redirect target for Google, and should not do the exchange for Microsoft (24-hour `spa` tokens)
_(inference from §4, §9)_.

## Implications for flock

Facts and trade-offs only; no decision.

### A client registered by the flock project

- **Google, Local mode.** One Desktop-app client with loopback redirect and PKCE matches what
  Thunderbird and KDE do. To serve more than 100 users it needs brand and restricted-scope
  verification: a verified domain with homepage and privacy policy, a demo video, a scope
  justification, and annual re-verification. Whether CASA applies to a device-only client is
  decided by Google [GI-restricted]. Google may also push an IMAP client toward the Gmail API
  under the minimum-scope rule, which would change flock's Gmail backend [GH-faq].
- **The open-source conflict is unresolved.** Google's guide embeds the desktop ID and secret in
  the app, while its Terms say credentials "may not be embedded in open source projects"
  [GI-oauth2][G-terms]. Both reference clients embed them in public source anyway. A middle path
  is keeping the values out of the repository and injecting them at release build time, so the
  source stays credential-free and official binaries carry them _(inference)_. Self-built
  binaries would then have no Google client until the builder supplies one.
- **Google, Server mode.** A project Web client would have to list every operator's HTTPS origin
  as a redirect URI. Each new one may trigger re-verification [GH-unverified], and the project
  would take on CASA for servers it does not run [GI-restricted] _(inference)_. A
  project-hosted relay redirect would avoid the origin list, but it puts project infrastructure
  in every Gmail sign-in and brings in the "through a server" CASA trigger _(inference)_.
- **Microsoft, Local mode.** One multitenant + personal public client, loopback redirect, PKCE,
  no secret, as Thunderbird and KDE do. Registration needs an Entra directory (free). Reaching
  tenants with risk-based step-up consent needs publisher verification, which needs a Partner
  Program account and a domain of the project's own [MS-breaking][MS-pubver]. Admins of locked-down
  tenants will want the client ID to allowlist [MS-consent].
- **Microsoft, Server mode.** The same public client can serve every instance through the device
  code flow, with no redirect URI. A confidential web client would hit the 100-redirect limit and
  require a project secret on every operator's server _(inference from [MS-reply], [MS-imap])_.
- **Ongoing cost.** Someone in the project owns two provider consoles, the Google verification
  cycle and the yearly CASA invoice if it applies. The project's identity appears on every
  consent screen.

### Clients registered by each Server-mode operator

- **Fits the redirect model.** The operator registers a Google Web client and a Microsoft web
  client for their own origin and keeps the secrets server-side, as both providers intend for
  confidential clients [GI-webserver][MS-clients].
- **Google limits per operator.** A Workspace operator can make the app *Internal*: no warning
  screen, no cap, no verification, organization users only. Admins may need to allow restricted
  Gmail access for internal apps [GH-exempt][GH-audience]. A personal-Gmail operator (family, a
  few friends) falls under the personal-use exception: unverified screen and a 100-user cap, but
  normal token lifetimes once the project is set to *In production* [GH-exempt][GH-unverified].
  Leaving it in *Testing* makes Gmail tokens expire every 7 days, which breaks background sync
  [GI-oauth2].
- **Microsoft per operator.** A single-tenant registration in the operator's own tenant needs no
  publisher verification for its own users. A multitenant + personal registration for mixed users
  meets the same consent rules as a project client [MS-tenancy][MS-pubver].
- **Cost moves to the operator.** Two developer consoles, redirect URIs, secrets in flock's config
  and their rotation. For Gmail, the operator's project carries the 100-user cap and the
  verification duty if it grows beyond personal use.
- **Local mode is unaffected.** There is no operator in Local mode, so this model alone leaves
  desktop users to bring their own client ID, or leaves Local mode without Gmail and Microsoft
  _(inference)_.

### Both

- **Thunderbird is the precedent.** It ships built-in clients and lets extensions override them per
  hostname and email domain for organizations that need their own [TB-ext]. For flock, the
  equivalent would be project clients as the default for Local mode, with configuration that lets
  an operator (or a Local-mode user) supply their own client ID and secret per provider
  _(inference)_.
- **Data model.** Configuration needs per-provider issuer settings: client ID, optional secret,
  tenant, redirect, and flow (loopback, web redirect, device code). Each stored grant has to
  record which client issued its refresh token, because a refresh token only works with the
  client that obtained it _(inference)_.
- **Trade-off.** Out-of-the-box sign-in for desktop users and small servers, at the price of the
  project carrying the Google verification and the provider relationships, plus a second
  configuration path to document and test.

### Common to every option

- **Store refresh tokens encrypted, on the backend.** Google requires refresh tokens "encrypted at
  rest" for restricted scopes [GI-gmailpolicy]. That feeds
  [dyle71/flock#10](https://github.com/dyle71/flock/issues/10).
- **Handle revocation.** Google drops Gmail-scoped refresh tokens on password change and after six
  months idle, and evicts the oldest past 100 per client and account [GI-oauth2]. Microsoft
  revokes on password change and admin action [MS-refresh]. Background sync needs a "sign in
  again" state per account.
- **Use the system browser and PKCE everywhere.** Both providers recommend PKCE; Google rejects
  embedded user-agents [GI-native][MS-authcode].

## Sources

Google Identity and Google Workspace developer docs (read 2026-11-11)

- [GI-native] OAuth 2.0 for iOS & Desktop Apps (last updated 2026-10-30):
  https://developers.google.com/identity/protocols/oauth2/native-app
- [GI-oauth2] Using OAuth 2.0 to Access Google APIs (Installed applications; Refresh token
  expiration) (last updated 2026-10-29): https://developers.google.com/identity/protocols/oauth2
- [GI-webserver] OAuth 2.0 for Web Server Applications (Redirect URI validation rules,
  `access_type`) (last updated 2026-10-30):
  https://developers.google.com/identity/protocols/oauth2/web-server
- [GI-restricted] Restricted scope verification (Security assessment; Exceptions to verification
  requirements) (last updated 2026-11-10):
  https://developers.google.com/identity/protocols/oauth2/production-readiness/restricted-scope-verification
- [GI-workspace] Additional considerations for Google Workspace (last updated 2026-05-26):
  https://developers.google.com/identity/protocols/oauth2/production-readiness/google-workspace
- [GI-scopes] Choose Gmail API scopes (last updated 2026-09-10):
  https://developers.google.com/workspace/gmail/api/auth/scopes
- [GI-xoauth2] OAuth 2.0 Mechanism (IMAP/SMTP XOAUTH2) (last updated 2026-09-15):
  https://developers.google.com/workspace/gmail/imap/xoauth2-protocol
- [GI-gmailpolicy] Google Workspace API user data and developer policy (approved Gmail use cases;
  required security measures) (last updated 2026-09-03):
  https://developers.google.com/workspace/gmail/api/policy
- [G-udp] Google API Services User Data Policy (Additional Requirements for Specific API Scopes):
  https://developers.google.com/terms/api-services-user-data-policy
- [G-terms] Google APIs Terms of Service, Sections 3c and 4b (last modified 2021-11-09):
  https://developers.google.com/terms

Google Cloud help center (read 2026-11-11)

- [GH-unverified] Unverified apps: https://support.google.com/cloud/answer/7454865
- [GH-audience] Manage App Audience (user type, publishing status):
  https://support.google.com/cloud/answer/15549945
- [GH-exempt] When is verification not needed: https://support.google.com/cloud/answer/13464323
- [GH-requirements] Verification requirements: https://support.google.com/cloud/answer/13464321
- [GH-casa] Security Assessment: https://support.google.com/cloud/answer/13465431
- [GH-annual] Annual Recertification: https://support.google.com/cloud/answer/13463816
- [GH-restricted] Restricted Scopes: https://support.google.com/cloud/answer/13464325
- [GH-faq] Frequently Asked Questions (IMAP / SMTP; Security Assessment):
  https://support.google.com/cloud/answer/13463817

App Defense Alliance

- [ADA-casa] CASA overview (last updated 2026-06-27): https://appdefensealliance.dev/casa
- [ADA-tier2] CASA Tier 2 Process (last updated 2024-11-07):
  https://appdefensealliance.dev/casa/tier-2/tier2-overview

Microsoft identity platform and Exchange docs (read 2026-11-11; `ms.date` in parentheses)

- [MS-clients] Public client and confidential client applications (2025-05-14):
  https://learn.microsoft.com/en-us/entra/identity-platform/msal-client-applications
- [MS-authcode] OAuth 2.0 authorization code flow (2026-01-09):
  https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow
- [MS-reply] Redirect URI (reply URL) best practices and limitations (2025-05-14):
  https://learn.microsoft.com/en-us/entra/identity-platform/reply-url
- [MS-desktop] Desktop app that calls web APIs: App registration (2024-04-09):
  https://learn.microsoft.com/en-us/entra/identity-platform/scenario-desktop-app-registration
- [MS-tenancy] Single and multitenant apps in Microsoft Entra ID (2025-03-13):
  https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps
- [MS-register] Register an application (2026-05-14):
  https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app
- [MS-breaking] What's new for authentication? ("Applications must be registered in a directory",
  June 2024): https://learn.microsoft.com/en-us/entra/identity-platform/reference-breaking-changes
- [MS-pubver] Publisher verification overview (2024-08-13):
  https://learn.microsoft.com/en-us/entra/identity-platform/publisher-verification-overview
- [MS-consent] Configure how users consent to applications (2025-06-15):
  https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent
- [MS-refresh] Refresh tokens in the Microsoft identity platform (2025-11-05):
  https://learn.microsoft.com/en-us/entra/identity-platform/refresh-tokens
- [MS-imap] Authenticate an IMAP, POP or SMTP connection using OAuth (2021-09-21):
  https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth
- [MS-smtpauth] Enable or disable SMTP AUTH in Exchange Online (2026-09-16):
  https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/authenticated-client-smtp-submission
- [MS-terms] Microsoft identity platform for developers Terms of Use, "Access Credentials"
  (2021-11-10): https://learn.microsoft.com/en-us/legal/microsoft-identity-platform/terms-of-use

Standards

- [RFC8252] RFC 8252, OAuth 2.0 for Native Apps, §7.3, §8.3, §8.5:
  https://www.rfc-editor.org/rfc/rfc8252

Thunderbird (read-only, `/home/dyle/Source/other/releases-comm-central`, commit `2669a12bd1`)

- [TB-providers] `mailnews/base/src/OAuth2Providers.sys.mjs` (scope constants lines 8–35,
  `kIssuers` lines 230–442, Google entry line 242, Microsoft entry line 327, sandbox pref line
  660, `registerProvider` line 686)
- [TB-oauth2] `mailnews/base/src/OAuth2.sys.mjs` (default redirect lines 112–113, PKCE line 222,
  external-browser branch lines 251–310, token request lines 425–446, `ExternalRequest` from line
  770, loopback listener from line 841)
- [TB-prefs] `mailnews/mailnews.js` lines 1061–1074
- [TB-ext] `mail/components/extensions/schemas/oauth_provider.json`
- [TB-log] `git log` of `OAuth2Providers.sys.mjs`: Bug 1967370 (2025-05-22, run-time client
  registration API), Bug 2045029 (loopback default `127.0.0.1`), Bug 2055286 (2026-08-07,
  extension override of built-in hostnames)

KMail and KDE PIM

- [KM-doc] `/home/dyle/Source/other/kmail/doc/kmail2/getting-started.docbook` lines 73–75, 471–481
  (commit `98c911893`)
- [KDE-imap-gmail] `resources/imap/gmailpasswordrequester.cpp`, KDE/kdepim-runtime `master`:
  https://github.com/KDE/kdepim-runtime/blob/master/resources/imap/gmailpasswordrequester.cpp
- [KDE-imap-outlook] `resources/imap/outlookpasswordrequester.cpp`, same repository
- [KDE-smtp-gmail] `src/plugins/smtp/gmailpasswordrequester.cpp`, KDE/kmailtransport `master`:
  https://github.com/KDE/kmailtransport/blob/master/src/plugins/smtp/gmailpasswordrequester.cpp
- [KDE-smtp-outlook] `src/plugins/smtp/outlookpasswordrequester.cpp`, same repository
- [KDE-outlook-req] `src/outlookoauthtokenrequester.cpp`, same repository
- [KDE-kgapi] `src/core/private/fullauthenticationjob.cpp` and `newtokensfetchjob.cpp`,
  KDE/libkgapi `master`: https://github.com/KDE/libkgapi/tree/master/src/core/private
