# 0002. An Account bundles receiving and sending; Identities belong to one Account

- Status: Proposed
- Date: 2026-11-11

Decided in
[Domain model: accounts, groups, colors, favorites, tags and identities](https://github.com/dyle71/flock/issues/8).

## Context

flock takes Thunderbird and KMail as its feature lodestar, and both cut the model of a mail
account the same way at heart: receiving and sending are separate things.

- Thunderbird keeps an account as an incoming server with its identities; outgoing SMTP servers
  are a list of their own, and each identity picks one of them
  (`docs/research/thunderbird-features.md`, TB-ACC-07, TB-ID-05).
- KMail keeps three independent lists: receiving accounts (Akonadi resources), sending
  transports (MailTransport) and identities (`docs/research/kmail-features.md`, Accounts and
  Identities).

flock v1 supports IMAP, SMTP and POP3 with OAuth2 for Google and Microsoft, and its model must
leave room for JMAP and Exchange later. At Gmail, Microsoft 365, a JMAP provider and an Exchange
server, receiving and sending belong to one sign-in and one mailbox; a separate list of sending
servers would only mirror that pairing by hand.

The maintainer chose among three cuts on 2026-11-11: Thunderbird's (an Account with Identities,
sending servers separate), KMail's (three independent lists), or one Account bundling receiving
and sending with its Identities.

## Decision

An **Account** is one mailbox at a provider together with its way of sending, owned by exactly
one User. Its **Identities**, each a name, an address and a signature, belong to exactly that
Account; an alias address is an Identity of its Account, never an Account of its own. Every
Account has exactly one default Identity. flock keeps no list of sending servers apart from the
Accounts.

## Consequences

- An Account of a later kind, JMAP or Exchange, fits the same shape: one sign-in, one mailbox,
  one way of sending.
- One credential set per Account covers both directions where the provider uses one sign-in;
  where it does not, as with some IMAP and SMTP hosts, the Account holds both.
- A User cannot send through one Account's server as another Account's Identity, and cannot
  share one SMTP server between Accounts; each Account repeats its own sending settings.
- Importing from Thunderbird or KMail has to resolve a shared SMTP server or a free-standing
  identity into the Account it is used with.
- The terms are fixed in `CONTEXT.md` (**Account**, **Identity**).
