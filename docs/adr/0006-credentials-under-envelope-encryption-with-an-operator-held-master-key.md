# 0006. Credentials under envelope encryption with an operator-held master key; an optional Master password in Local mode

- Status: Proposed
- Date: 2026-11-11

Decided in [Credential storage for mail accounts](https://github.com/dyle71/flock/issues/10).

## Context

flock's backend syncs every Account completely and keeps syncing while no client is connected
([0003](0003-one-binary-two-runtime-modes.md)). To do so it must present each Account's
provider credentials — passwords for receiving and sending, OAuth2 access and refresh tokens —
without a User at hand. In Server mode an Admin must never see another User's credentials, and
the API is public, so credentials must never travel back out of it
([0004](0004-a-public-stable-rest-api-code-first-openapi-in-server-mode.md)).

Two models were weighed for Server mode on 2026-11-11:

- a key derived from each User's sign-in password: the operator can read nothing, but every
  User's sync pauses after each server restart until that User signs in again, and an Admin's
  password reset loses the User's credentials;
- a master key held by the server: sync runs at all times, at the price that whoever controls the
  host and the key can decrypt the credentials.

On self-hosted servers the Admin and the host operator are usually the same person. Thunderbird
keeps credentials readable unless its optional Primary Password is set
(`docs/research/thunderbird-features.md`). The maintainer chose the server-held key, and for
Local mode an optional Master password on top of the OS keyring.

## Decision

- **Envelope encryption in both modes.** Credentials are encrypted with a per-User data key; each
  data key is encrypted with one master key. flock's own secrets — Access tokens and session ids —
  are stored hashed, not encrypted.
- **Server mode** reads its master key from the file named by `FLOCK_MASTER_KEY_FILE`, created by
  `flock keygen`. Without it Server mode does not start, and it never writes the key into its data
  directory. The specification states that the Admin sees no credentials through flock, while the
  host operator holding the key can decrypt them. There is no Master password in Server mode.
- **Local mode** keeps the master key in the OS keyring. Without a keyring it keeps it in a file
  with mode `0600` in the config directory and shows a persistent notice.
- **The Master password** is opt-in, Local mode only. When set, the master key is wrapped with a
  key derived from it by argon2id, and nothing stays in the keyring. It guards the credentials
  only, not mail or the search index. Until it is entered after a start, stored mail stays readable
  and no Account syncs or sends. Forgetting it resets only the credentials. There is no auto-lock.
- **Credentials are write-only**: neither API nor UI ever returns a stored password or token, only
  whether one is set.
- `flock rotate-master-key` re-wraps every data key under a new master key; deleting a User
  destroys their data key.

## Consequences

- Sync in Server mode survives restarts without any User signing in.
- The operator's control of the host and the key file is a trust boundary flock does not close;
  the documentation says so.
- The key file must be backed up apart from the data directory; losing it loses every credential,
  not the mail.
- A deleted User's credentials stay unreadable in old backups, because their data key is gone.
- With a Master password set, a desktop start without entering it shows stored mail but fetches
  and sends nothing.
- Whole-store encryption of mail, metadata and the search index is left to the storage decision.
- The term is fixed in `CONTEXT.md` (**Master password**).
