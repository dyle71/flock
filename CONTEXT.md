# flock

An e-mail client for people who run many mail accounts at once: it collects them in one place,
lets them be grouped, colored and marked as favorites, and works fully offline. Its name is a
flock of birds, the carrier pigeons of a dovecote; the vocabulary below may borrow from that
image where a term earns it.

## Language

**flock**:
The product as a whole: the server and every client that talks to it. Always written in lower
case.
_Avoid_: Flock (capitalized), the app, the mailer

**User**:
The person who owns Accounts in flock and sees them. In the local runtime mode there is exactly
one User; in server mode there are many, and none sees another's Accounts.
_Avoid_: Owner, member, customer

**Account**:
One mailbox at a mail provider, bundling how mail arrives and how it is sent, owned by exactly
one User. Its Identities are the sender addresses it may send as.
_Avoid_: Mailbox (that is the provider's word for the store), resource, transport, profile

**Identity**:
A sender an Account may send as: a name, an address and a signature. Belongs to exactly one
Account; an alias address of an Account is an Identity, never an Account of its own. Every
Account has exactly one default Identity. A reply goes out as the Identity the message was
addressed to, else as the default Identity of the message's Account; a new message written
outside any Account goes out as the User's default Identity, changeable while composing.
_Avoid_: Sender, alias, persona

**Loft**:
A group of Accounts a User forms, named after the dovecote that houses a flock of pigeons. An
Account sits in at most one Loft or in none; a Loft gathers Accounts, never Folders or other
Lofts. The User orders Lofts and Accounts in the sidebar by hand. Deleting a Loft leaves its
Accounts and their mail untouched, outside any Loft.
_Avoid_: Account group, group, flock (that is the product), folder group

**Folder**:
A container of messages inside an Account, as the provider keeps it, including the special
ones such as Inbox, Sent, Drafts, Archive, Trash and Spam. A message may sit in several Folders
of the same Account at once, the way a Gmail label works; it still counts once.
_Avoid_: Mailbox, collection, label (unless quoting a provider)

**Local folders**:
Folders that live only in flock and at no provider. A POP3 Account keeps its own; in addition
every User has one set of Local folders, outside any Account, into which messages from any
Account can be moved and kept.
_Avoid_: Archive (that is a special Folder), offline folders, on my computer

**Unified view**:
A view that shows one kind of special Folder, such as every Inbox, across several Accounts at
once: across the Accounts of one Loft, or across all Accounts. It holds no messages of its own;
a message in it appears once. Every kind of special Folder has one; the User hides those not
wanted, and only Inbox and Drafts show by default.
_Avoid_: Smart folder, unified mailbox, virtual folder

**Saved search**:
A search a User keeps under a name and opens like a Folder, over any combination of Lofts,
Accounts and Folders, or over all of them. It holds no messages of its own.
_Avoid_: Virtual folder, search folder, smart folder

**Thread**:
The messages of one conversation within one Account, gathered across all of its Folders, so a
reply the User sent shows next to the message it answers. A Thread never spans two Accounts.
_Avoid_: Conversation, discussion, chain

**Tag**:
A named, colored marker a User puts on messages. A Tag belongs to the User and means the same
across all of their Accounts; where a provider can store it with the message, it travels there.
_Avoid_: Label, keyword (that is the IMAP mechanism), category

**Star**:
The single marker that says a message matters, kept by the provider as the message's flagged
state.
_Avoid_: Favorite, flag, important

**Favorite**:
An entry a User pins to the top of the sidebar: a Folder, a Saved search or a Unified view.
Messages are never Favorites; they get a Star.
_Avoid_: Pin, bookmark, starred folder

**Color**:
One of the named colors of the Palette that a User gives a Loft, an Account, a Folder or a Tag.
A Folder without a Color of its own shows its Account's Color; an Account does not take its
Loft's Color. In a Unified view every message shows its Account's Color.
_Avoid_: Highlight, accent

**Palette**:
The fixed set of named Colors a User picks from, the same one klicker.ing uses: each Color comes
with its own fixed text color, so that labels stay legible on it. There is no free color choice.
_Avoid_: Color picker, theme, swatches

## Relationships

- A **User** owns zero or more **Accounts**; an **Account** belongs to exactly one **User**.
- An **Account** has one or more **Identities** and one or more **Folders**.
- An **Account** sits in at most one **Loft**; a Loft gathers Accounts, never Folders.
- A **User** has one set of **Local folders** outside every Account.
- A **Thread** belongs to exactly one **Account** and gathers messages from any of its Folders.
- An **Account** has exactly one default **Identity**; a **User** has one default Identity.
- A **Unified view** and a **Saved search** read messages from Folders; they own none.
- A **Tag** belongs to a **User**; a message carries zero or more Tags and at most one Star.

## Flagged ambiguities

- "Favorite" was used for messages as well as for Folders; resolved: Folders, Saved searches and
  Unified views are Favorites, a message gets a **Star**.
- "Group" was used for gathering Accounts; resolved: it is a **Loft**, and a Loft never gathers
  Folders. Cross-cutting collections are **Saved searches**.
- A Gmail label is a **Folder**, never a **Tag**: Tags belong to the User across all Accounts,
  labels belong to one Account.
