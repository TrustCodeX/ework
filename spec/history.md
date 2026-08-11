---
docname: draft-wilhelm-ework-history-00
title: "The e-work Protocol (EWP): Entries and Verifiable History"
abbrev: EWP History
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC8032,RFC8785,RFC8259
---

This document specifies the history model of the e-work Protocol (EWP). The unit
of history is the **entry**, an atomic signed object carrying a message and an
optional action. An entry that carries an action is immutable.

Entries are chained by content hash, so that a host that alters or withholds
history is detectable by any party holding a copy, without that party trusting
the host.

<!-- abstract -->

# Introduction

Most task systems model comments and state changes as separate things: a comment
stream, and an audit log of transitions. That separation loses the connection
between them, which is usually the part that matters. "Why was this marked
done?" is answered by the comment attached to the transition, and in a
two-stream model that attachment is a convention rather than a structure.

EWP makes the entry the unit: `{ message, action }`, where an ordinary comment
is an entry with `action: null`. The state change and its justification are the
same signed object, and cannot come apart.

## Conventions and Definitions

<!-- rfc2119 -->

# Entry Object

~~~
{
  "@type": "Entry",
  "uid": "019fd1dc-2ee1-7c61-b9fe-175808e9c969",
  "task": "019fd1d5-e84c-76a0-9961-f1ccbcead44c",
  "author": "contact:z6MkK7f3a2q9...",
  "authorKey": "dev-a1b2",
  "actorClass": "human",
  "message": "Scheduled for Tuesday.",
  "action": { "type": "accept" },
  "after": [ { "entry": "019fd1dc-2ed0-...", "hash": "sha256:..." } ],
  "sentAt": "2026-09-01T10:05:00Z",
  "proof": { "type": "ed25519-jcs-2026", "by": "contact:z6MkK7f3a2q9...",
             "key": "ed25519:...", "sig": "..." }
}
~~~

`author` identifies the signer **in the vocabulary of the group where the entry
is published**. In a personal group or a project circle it is the principal
identity. In a relationship group with an issuer it is the identifier of that
relationship's contact key, never the root `did:key`, and `proof.by` is the
contact key itself. A stable global identifier in an entry addressed to an issuer
would let two issuers join their records and undo the unlinkability the
per-relationship address exists to provide.

`message` is plain text or limited Markdown, with no HTML and no reference to a
remote resource. It MAY be empty when there is an action (acting without saying
anything is legitimate), and MUST NOT be empty when the policy requires a
justification for that action.

`anchor` (optional, a JSON Pointer) pins the entry to a field of the task, and
MUST belong to the same compartment as the entry: you do not comment on what
you cannot see.

**Atomicity by construction.** Message and action are fields of the same signed
object. There is no path by which the reason separates from the act, is
reordered, or is reattributed to another author. Receivers MUST NOT apply the
action while discarding the message, nor the reverse.

## Actions

`accept` (`slotId` when the payload offers options), `decline`, `counter`
(`task`: the proposed revision), `start`, `ack`, `complete` (optional
`result`), `fail`, `cancel`, `confirm`, `contest`, `snooze` (`until`),
`reschedule` (`due`, `start`), `update` (`task`: the revision), `reassign`
(`to`), `attach` (`attachments`: the list added), `detach` (`attachments`: the
`id`s removed).

An entry with `action: null` is an ordinary comment. An entry carries **at most
one action**: an operation that changes two things becomes two entries, which
is more honest than one entry of ambiguous meaning.

Third-party actions use a namespace of their own
(`com.example/action/inspect@1`). Clients MUST ignore an unknown action while
preserving the entry, so that the message and the authorship are not lost.

`actorClass` records the class of the signing device, specified in the
autonomous actors document. Receivers MUST resolve the signing key in the
author's current identity document and use only the class signed by the root
key there. The class carried inside the entry is display redundancy, MUST NOT
satisfy any policy, and a mismatch between the two MUST make the entry void.
Clients MUST display the class when it is not `human`.

## Attaching after creation

A task is born with `ework:attachments`, and that is almost never where the
evidence appears: the receipt exists after paying, the photo after fixing, the
duplicate bill after somebody asks. Attaching later is a deliberate act aimed
at the counterparties, not a side effect of another operation, so it is an
entry with an action and never a direct write to the object.

- `attach` adds. The attachments travel in `action.attachments`, and the blobs
  MUST be on the host before the entry arrives.
- Whoever MAY comment MAY attach: a participant who sees the entry's
  compartment. Attaching is NOT the offerer's privilege, and restricting it to
  them would restore the very problem this solves, because whoever does the
  work is precisely who holds the evidence.
- `detach` removes from the current list, by `id`. It deletes nothing: the
  `attach` entry stays in the history with its author, date and justification,
  and clients MUST show both.
