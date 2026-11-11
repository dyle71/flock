# Packaging a Rust sidecar inside Electron

Research for [dyle71/flock#7](https://github.com/dyle71/flock/issues/7), part of the wayfinder map
[dyle71/flock#1](https://github.com/dyle71/flock/issues/1). Feeds the HITL decision "Runtime modes:
local and server" ([dyle71/flock#9](https://github.com/dyle71/flock/issues/9)).

Researched 2026-11-11. Versions current on that day: Electron 45.3.0, electron-builder 26.18.1
(27.0.0 is in alpha, and electron.build's default docs already describe 27), electron-updater 6.8.12,
Electron Forge 8.0.1. Statements marked _(inference)_ are my own reasoning from the cited facts. No
source states them directly.

## Question

How does an Electron app ship, start, supervise and stop a bundled native binary? That covers
packaging, process API, port or socket selection, securing the local channel, crash and restart,
single instance, background sync with a tray, and start at login. How are the shell and the binary
updated and signed together on Linux, macOS and Windows? Can the same local sidecar safely serve a
plain browser on localhost? Which real apps already do this?

## Summary

- **Shipping.** electron-builder's `extraResources` copies the binary into `Contents/Resources`
  (macOS) or `resources/` (Linux, Windows). The app finds it at runtime through
  `process.resourcesPath`. Both electron-builder and Forge (through `@electron/osx-sign`) sign
  nested Mach-O binaries on macOS. electron-builder also signs extra `.exe` files on Windows.
  Apple's preferred place for a helper tool is `Contents/MacOS` or `Contents/Helpers`, not
  `Contents/Resources`.
- **Starting.** `utilityProcess` only runs a Node.js script. A native binary is started with Node's
  `child_process.spawn` from the main process.
- **Channel.** A loopback TCP port works directly with Chromium's `fetch` and the SPA. A Unix
  socket or named pipe keeps out other OS users and every browser page, but the renderer cannot
  reach it without a main-process proxy.
- **Security.** Anything on 127.0.0.1 can be reached by every local user and process. Browser
  pages can also reach it, through CSRF and DNS rebinding (Ollama CVE-2024-28224). The standard
  defenses are a per-launch secret, a strict `Host` header check, and `Origin` or `Sec-Fetch-Site`
  checks. Chrome 142's Local Network Access prompt adds browser-side gating. It does not replace
  server-side checks.
- **Lifecycle.** On Windows, libuv puts non-detached children in a kill-on-close job object, so the
  sidecar dies with Electron. On Linux and macOS it does not, so the sidecar has to watch its
  parent. stdin EOF works on all three. Electron's login-item API covers only macOS and Windows.
  Linux needs an XDG autostart file, or the Background portal under Flatpak.
- **Updates.** When the sidecar ships inside the app bundle, one update artifact replaces both
  pieces at once. electron-updater supports macOS (zip), Windows NSIS, AppImage, deb, rpm and
  pacman, but not Squirrel.Windows. Forge's update path is Electron's built-in `autoUpdater`, and
  Electron has no built-in auto-update for Linux. Flatpak updates through its remote.

## 1. Shipping the binary in the package

**electron-builder.** The v26 config schema defines `extraResources` this way: "copy the file or
directory with matching names directly into the app's resources directory (`Contents/Resources`
for MacOS, `resources` for Linux and Windows)" ([electron-builder 26.18.1 `scheme.json`][eb-scheme]).
v27 moves `files`, `extraResources` and `extraFiles` under a new `fileOptions` key and warns about
the old placement ([v27 configuration][eb-config]). `asarUnpack` exists for files that must not
live inside `app.asar` ([schema][eb-scheme]). A sidecar placed with `extraResources` never enters
the asar archive.

**Electron Forge.** `packagerConfig.extraResource` does the same job. Ollama's former Electron macOS
app used it to put its Go `ollama` binary into `Contents/Resources`, then started
`path.join(process.resourcesPath, 'ollama')` when packaged ([ollama `macapp/forge.config.ts`][ollama-forge],
[`macapp/src/index.ts`][ollama-index]).

**Mailspring** uses a different layout. It keeps its C++ sync engine next to the app sources and
rewrites the path from `app.asar` to `app.asar.unpacked` before spawning it
([Mailspring `mailsync-process.ts`][mailspring-proc]).

**macOS bundle placement.** Apple's "Placing content in a bundle" lists "help app, helper tool" at
`Contents/MacOS/` or `Contents/Helpers/`, and "resource" at `Contents/Resources/`. It defines code
content as "executable code, like a helper tool … executable code means a Mach-O image"
([Apple: Placing content in a bundle][apple-bundle]). TN2206 explains why the location matters:
"putting code into other places will cause it to be sealed as data (resource) files. This causes
this code to be sealed twice … Also, this can break the outer signature of apps that use their own
update mechanisms to replace nested code" ([TN2206][tn2206]). In practice, `extraResources` lands a
Mach-O binary in `Contents/Resources` and still signs and notarizes, as Ollama's app shows. It is
just not Apple's recommended location. electron-builder's `extraFiles` copies relative to
`Contents/` instead, so it can target `Contents/MacOS` _(inference from the `extraFiles` /
`extraResources` distinction; verify with a build)_.

## 2. Starting the binary: `child_process` versus `utilityProcess`

- `utilityProcess.fork(modulePath)` "creates a child process with Node.js and Message ports
  enabled". `modulePath` is the "Path to the script that should run as entrypoint". It cannot run
  an arbitrary executable. Its stdin can only be `ignore`: "Configuring stdin to any property other
  than ignore is not supported" ([Electron utilityProcess][e-utility]).
- So the sidecar is started with Node's `child_process.spawn(binary, args, { env, windowsHide })`
  in the main process ([Node child_process][node-cp]). Ollama's old app and Mailspring both do this
  ([ollama][ollama-index], [Mailspring][mailspring-proc]).
- Mailspring builds the child's environment on a null-prototype object to block environment
  injection through prototype pollution (GHSA-gjr7-3mj2-cr54), and passes account secrets to the
  child over stdin, not argv ([Mailspring][mailspring-proc]).

## 3. The channel: loopback port versus Unix socket or named pipe

**Loopback TCP.**
- **Ephemeral ports.** If the sidecar binds `127.0.0.1:0` and reports the port it got (for example
  as a first JSON line on stdout), there is no race between picking a free port and binding it
  _(inference)_.
- **Fixed ports.** These are simpler for a browser user who bookmarks a URL, but they collide with
  other software. Rancher Desktop binds a fixed `127.0.0.1:6107` ([httpCommandServer.ts][rd-server]).
  Ollama uses a fixed well-known port.
- **Origins change with the port.** A web origin includes the port ([WHATWG HTML: origin][html-origin]).
  If the SPA is loaded from `http://127.0.0.1:<random>`, its localStorage, IndexedDB and service
  workers belong to a new origin on every launch _(inference)_.
- **Cookies ignore ports.** "Cookies do not provide isolation by port … servers SHOULD NOT both run
  mutually distrusting services on different ports of the same host and use cookies to store
  security-sensitive information" ([RFC 6265 §8.5][rfc6265]).

**Unix domain socket / named pipe.**
- Filesystem permissions (0600, or a directory such as `$XDG_RUNTIME_DIR`) shut out other users.
  Browsers cannot address a socket at all, so CSRF and DNS rebinding do not apply.
- **Rust support.** In tokio, `UnixListener` and `UnixStream` are Unix-only. Windows gets
  `tokio::net::windows::named_pipe` ([tokio::net][tokio-net]). Two code paths are needed.
- **Renderer access.** Chromium's `fetch` cannot use a socket path. The renderer would reach the
  sidecar through a custom scheme registered with `protocol.handle(scheme, handler)`. The main
  process forwards each request with Node's `http.request({ socketPath })`.
  `registerSchemesAsPrivileged` with `standard`, `secure`, `supportFetchAPI` and `stream` lets the
  SPA use `fetch` and streaming against that scheme ([Electron protocol][e-protocol]). Serving the
  SPA from a custom scheme also gives it a stable origin. Electron's security checklist recommends
  custom protocols over `file://` ([Electron security][e-security]).
- **Prior art.** Mullvad VPN's Electron GUI talks gRPC to its Rust daemon over `/var/run/mullvad-vpn`
  (Linux, macOS) or `//./pipe/Mullvad VPN` (Windows). The daemon sets explicit endpoint permissions
  ([mullvad-paths `rpc_socket.rs`][mullvad-sock], [mullvad-management-interface][mullvad-mi]).
  Keybase's Electron GUI talks to its Go service over `keybased.sock`, under the XDG runtime dir on
  Linux and `\\.\pipe\kbservice…` on Windows ([keybase `platform.tsx`][keybase-platform]).
- **Trade-off for flock** _(inference)_. A socket only covers the Electron path. Server mode and any
  browser access need TCP anyway, so a socket-only local mode means two transports.

## 4. Securing the local channel

**Threats to a loopback HTTP sidecar** _(inference, with cited instances)_:

1. **Other local users and processes.** Any of them can connect to 127.0.0.1.
2. **Web pages in the user's normal browser.**
   - **CSRF.** A page can send "simple" cross-origin requests (form POSTs) to the sidecar.
   - **DNS rebinding.** An attacker's hostname is re-resolved to 127.0.0.1, after which the page
     reads responses as same-origin. CVE-2024-28224: "Ollama before 0.1.29 has a DNS rebinding
     vulnerability that can inadvertently allow remote access to the full API" ([NVD][cve-ollama]).
3. **The flock renderer itself, if it is compromised.** An e-mail client renders hostile HTML.
   Whatever secret the renderer holds, an XSS in the renderer holds too. That argues for isolating
   message bodies (sandboxed frames) and for keeping the secret out of renderer JS where possible.

**Defenses and their sources:**

- **Per-launch secret.** Electron generates a random token, hands it to the sidecar, and attaches it
  to every request. Rancher Desktop does this for its own loopback API with a random password,
  HTTP Basic auth, and a state file written with `mode: 0o600` for its CLI
  ([httpCommandServer.ts][rd-server]). Jupyter Server's token authentication is "on by default": a
  token is generated at launch, logged as a `?token=` URL, and swapped for a cookie on first visit
  ([Jupyter Server security][jupyter-sec]).
- **Delivering the secret** _(inference)_:
  - **stdin.** The cleanest route. Mailspring sends secrets this way.
  - **Environment.** `/proc/<pid>/environ` requires a ptrace-mode access check
    ([proc_pid_environ(5)][proc-environ]).
  - **argv.** Avoid it. Other users can normally read `/proc/<pid>/cmdline`
    ([proc_pid_cmdline(5)][proc-cmdline]).
- **Keeping the secret out of renderer JS.** `session.webRequest.onBeforeSendHeaders` can add an
  `Authorization` header in the main process to requests matching the sidecar's URL pattern.
  "Only the last attached `listener` will be used," so one module must own this hook
  ([Electron webRequest][e-webrequest]). A custom-protocol proxy (section 3) does the same.
- **Host header allowlist** against DNS rebinding. Syncthing wraps its GUI handler in
  `localhostMiddleware`, which rejects any `Host` that is not `localhost`, `*.localhost` or a
  loopback IP with "Host check error" (403), unless `InsecureSkipHostCheck` is set
  ([syncthing `lib/api/api.go`][syncthing-api]).
- **CSRF.**
  - **Custom headers.** OWASP: "All modern browsers designate requests with custom headers as 'to be
    preflighted'". A required custom header or bearer token therefore blocks form-based CSRF.
  - **Fetch metadata.** `Sec-Fetch-Site` is "the primary signal". Origin verification is a
    "mandatory" fallback, and "Missing headers and `Origin: null` are not evidence of a same-origin
    request".
  - **SameSite.** "defense-in-depth … does not replace a proper CSRF defense"
    ([OWASP CSRF cheat sheet][owasp-csrf]).
  - Syncthing also wraps `/rest` in a CSRF manager ([api.go][syncthing-api]).
- **Browser-side gating.** Chrome's Local Network Access asks permission before a public site may
  send requests "from the public network to a local network or loopback destination". The prompt
  "is launching in Chrome 142" (update of 2025-09-29). It succeeds the paused Private Network
  Access preflight effort. Local-to-localhost protections are listed as future work
  ([Chrome: Local Network Access][chrome-lna]). It reduces exposure in Chrome only and gives no
  protection against other local processes.
- **Renderer hardening** belongs to the Electron security checklist: context isolation, sandbox,
  CSP, limited navigation, IPC sender validation ([Electron security][e-security]).

## 5. Supervision: crashes, restarts, shutdown, orphans

**Detection.** The `ChildProcess` emits `exit` with `code` and `signal`
([Node child_process][node-cp]).

**Restart policy examples:**

- Ollama's old app restarted the server one second after any exit. It removed that listener in
  `before-quit` before sending `SIGINT` ([ollama `index.ts`][ollama-index]).
- Mailspring keeps a per-account crash history (`MAX_CRASH_HISTORY = 10`). It stops restarting if a
  sync worker "has … crashed more than 5 times in the last 5 minutes, or failed to authenticate
  twice in the last 30", and shows the error ([Mailspring `mailsync-bridge.ts`][mailspring-bridge]).

**Shutdown.**

- **Windows.** Node's `subprocess.kill()` has no graceful signal there: "`'SIGKILL'`, `'SIGTERM'`,
  `'SIGINT'` and `'SIGQUIT'` terminate the process forcefully and abruptly" ([Node child_process][node-cp]).
  A graceful sidecar shutdown on Windows needs its own channel, such as closing stdin or calling an
  authenticated shutdown endpoint _(inference)_.
- **Missing quit events.** `before-quit` and `will-quit` are "not … emitted if the app is closed
  due to a shutdown/restart of the system or a user logout" on Windows ([Electron app][e-app]).

**Orphans: the sidecar must not outlive a crashed Electron.**

- **Windows.** libuv puts every non-detached child in a job object with
  `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`: "if this process exits it is closed and all the processes
  it contains are killed" ([libuv `src/win/process.c`][libuv-win]). That flag "causes all processes
  associated with the job to terminate when the last handle to the job is closed"
  ([Microsoft: JOBOBJECT_BASIC_LIMIT_INFORMATION][ms-job]).
- **Linux.** The sidecar can call `prctl(PR_SET_PDEATHSIG)`. The "parent" is "the thread that
  created this process", so the spawning thread matters ([PR_SET_PDEATHSIG(2const)][pdeathsig]).
- **macOS.** No equivalent was found. The portable pattern is to exit when stdin reaches EOF, which
  happens when the parent dies on all three OSes _(inference)_.
- **Daemon conflict.** Any of these mechanisms conflicts with the sidecar deliberately outliving
  the window (section 7).

## 6. Single instance

- `app.requestSingleInstanceLock(additionalData)` together with the `second-instance` event in the
  primary instance is Electron's mechanism. If the second instance runs as another user, "the
  `argv` array will not include the arguments" ([Electron app][e-app]). Ollama's old app used it
  ([index.ts][ollama-index]).
- The Electron lock does not stop a second sidecar from opening the same mail store, for example a
  `flock` binary started by hand in local mode. A lock in the sidecar's data directory (an exclusive
  file lock or a DB-level lock) covers that case _(inference)_.

## 7. Background sync, tray, start at login

**Keeping the app alive.** "If you do not subscribe to this event and all windows are closed, the
default behavior is to quit the app". Handling `window-all-closed` keeps the main process, and so
the supervised sidecar, running with no window ([Electron app][e-app]).

**Tray on Linux.** `Tray` uses StatusNotifierItem and falls back to `GtkStatusIcon`. Activation
semantics vary: "for some environments it is left mouse click, but for some it might be double left
mouse click". Menu changes need another `setContextMenu` call ([Electron Tray][e-tray]).

**Start at login:**

- **macOS and Windows.** `app.setLoginItemSettings` and `getLoginItemSettings` are documented for
  these two only. `type` (macOS) defaults to `mainAppService`. `args` (Windows) must match between
  set and get. As of Electron 45, `openAsHidden` was removed. Also: "When an app isn't packaged,
  code signed, and notarized, `openAtLogin` may silently fail" ([Electron app][e-app]).
- **Linux.** Electron has no API. Native packages write a `.desktop` file into the user's autostart
  directory, which is `~/.config/autostart/` when `$XDG_CONFIG_HOME` is unset
  ([XDG Autostart spec][xdg-autostart]).
- **Flatpak.** Apps request this through the portal: `org.freedesktop.portal.Background.RequestBackground`
  with `autostart: true` and an optional `commandline` ([Background portal][portal-bg]). Electron
  does not wrap this portal, so it needs a D-Bus call from the main process _(inference)_.

**Alternative topology** _(inference)_. Mullvad and Keybase run their backend as an independent
service that the GUI connects to ([mullvad][mullvad-sock], [keybase][keybase-platform]). A flock
sidecar could likewise register its own login item, LaunchAgent, systemd user unit or scheduled
task, and keep syncing even when Electron is not running. The price:

- the sidecar's lifecycle is decoupled from the app's;
- updates must stop and restart a process they did not start;
- section 5's orphan protection has to go.

## 8. Updating and signing shell and sidecar together

**Packaging tools and update paths:**

| Concern | electron-builder + electron-updater | Electron Forge |
| --- | --- | --- |
| Update engine | electron-updater (own engine, reads `latest*.yml`) | Electron's built-in `autoUpdater` (Squirrel.Mac, Squirrel.Windows or MSIX) via `update-electron-app` / update.electronjs.org or a self-hosted server |
| macOS | dmg + **zip** (zip required for Squirrel.Mac). App must be signed | App must be signed |
| Windows | **NSIS** only. "Squirrel.Windows is not supported" | Squirrel.Windows maker. Electron's `autoUpdater` also detects MSIX |
| Linux | AppImage auto-installs on next launch. deb, rpm and pacman elevate via pkexec/sudo and need `installPendingUpdateIfAvailable()` | "There is no built-in support for auto-updater on Linux". Use the distro package manager |
| Delta updates | Block maps for NSIS and the macOS zip. AppImage embeds its map | Squirrel deltas (nupkg) |
| Staged rollout | `stagingPercentage` in `latest.yml` | Self-hosted servers (nucleus, hazel, …) |

Sources: [electron-builder auto-update][eb-autoupdate], [Electron autoUpdater][e-autoupdater],
[Forge auto-update][forge-autoupdate], [Forge Squirrel.Windows maker][forge-squirrel].

**Forge makers.** First-party makers are AppX, deb, DMG, Flatpak, MSIX, pkg, RPM, Snapcraft,
Squirrel.Windows, WiX MSI and ZIP. There is no first-party AppImage maker ([Forge docs index][forge-llms]).

**One artifact, one version** _(inference)_. Every format above replaces the app directory or
bundle as a unit: the `.app` via Squirrel.Mac, the NSIS install dir, the AppImage file, the
package. A sidecar inside the bundle is therefore always updated in lockstep with the shell. Two
consequences:

- The updater must stop the sidecar before installing. On Windows a running `.exe` cannot be
  replaced, and quit events may be skipped (section 5).
- Version skew remains possible only in the "Electron points at a remote flock server" path, so
  the shell and server need a version or capability handshake.

### macOS: signing, hardened runtime, notarization

- **Notarization requirements.** The notary service requires "code-signing for all of the
  executables you distribute", a Developer ID certificate, "the Hardened Runtime capability for
  your app and command line targets", a secure timestamp, and no `get-task-allow`. "Beginning in
  macOS 10.15, all software built after June 1, 2019, and distributed with Developer ID must be
  notarized." `altool` no longer works; use `notarytool` and `stapler`
  ([Apple: Notarizing macOS software][apple-notarize], [Resolving common notarization issues][apple-notarize-issues]).
- **Signing order.** Sign inside out: "all nested code must already be signed correctly or the
  signing attempt will fail". `--deep` signing "is for emergency repairs and temporary adjustments
  only" ([TN2206][tn2206]).
- **@electron/osx-sign** (used by Forge and Packager) walks the whole bundle, signs every binary
  file it finds, and accepts per-file options through `optionsForFile(filePath)`. A Rust binary
  can therefore get its own minimal entitlements
  ([osx-sign `util.ts` walk][osxsign-util], [Forge macOS signing][forge-mac]).
- **electron-builder v26** has `mac.binaries` ("Paths of any extra binaries that need to be
  signed"), `hardenedRuntime` (default `true`) and `entitlementsInherit` ([schema][eb-scheme]).
  v27 docs: "All binaries inside the app bundle must be signed". Electron needs exactly one
  exception, `com.apple.security.cs.allow-jit`, granted by default. An inherit plist "applies one
  plist to every nested binary". Notarization runs when `APPLE_API_KEY`/`APPLE_API_KEY_ID`/`APPLE_API_ISSUER`
  or `APPLE_ID`/app-specific password/`APPLE_TEAM_ID` are set, and the ticket is stapled
  automatically ([electron-builder notarization][eb-notarize], [mac][eb-mac]).
- **Entitlements for the sidecar** _(inference)_. A Rust sidecar does not JIT, so it needs no
  entitlements beyond the hardened runtime itself. Apple's own helper-tool guide enables Hardened
  Runtime on the tool target. It notes that for sandboxed (App Store) apps the tool carries only
  `app-sandbox` + `inherit` ([Apple: Embedding a helper tool][apple-helper]).
- **Updates.** Squirrel.Mac auto-update requires a signed app ([Electron autoUpdater][e-autoupdater]).

### Windows: Authenticode, installers

- **Signing scope in electron-builder.** It signs the main `.exe`. A transformer also signs every
  extra file whose name ends in `.exe` (or matches `signExts`) when copied through
  `extraResources`/`extraFiles`, so `flock.exe` gets signed with the same identity
  ([winPackager.ts][eb-winpackager]).
- **Signing backends.** `win.azureSignOptions` (Azure Trusted/Artifact Signing) or
  `win.signtoolOptions` ([schema][eb-scheme]).
- **Update verification.** `verifyUpdateCodeSignature` (default `true`) checks an update's
  signature against `publisherName` before installing ([schema][eb-scheme]).
- **Microsoft's current guidance** ([Code signing options, updated 2026-10-29][ms-signing]):
  - **Azure Artifact Signing** (formerly Trusted Signing): about $9.99/month, no hardware token,
    CI-friendly. Available to organizations in the USA, Canada, EU and UK. Individuals in the USA
    and Canada only.
  - **OV and EV certificates.** Since June 2023 the CA/Browser Forum requires OV keys on an HSM or
    hardware token. EV "previously bypassed SmartScreen … That behavior was removed in 2024".
  - **No certificate gets instant SmartScreen trust.** Reputation builds over time with a
    consistent identity.
  - **SignPath Foundation** offers free signing for qualifying open-source projects. That is
    relevant for a GPL-3.0 project.
- **Electron's code-signing tutorial** still says "Microsoft requires an EV certificate" since June
  2023 ([Electron code signing][e-codesign]). Microsoft's page above is newer and more precise:
  the 2023 change was the HSM key requirement.
- **Squirrel.Windows** is "a no-prompt, no-hassle, no-admin method". It re-launches the app with
  special arguments on install, update and uninstall ([Forge Squirrel.Windows][forge-squirrel]).

### Linux: AppImage, deb/rpm, Flatpak

- **No package signing.** electron-builder does not sign Linux packages. electron-updater's
  `allowUnverifiedLinuxPackages` defaults to `true` ([electron-builder auto-update][eb-autoupdate]).
  `linux.forceCodeSigning` affects only update manifests ([electron-builder code signing][eb-codesign]).
  For deb and rpm, trust normally comes from a signed repository the user adds _(inference)_.
- **Flatpak.** The usual base is `org.electronjs.Electron2.BaseApp` on the Freedesktop runtime,
  with zypak wrapping the binary ([Flatpak: Electron][flatpak-electron]). electron-builder's
  `FlatpakOptions` defaults to that base app but with `baseVersion`/`runtimeVersion` `20.08`, which
  is very old, so they would need overriding ([schema][eb-scheme]). Updates come from the Flatpak
  remote (e.g. Flathub), not from electron-updater. A sidecar inside `/app` runs in the same
  sandbox and needs `--share=network` for IMAP/SMTP _(inference)_.

## 9. Serving a plain browser on localhost from the local sidecar

**Is it safe?** The question is whether a loopback API that any browser tab can address can be
safe at all. The sources above give the needed controls _(inference for the combination)_:

1. **Host check.** Reject any request whose `Host` is not an expected loopback name (Syncthing's
   approach), which defeats DNS rebinding.
2. **Authentication on every API call.** Jupyter's model is a launch token in a one-time URL,
   exchanged for a session cookie. Rancher Desktop uses Basic auth with a 0600 credentials file.
3. **CSRF defense** for cookie-authenticated requests: a required custom header, plus
   `Sec-Fetch-Site`/`Origin` checks (OWASP).
4. **Cookie scope.** Cookies are shared across ports on the same host (RFC 6265 §8.5). Any other
   local web app on `127.0.0.1` or `localhost` can read and overwrite them. A dedicated host such
   as `flock.localhost` isolates them by host. Syncthing's host check already accepts
   `*.localhost` names.

**Alternative.** Don't serve browsers in local mode at all: browser access is what server mode is
for, and in local mode the sidecar only answers to the Electron shell (or listens on a socket).

## 10. Real-world examples

| App | Shell ↔ backend | Backend | How it is started | Channel and auth |
| --- | --- | --- | --- | --- |
| **Mailspring** (GPL-3.0 mail client) | Electron ↔ C++ `mailsync`, one process per account | Native, unpacked next to the asar | `child_process.spawn`. Crash tracker stops after 5 crashes in 5 min | JSON over stdin/stdout. Secrets over stdin |
| **Ollama** (old macOS app, removed 2025-11) | Electron (Forge) ↔ Go `ollama serve` | `extraResource` → `Contents/Resources` | `spawn`, restart after 1 s, `SIGINT` on quit. Single-instance lock, tray | Fixed loopback port, no auth (CVE-2024-28224 DNS rebinding) |
| **Mullvad VPN** (GPL-3.0) | Electron ↔ Rust `mullvad-daemon` | Separately installed system service | OS service manager | gRPC over UDS / named pipe with explicit permissions |
| **Keybase** | Electron ↔ Go `keybase service` | Separate service | Service manager / CLI | RPC over `keybased.sock` / named pipe |
| **Rancher Desktop** (Apache-2.0) | Electron ↔ Go helpers (`rdctl`, …) | Bundled binaries | Electron main | Loopback HTTP `127.0.0.1:6107`, random Basic-auth password in a 0600 file |
| **Syncthing** (not Electron) | Browser ↔ Go daemon GUI | — | — | Loopback HTTP with Host check, CSRF, API key, optional password |

Sources: [Mailspring][mailspring-proc], [Mailspring bridge][mailspring-bridge],
[Ollama forge config][ollama-forge], [Ollama index.ts][ollama-index],
[Ollama removal commit 54a76d3][ollama-removal], [Mullvad][mullvad-sock], [Keybase][keybase-platform],
[Rancher Desktop][rd-server], [Syncthing][syncthing-api].

## Implications for flock

These are facts and trade-offs for the "Runtime modes: local and server" decision. None of them is
a decision.

1. **Process API is settled by Electron itself.** A native binary means `child_process.spawn`
   from the main process. `utilityProcess` is out.
2. **Transport is the central trade-off.**
   - **Loopback TCP.** One transport for local mode, server mode and a possible browser. But
     local mode then inherits every loopback threat in section 4 and needs Host check, per-launch
     token and CSRF defenses from day one.
   - **UDS / named pipe.** Removes the browser and other-user threats for the Electron path. It
     costs a main-process proxy (custom protocol), two Rust listener implementations (Unix
     sockets, Windows named pipes), and still needs TCP for server mode.
3. **The SPA's origin matters.**
   - **From `http://127.0.0.1:<ephemeral>`.** The origin changes every launch, so client-side
     storage does not survive restarts.
   - **From a custom scheme with a proxy.** The origin is stable and the token can stay in the main
     process.
   - **From a fixed port.** Collides with other software and needs a port from the project's
     still-unassigned port series (CLAUDE.md).
4. **The token belongs out of argv.**
   - **Delivery.** stdin or the environment, not argv.
   - **Visibility.** Injecting it from the main process (`webRequest` or the protocol proxy) keeps
     it away from renderer JS. This matters because an e-mail client renders hostile HTML.
5. **Lifecycle choices interact.** "Sidecar dies with Electron" and "sidecar keeps syncing with
   Electron gone" exclude each other:
   - **Sidecar dies with Electron.** Windows gives this for free through libuv's job object. Linux
     and macOS need stdin-EOF or PDEATHSIG. "Keep syncing when the window is closed" then means
     keeping Electron's main process alive with a tray.
   - **Sidecar keeps syncing without Electron.** It needs its own autostart and service
     registration per OS, as Mullvad and Keybase do, and the updater must manage a process it
     did not start.
6. **Start at login** is one Electron call on macOS and Windows. Linux needs either an XDG
   autostart file (native packages) or the Background portal (Flatpak).
7. **Shipping the sidecar inside the app bundle keeps versions in lockstep.** Version skew only
   exists against a remote server, so the API needs version negotiation for that case. Updaters
   must stop the sidecar before replacing files, especially on Windows.
8. **Tooling.**
   - **electron-builder + electron-updater.** The only path with auto-update on AppImage, deb and
     rpm. NSIS on Windows. Signs sidecar `.exe`s and Mach-O binaries automatically. v27 is about to
     change the config schema.
   - **Forge.** Squirrel/MSIX and Electron's `autoUpdater`, no Linux auto-update, no first-party
     AppImage.
   - **Flatpak** updates outside both, and electron-builder's Flatpak defaults are stale.
9. **Signing cost and eligibility.**
   - **macOS.** Apple Developer Program, plus a Developer ID certificate, hardened runtime and
     notarization for both binaries.
   - **Windows.**
     - **Azure Artifact Signing.** Cheapest, but region-limited for individuals.
     - **OV certificate on an HSM.**
     - **SignPath Foundation.** Free for qualifying OSS.
     - No option avoids initial SmartScreen warnings.
   - **Linux.** Repository signing, outside electron-builder.
10. **Browser access to the local sidecar is optional.** If it is offered, it needs a Host check,
    authentication, CSRF defense and host-scoped cookies (e.g. `flock.localhost`). Not offering it
    leaves browser access to server mode alone.

## Open questions

- Does electron-builder's NSIS installer detect and stop a running sidecar process with a different
  executable name before overwriting it? This needs a test build. Not verified here.
- Does placing the Rust binary in `Contents/MacOS` via `extraFiles`, rather than in
  `Contents/Resources`, change anything for Squirrel.Mac updates or notarization? Verify with a
  signed test build.
- How do GNOME's background-app monitoring and the Flatpak Background portal treat an Electron app
  that keeps running with only a tray icon? Not researched in depth.

## Sources

[eb-scheme]: https://raw.githubusercontent.com/electron-userland/electron-builder/electron-builder%4026.18.1/packages/app-builder-lib/scheme.json
[eb-config]: https://www.electron.build/docs/configuration
[eb-autoupdate]: https://www.electron.build/docs/features/auto-update/
[eb-codesign]: https://www.electron.build/docs/features/code-signing/
[eb-notarize]: https://www.electron.build/docs/features/code-signing/notarization
[eb-mac]: https://www.electron.build/docs/mac
[eb-winpackager]: https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/packages/app-builder-lib/src/winPackager.ts
[e-utility]: https://www.electronjs.org/docs/latest/api/utility-process
[e-app]: https://www.electronjs.org/docs/latest/api/app
[e-tray]: https://www.electronjs.org/docs/latest/api/tray
[e-webrequest]: https://www.electronjs.org/docs/latest/api/web-request
[e-protocol]: https://www.electronjs.org/docs/latest/api/protocol
[e-security]: https://www.electronjs.org/docs/latest/tutorial/security
[e-autoupdater]: https://www.electronjs.org/docs/latest/api/auto-updater
[e-codesign]: https://www.electronjs.org/docs/latest/tutorial/code-signing
[forge-autoupdate]: https://www.electronforge.io/advanced/auto-update
[forge-squirrel]: https://www.electronforge.io/config/makers/squirrel.windows
[forge-mac]: https://www.electronforge.io/guides/code-signing/code-signing-macos
[forge-llms]: https://www.electronforge.io/llms.txt
[osxsign-util]: https://github.com/electron/osx-sign/blob/main/src/util.ts
[node-cp]: https://nodejs.org/api/child_process.html
[libuv-win]: https://github.com/libuv/libuv/blob/v1.x/src/win/process.c
[ms-job]: https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-jobobject_basic_limit_information
[ms-signing]: https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/code-signing-options
[pdeathsig]: https://man7.org/linux/man-pages/man2/PR_SET_PDEATHSIG.2const.html
[proc-environ]: https://man7.org/linux/man-pages/man5/proc_pid_environ.5.html
[proc-cmdline]: https://man7.org/linux/man-pages/man5/proc_pid_cmdline.5.html
[apple-bundle]: https://developer.apple.com/documentation/bundleresources/placing-content-in-a-bundle
[apple-notarize]: https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution
[apple-notarize-issues]: https://developer.apple.com/documentation/security/resolving-common-notarization-issues
[apple-helper]: https://developer.apple.com/documentation/xcode/embedding-a-helper-tool-in-a-sandboxed-app
[tn2206]: https://developer.apple.com/library/archive/technotes/tn2206/_index.html
[tokio-net]: https://docs.rs/tokio/latest/tokio/net/index.html
[html-origin]: https://html.spec.whatwg.org/multipage/browsers.html#concept-origin-tuple
[rfc6265]: https://www.rfc-editor.org/rfc/rfc6265#section-8.5
[owasp-csrf]: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
[chrome-lna]: https://developer.chrome.com/blog/local-network-access
[cve-ollama]: https://nvd.nist.gov/vuln/detail/CVE-2024-28224
[jupyter-sec]: https://jupyter-server.readthedocs.io/en/latest/operators/security.html
[syncthing-api]: https://github.com/syncthing/syncthing/blob/main/lib/api/api.go
[xdg-autostart]: https://specifications.freedesktop.org/autostart/latest/
[portal-bg]: https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.Background.html
[flatpak-electron]: https://docs.flatpak.org/en/latest/electron.html
[mailspring-proc]: https://github.com/Foundry376/Mailspring/blob/master/app/src/mailsync-process.ts
[mailspring-bridge]: https://github.com/Foundry376/Mailspring/blob/master/app/src/flux/mailsync-bridge.ts
[ollama-forge]: https://github.com/ollama/ollama/blob/e806184023/macapp/forge.config.ts
[ollama-index]: https://github.com/ollama/ollama/blob/e806184023/macapp/src/index.ts
[ollama-removal]: https://github.com/ollama/ollama/commit/54a76d3773
[mullvad-sock]: https://github.com/mullvad/mullvadvpn-app/blob/main/mullvad-paths/src/rpc_socket.rs
[mullvad-mi]: https://github.com/mullvad/mullvadvpn-app/blob/main/mullvad-management-interface/src/lib.rs
[keybase-platform]: https://github.com/keybase/client/blob/master/shared/constants/platform.tsx
[rd-server]: https://github.com/rancher-sandbox/rancher-desktop/blob/main/pkg/rancher-desktop/main/commandServer/httpCommandServer.ts

- Electron: [utilityProcess][e-utility], [app][e-app], [Tray][e-tray], [webRequest][e-webrequest],
  [protocol][e-protocol], [security checklist][e-security], [autoUpdater][e-autoupdater],
  [code signing tutorial][e-codesign]
- electron-builder: [v26 config schema][eb-scheme], [configuration (v27)][eb-config],
  [auto-update][eb-autoupdate], [code signing][eb-codesign], [notarization][eb-notarize],
  [mac][eb-mac], [winPackager.ts][eb-winpackager]
- Electron Forge: [auto-update][forge-autoupdate], [Squirrel.Windows maker][forge-squirrel],
  [macOS signing][forge-mac], [docs index][forge-llms]; [@electron/osx-sign `util.ts`][osxsign-util]
- Apple: [Placing content in a bundle][apple-bundle], [Notarizing macOS software][apple-notarize],
  [Resolving common notarization issues][apple-notarize-issues],
  [Embedding a helper tool in a sandboxed app][apple-helper], [TN2206 Code Signing in Depth][tn2206]
- Microsoft: [Code signing options for Windows app developers][ms-signing],
  [JOBOBJECT_BASIC_LIMIT_INFORMATION][ms-job]
- Node / libuv / Linux: [child_process][node-cp], [libuv `src/win/process.c`][libuv-win],
  [PR_SET_PDEATHSIG][pdeathsig], [proc_pid_environ(5)][proc-environ], [proc_pid_cmdline(5)][proc-cmdline]
- Standards and web security: [RFC 6265 §8.5][rfc6265], [WHATWG HTML origin][html-origin],
  [OWASP CSRF cheat sheet][owasp-csrf], [Chrome Local Network Access][chrome-lna],
  [CVE-2024-28224][cve-ollama], [Jupyter Server security][jupyter-sec]
- Linux desktop: [XDG Autostart][xdg-autostart], [Background portal][portal-bg],
  [Flatpak Electron guide][flatpak-electron]
- Rust: [tokio::net][tokio-net]
- Real-world code: [Mailspring process][mailspring-proc], [Mailspring bridge][mailspring-bridge],
  [Ollama forge config][ollama-forge], [Ollama index.ts][ollama-index], [Ollama removal][ollama-removal],
  [Mullvad socket path][mullvad-sock], [Mullvad management interface][mullvad-mi],
  [Keybase platform][keybase-platform], [Rancher Desktop command server][rd-server],
  [Syncthing api.go][syncthing-api]
