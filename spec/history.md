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
  "author": "did:key:z6MkCleia...",
  "authorKey": "dev-a1b2",
  "actorClass": "human",
  "message": "Scheduled for Tuesday.",
  "action": { "type": "accept" },
  "after": [ { "entry": "019fd1dc-2ed0-...", "hash": "sha256:..." } ],
  "sentAt": "2026-09-01T10:05:00Z",
  "proof": { "type": "ed25519-jcs-2026", "by": "did:key:z6MkCleia...",
             "key": "ed25519:...", "sig": "..." }
}
~~~

## Actions

`accept`, `decline`, `counter`, `start`, `ack`, `complete`, `fail`, `cancel`,
`confirm`, `dispute`, `postpone`, `reschedule`, `update`, `reassign`.

An entry with `action: null` is an ordinary comment.

`actorClass` records the class of the signing device, specified in the
autonomous actors document. It is taken from the identity document, never from
what the entry claims about itself: an entry that could assert its own actor
class would make human approval unverifiable.

# Immutability

**An entry carrying an action MUST NOT be removed or altered.** An entry with
`action: null` MAY be retracted, via `task.entry.retract`, and the retraction is
itself an entry, so the fact that something was retracted remains visible.

The asymmetry is deliberate. A comment is speech and can be withdrawn. An action
changed the state of a shared object, and other parties acted on that change;
removing it rewrites a history other people relied on.

Corrections are made by adding an entry, never by editing one. A correction that
overwrites is indistinguishable from tampering.

# Causal Chain

Each entry references the entries it follows, by identifier **and by content
hash**:

~~~
"after": [ { "entry": "019fd1dc-2ed0-...", "hash": "sha256:9c4f..." } ]
~~~

The hash covers the canonical form {{RFC8785}} of the referenced entry with proof
fields removed.

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

## Verification requires the author's keys

Verifying signatures requires the identity document of each author. For a
history whose authors are all at the verifier's own host, those documents are
local. For a federated task they are not, and a verifier that cannot resolve
them can check the chain but not the signatures.

Implementations SHOULD resolve unknown authors' identity documents through the
handle proof of the identity document and cache them. An implementation that
reports a cross-host history as unverifiable, when the only obstacle is an
unresolved key, has made the property useless exactly where it matters most.

# Derived Activity

An activity feed is a **projection** of entries, never a separate stored object.
Anything shown in an activity view MUST be derivable from the entries a party
holds.

The rule exists so that there is one source of truth. A stored activity log
alongside an entry chain drifts, and when the two disagree there is no principled
way to say which is right.

# Security Considerations

The chain protects against the host, which is the adversary that matters here: a
host holds the entries and could alter or withhold them. It does not protect
against an author who signs something false, which is out of scope for any
signature scheme.

A host that removes the tail of a history leaves no local trace, since the last
entry references only its predecessors. Detection in that case requires
comparing with another party's copy, which federation makes possible: the same
task exists at each participant's host, and their chains must agree.

Timestamps are asserted by the author and MUST NOT be treated as authoritative.
Ordering comes from the chain, not from `sentAt`.

# Privacy Considerations

The chain is history, and history is not deletable in place. The regulatory
document specifies how erasure works in a system with immutable history: the
container is destroyed, never an entry in the middle. Either the context exists
intact and auditable, or it ceases to exist entirely for that party. The two
properties coexist with no hidden exception.