- A comment's own attachments do NOT enter the task's list. The two places mean
  different things: the task's is the dossier whoever arrives later needs, the
  comment's is the evidence for that statement. A client MAY offer to promote
  one to the other, and promoting is a new `attach` entry, never a
  reinterpretation of the old one.

**The current list is derived.** The `ework:attachments` a client shows is the
birth list, plus the `attach` entries, minus the `detach` entries, in the order
below. Nobody rewrites the list in place. That gives for free what the raw list
never had: knowing who added each file, when, and under what justification.

**`update` MUST NOT prune what is not its own.** The offerer's revision carries
the whole task, and whoever applies it MUST preserve attachments added by other
participants' `attach` entries. Replacing the list with the envelope's would
erase someone else's contribution with no entry recording the erasure, which is
exactly the hole immutability closes for the rest of the history. An offerer who
wants a specific attachment gone uses `detach`, which stays written.

## Authority

A valid signature proves **who wrote** an entry. It never proves that the author
was **allowed to write it**. These are different questions, and conflating them
lets any legitimate participant sign another participant's transition.

Each action has a required role, drawn from the participant roles of the data
model document:

| Action | Signer MUST hold |
|---|---|
| `accept`, `decline`, `counter` | `owner` |
| `start`, `complete`, `fail` | `executor` |
| `ack` | whoever the notification reached, per the urgency document |
| `cancel` | `owner`, or `issuer` on its own unaccepted offer |
| `confirm`, `contest` | whoever `ework:execution.confirmedBy` designates |
| `snooze` | `owner` (local effect) |
| `reschedule` | `owner`, `executor` or `issuer` |
| `update` | `issuer` |
| `reassign` | `owner` |
| `attach` | anyone who may comment |
| `detach` | whoever signed the corresponding `attach`, or the `owner` |

Receivers MUST resolve the author against the task's current participants and
verify the required role **before** applying any effect. An entry whose action
requires a role the author does not hold MUST be recorded as a **void entry**:
it enters the history, with author, message and signature intact, and changes no
state. Clients MUST show void entries as refused attempts, naming the missing
role, because losing the trail of who tried to act outside their role loses the
abuse signal itself.

The `owner` MAY widen the authority of an action per task, in
`ework:execution.maySign`, and MUST NOT narrow it below this default nor grant
itself a role it does not hold. Seeing a compartment authorises commenting, never
transitioning.

Without this rule an issuer signs `accept` on its own offer and produces a signed
record that the charge was accepted, and a supplier signs `complete` to release
its own delivery. Both entries are cryptographically impeccable, which is exactly
why the defence cannot live in the signature.

# One Envelope Type

Entries travel in `task.entry`. Accepting, declining, completing and snoozing
are no longer distinct envelope types; they became values of `action`.

This is not mere tidying: the envelope type sits **in the clear** in the
routing header, so separate types told hosts that a task had been completed,
declined or postponed. With a single type, the host sees that there was
activity and nothing more. A metadata leak disappears by simplification, and
the exhaustive list of what a host can observe shrinks.

# Required Justification

The task's execution policy declares when a message is mandatory:

~~~
"ework:execution": { "messageRequired": ["decline", "fail", "contest", "cancel"] }
~~~

Possible values are a list of action types, `[]` (never), or `["*"]` (every
action). Receivers MUST reject an entry with no message when that task's policy
requires one for that action, and the rejection is the peer's, not the host's,
which reads nothing.

This is what implements the question "why do you want to mark this complete?":
the client asks because the action requires it, and the answer becomes the
record. Declining without saying why remains acceptable in a relationship with
an issuer and is usually unacceptable inside a project.

# Audience

An entry's audience is the cryptographic group it is published to, reusing
compartments rather than inventing a parallel permission system:

| Intent | Group | Who reads it |
|---|---|---|
| Personal note | the identity's personal group | only you, on all your devices |
| Interaction with the issuer | the relationship's group | you and that issuer |
| Project work | the project's compartment | the members of that compartment |

The consequences come for free: the note "pay this after payday" never reaches
the company, and the price discussion in the financial compartment never
reaches the fitter. Clients MUST show who an entry goes to **before** sending,
because getting the audience wrong is the most likely leak in the system.

An entry carrying an action goes to the group where the action makes sense: a
declined offer goes to the relationship's group, because the issuer needs to
know; a completed project task goes to its compartment. A personal note uses a
one-member group, the degenerate case of compartments, which implementations
MUST make cheap because it is the most common of all.

# Mentions

`mentions` lists identities to notify. A mention generates a notification and
MUST NOT raise urgency above what that relationship's consent allows, or
mentioning would become a back door for forcing attention. Mentioning someone
outside the entry's group makes no sense, and clients MUST warn when it
happens.

# Immutability

**An entry carrying an action MUST NOT be removed or altered.** An entry with
`action: null` MAY be retracted, via `task.entry.retract`, and the retraction is
itself an entry, so the fact that something was retracted remains visible.

The asymmetry is deliberate. A comment is speech and can be withdrawn. An action
changed the state of a shared object, and other parties acted on that change;
removing it rewrites a history other people relied on.

Corrections are made by adding an entry, never by editing one. A correction that
overwrites is indistinguishable from tampering.

## Retraction and editing

**An entry carrying an action can NEVER be removed or edited.** This is not
"conforming peers should keep it": it is a normative prohibition, and an
implementation that offers to delete an entry with an action is
non-conforming. It holds for every action, including one by a project owner or
a host administrator.

The reason is auditability: a history has evidentiary value only if nobody can
prune it after the fact. A record where the author deletes what they did once
the conversation heats up serves none of the purposes it exists for. It holds
equally for the message attached to the action: removing the justification and
leaving the action would be worse than nothing, because it would produce a
record that looks complete and lies by omission.

Correcting an entry with an action means **adding**, never subtracting: a new
entry with `action: null` and `after` pointing at the original, saying what was
wrong. Undoing the effect is also a new action (`start` after `complete`), and
the history shows both, in the order they happened.

`task.entry.retract` exists only for entries with `action: null`. Conforming
peers hide and stop syncing that content, and even then **retraction is a
request, not a guarantee**: the interface MUST say so in those words, because
whoever already received it already has it, and deleting remotely in a system
with end-to-end encryption and third-party clients is a promise nobody keeps.
Editing a comment without an action follows the same principle: a new entry
replacing the previous one, with the original marked as edited, never a silent
rewrite.

The rule in one line, which clients MUST reflect in the interface: **what you
said can be taken back; what you did cannot.**

# Causal Chain

Each entry references the entries it follows, by identifier **and by content
hash**:

~~~
"after": [ { "entry": "019fd1dc-2ed0-...", "hash": "sha256:9c4f..." } ]
~~~

The hash covers the canonical form {{RFC8785}} of the referenced entry,
`proof` included.

The first entry of a task anchors on the id and the hash of the envelope that
originated it (`task.offer`, or the local creation), closing the chain at its
origin.

This gives two properties that a signature alone does not:

**Alteration is detectable.** Changing the content of an entry changes its hash,
so the reference in the following entry no longer matches. A verifier walking
the chain finds the break and can name the altered entry.

**Withholding is detectable.** An entry referencing a predecessor that is absent
from the history the verifier received proves that something was removed or is
being withheld, even though the verifier cannot know what it said.

Verifiers MUST report the two conditions distinctly, because they mean different
things: a broken hash means content was altered, and a missing reference means
content was removed or retained.

## Ordering and the clock

Rendering order is the topological order of the causal graph, with ties broken by
the entry's own canonical hash, ascending. Ties MUST NOT be broken by `sentAt`.

A timestamp is a field the author chooses and signs, so breaking ties by it hands
the apparent order to whoever wants to fabricate it: omit from `after` whatever
should not appear first, and date the entry in the past. An entry's own hash is
not chosen freely, which is why it is the criterion. `sentAt` stays in the object
as display information and decides nothing.

Receivers MUST reject an entry whose `sentAt` is more than five minutes ahead of
local receipt time, and MUST record the local receipt time. Wherever last-writer-
wins compares times, the compared value MUST be `min(sentAt, receivedAt)`.
Otherwise a revision dated in the future wins permanently, and no legitimate
later edit can override it.

## Fork detection

A chain that only looks backwards detects pruning and alteration. It does not
detect a withheld suffix. Because `after` declares only what its author knew, a
host that withholds an entry **together with every later entry referencing it**
hands each victim an internally consistent suffix: nothing points at a missing
parent, and the gap is formally indistinguishable from legitimate concurrency.
The result is a server showing different histories to different members without
breaking a single hash.

Closing this requires looking sideways:

- Every entry MUST include in `after` the current known head of each co-author of
  the group, not only the entries it replies to. Withholding one entry then
  requires withholding everything anyone else wrote afterwards, which a host
  cannot sustain without stopping the task entirely.
- Participants MUST periodically publish a **signed head** to the group, the pair
  (task, hash of the most recent entry they know), even with nothing to say.
  Clients MUST compare received heads against what they hold and MUST flag as a
  possible fork any head referencing an entry they do not have.
- Verifiers MUST refuse a chain whose hash does not match, and MUST flag gaps
  as suspected withholding. This was a SHOULD: the evidentiary value claimed
  for the history depends on the gap being seen, and optional detection does
  not sustain it.

Signed heads are new metadata for the host, which learns the cadence of these
publications. The cost is accepted because what it buys is the detection of an
attack that otherwise leaves no trace at all.

## Verification requires the author's keys

Verifying signatures requires the identity document of each author. For a
history whose authors are all at the verifier's own host, those documents are
local. For a federated task they are not, and a verifier that cannot resolve
them can check the chain but not the signatures.

Implementations SHOULD resolve unknown authors' identity documents through the
handle proof of the identity document and cache them. An implementation that
reports a cross-host history as unverifiable, when the only obstacle is an
unresolved key, has made the property useless exactly where it matters most.

# Auditability

A task's list of signed entries **is** the audit artefact. Exporting it
produces a record a third party can verify without trusting anyone: each line
carries author, key, actor class, action, justification, attachments and
signature, and the causal chain proves the order.

What gives it evidentiary value is immutability combined with the chaining:
an entry with an action is neither deleted nor edited, and verification is
mechanical, since recomputing the chain's hashes detects removal, editing and
withholding without trusting whoever exported it. An audit the audited party
can rewrite is not an audit.

Implementations MUST offer export of a task's history in a verifiable format,
and verification MUST NOT depend on the host that stored it.

# Derived Activity

An activity feed is a **projection** of entries, never a separate stored object.
Anything shown in an activity view MUST be derivable from the entries a party
holds.

The rule exists so that there is one source of truth. A stored activity log
alongside an entry chain drifts, and when the two disagree there is no principled
way to say which is right.

Changes already recorded in other signed operations (a property edit in the
owner's box, a member joining or leaving a compartment) are derived locally by
the client from what it already holds, and MUST NOT be transmitted as
assertions. That eliminates an entire class of forgery: nobody can insert a
"so-and-so changed the deadline" that never happened, because no channel exists
through which that assertion could enter.

Clients MUST merge entries and derived activity into a single timeline,
distinguishing the two visually.

# Anti-abuse

An entry is a delivery channel, so it inherits the rules of the anti-abuse
document:

- An issuer's entry MUST stay within the granted purpose. Advertising in a
  comment is the same violation as advertising in a task.
- No references to remote resources, for the same reason as in tasks: the end
  of the tracking pixel.
- Entries count against the credential's rate limit: they are not a parallel
  lane for flooding someone.
- An entry NEVER creates a new delivery right: whoever cannot send a task
  cannot comment on it either.

# Security Considerations

The chain protects against the host, which is the adversary that matters here: a
host holds the entries and could alter or withhold them. It does not protect
against an author who signs something false, which is out of scope for any
signature scheme.

A host that removes the tail of a history leaves no local trace from backward
references alone, since the last entry references only its predecessors. Signed
heads close this: each participant periodically states the head it knows, so a
removed tail contradicts a head someone else published.

**Authenticity is not authority.** The signature answers who wrote an entry; the
role answers whether that party could write it. While the two were conflated, any
legitimate member of a group could sign any other member's transition, producing
an entry valid by every criterion this specification listed, and the audit
artifact forged itself with an impeccable signature. Conformance suites MUST test
that an action signed without the required role enters the history and changes no
state.

Atomicity is structural, not an implementation rule: message and action are the
same signed object. Conformance suites MUST test that an entry carrying an
action and a message is not accepted by half, and MUST test the hash chain from
the outside: tampering is a detectable chain break, not a matter of faith in
someone else's implementation.

An unknown action needs to be ignored with the entry preserved; discarding the
whole entry would let a peer make someone else's record disappear by inventing
an exotic action.

Timestamps are asserted by the author and MUST NOT be treated as authoritative.
Ordering comes from the chain, not from `sentAt`.

# Privacy Considerations

The chain is history, and history is not deletable in place. The regulatory
document specifies how erasure works in a system with immutable history: the
container is destroyed, never an entry in the middle. Either the context exists
intact and auditable, or it ceases to exist entirely for that party. The two
properties coexist with no hidden exception.

Collapsing the envelope types reduces metadata: hosts no longer learn whether
the activity was a completion, a refusal or a postponement. The most likely
error remains human, writing to the wrong audience, and the defence is in the
interface. Entries inherit the group's encryption, so hosts see only one more
envelope of size X for group Y.

# Open Questions

1. An entry anchored on a sealed field: allow it for whoever is in the narrow
   compartment, or forbid it for simplicity?
2. A normative limit on message size and on the number of entries per task.
3. A draft entry synced across a person's own devices: useful, or a
   complication?
4. How to present a timeline holding entries from different compartments to
   someone who belongs to several, without confusing each entry's audience.
5. Is an explicit `note-to-self` action type worth having, or is a personal
   note always a null action in the personal group?
