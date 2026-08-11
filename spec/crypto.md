---
docname: draft-wilhelm-ework-crypto-00
title: "The e-work Protocol (EWP): Cryptography and Encryption Modes"
abbrev: EWP Crypto
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC9420,RFC8032,RFC8785,RFC8446
---

This document specifies the cryptographic layer of the e-work Protocol (EWP):
the signature scheme, the two encryption modes, the mapping of protocol groups
onto MLS {{RFC9420}} groups, and an exhaustive statement of the metadata a host
observes in each mode.

<!-- abstract -->

# Introduction

EWP defines two modes and requires that a host declare which one it operates.
The distinction is invisible in a user interface, so a protocol that allowed it
to be implicit would let every implementation claim the stronger one.

## Conventions and Definitions

<!-- rfc2119 -->

# Modes

**End-to-end encrypted mode.** Content is encrypted to a group; the host stores
and forwards ciphertext it cannot read. This is the target state and the default
the protocol is designed around.

**Assisted mode.** The host reads content. This exists because it is what makes
several things possible before group encryption is deployed: server-side search,
recovery when a user loses every device, bridges to systems that do not speak
EWP, and simpler operation for a host serving people who prefer that trade.

A host MUST advertise its mode in the discovery document. A client MUST show the
mode to the user. **An implementation MUST NOT describe assisted mode as
providing confidentiality against the host operator**, because that is precisely
what it does not provide.

# Signatures

Signing is Ed25519 {{RFC8032}} over the canonical form {{RFC8785}} of the object
with proof fields removed. This applies uniformly: identity documents, consent
objects, entries, and envelopes.

Independent implementations MUST produce byte-identical canonical forms.
Canonicalisation is the part of a signature scheme that fails silently in
practice, and the only reliable test is cross-implementation verification.

# Groups

In end-to-end encrypted mode, each context is an MLS {{RFC9420}} group.

| Context | Members | Purpose |
|---|---|---|
| Personal | All devices of one identity | Multi-device synchronisation |
| Relationship | Two parties | Everything in one consent relationship |
| Project compartment | Members of one circle | Compartmentalisation |

MLS provides forward secrecy, post-compromise security, and efficient membership
change. Adding a member is a group operation; removing one rekeys the group, so
the removed member cannot read subsequent content.

The mandatory ciphersuite of v0.1 is `MLS_128_DHKEMX25519_AES128GCM_SHA256_Ed25519`.

KeyPackages for each device are published on the identity's host, which serves
them through `Group/fetch`; that is what makes it possible to add someone who
is offline. The `group.commit`, `group.welcome`, `group.proposal` and
`group.message` envelopes carry MLS across federation. Hosts act as a blind
Delivery Service: they order commits per group and replicate them, and are
never members.

Whether history is re-shared with a joining member is a choice of the project,
visible to every member. Both behaviours are legitimate and they have
different consequences.

# History for New Members

MLS gives no retroactive access. EWP defines **snapshot re-sharing**: when a
member joins a compartment whose `historyForNewMembers` policy is `reshare-all`,
the default, the existing member who performed the Add MUST send the group, in
the new epoch, a snapshot of the state **of that compartment**: tasks, metadata,
and the keys of the referenced attachments. It is an explicit act, auditable in
the project log. With `from-join`, nothing is re-shared.

The snapshot MUST NOT contain a task, a metadatum or an attachment key belonging
to a compartment the destination group does not own. Groups are per compartment,
never per project, and whoever performs the Add MUST clip the snapshot to the
compartment of the group they publish it in.

The distinction carries the whole guarantee: were the snapshot scoped to the
project, adding a supplier to the narrow circle would hand them the entire
project, attachment keys included, with no adversary at all and a perfectly
conforming client.

Implementations MUST treat re-sharing as a per-compartment operation even where
the interface presents it as "add to the project", and the conformance suite
MUST test that adding a member to a narrow compartment, in a project that also
has a wide one, lets nothing from the wide compartment cross.

# Group Membership Authority

The claim that a malicious host cannot forge membership depends entirely on an
authorisation policy existing. The policy is normative and per group type:

| Group | MAY propose Add | MAY propose Remove |
|---|---|---|
| Personal | any already verified device of the same identity | same |
| Relationship | each side, its own devices only | each side its own; either side ends the group |
| Project | an identity holding `owner` or `admin` | same |

Members MUST validate the proposer's authority **before** accepting the commit,
and MUST reject an epoch resulting from an Add or Remove proposed without it. In
a relationship group, one side MUST NOT add a device belonging to the other: a
cross Add is indistinguishable in the tree from a legitimate new member, and is
the shortest path for a host to insert its own key.

Validation uses the compartment's current Project object, and the role change
that authorises an Add MUST have been published before the commit that uses it.
A role granted and used in the same instant keeps the other members from seeing
the grant in time. The case where the Project and the tree diverge is handled
in the compartments document, where the reconciliation of the two lists is
normative.

# Content and Attachments

Application messages SHOULD be padded to the size classes of the core document
before encryption: the host sees the class, never the exact size, and a bare
acceptance stops being distinguishable from a counter-proposal carrying a
revision.

Each attachment blob is encrypted with a key of its own (AEAD A256GCM, unique
nonce); the blob stored and transferred is the ciphertext, and `enc.keyWrapped`
in the Task carries the key, protected by the group. Re-sharing an attachment
with another group means re-wrapping the key, without re-uploading the blob.

Status for the issuer travels as an entry in the relationship group, signed by
the user's identity, with content minimised by the `statusSharing` policy: the
issuer receives the milestone, not the user's box.

# Cryptographic Agility

Every group declares its ciphersuite. The mandatory suite of v0.1 is
`MLS_128_DHKEMX25519_AES128GCM_SHA256_Ed25519`. Implementations MUST support
the coexistence of suites: new groups are born on the best suite available, and
migrating an existing group means recreating it, reusing the epoch mechanics
that already exist. Implementations MUST NOT invent a suite outside the MLS
registry.

A post-quantum hybrid suite, combining ML-KEM with X25519, is the target as soon
as it stabilises in the MLS ecosystem. Stating this here is not a promise of a
date; it is a statement that the design must not make the change hard.

# Assisted Mode and Shared Content

Granularity is per box, per collection. Enabling it is an explicit act of the
owner, with the consequence stated in the interface, and the state is visible
in the session and auditable. Mechanically, the client adds a **device of the
host** to the personal group of that collection: the host becomes a member,
with a key of its own, auditable in the MLS tree, so the downgrade is never
invisible. That membership is what enables server-side `Task/query`, hosted
rules, server-side escalation, and gateways that require reading. Reverting is
a Remove of the host's device, with a new epoch. What the host saw while a
member, it saw.

Assisted mode MUST NOT reach content of a group the identity does not solely own.
Clients MUST exclude from the hosted index every task whose compartment belongs to
a project with other members, and the host MUST NOT accept such an object for
indexing. Where the owner insists on downgrading a collection holding shared
content, the client MUST publish a signed entry in each affected project's log
declaring the downgrade.

Per-box granularity was misleading on its own: a member's box carries the tasks of
the circles they belong to, and hosted query requires the host to read decrypted
tasks. One member enabling assisted mode was enough for their host to see an
entire circle in the clear, with no one else in the circle able to know. Downgrade
is never silent, and it was silent to everyone except the person who enabled it.

# Metadata Visible to the Host

This section is exhaustive by intent. A privacy claim that omits what leaks is
worse than no claim.

In **end-to-end encrypted mode** the host observes:

- Which addresses exchange envelopes, and in which direction.
- Timing and frequency of exchanges.
- Approximate envelope size, reduced by padding to size classes.
- Envelope type, since it is in the header and routing depends on it.
- The opaque group id and the epoch number of `group.*` envelopes. In a project
  with compartments this lets the host infer that some members participate in
  fewer groups than others, and the existence of a sealed section in a task is
  visible to the members of the wide compartment.
- The coarse `urgencyHint`, where present.
- The public identity document.
- For scheduled release, the pair (destination, release time), with neither
  content nor reason.
- Blob sizes and the set of hosts to which each blob has been granted.
- The cadence of the signed heads of the history: the host sees that a
  participant published a head, for which group and when, and sees neither the
  task it refers to nor what it chains. This is new metadata, knowingly
  accepted, because what it buys is fork detection, which without it leaves no
  trace at all.

In **assisted mode** the host additionally observes everything above plus the
content: task titles, descriptions, payloads, attachments, and entries.

Padding to size classes reduces the size channel. The remaining channels are
inherent to store-and-forward and cannot be removed by any cryptographic
measure, only by mixing or cover traffic, which EWP does not specify.

Any addition to this list requires a privacy review. Mitigations studied and
**not** promised in v0.1: blob padding, temporal delivery batches, and sender
obfuscation on the final hop (sealed sender).

# Key Management

Root keys are derived from a recovery kit and stay offline. Device keys live on
devices. Contact keys derive relationship addresses. The layering, the rotation
authority, and the rule that a compromised key never authorises its own
replacement are specified in the identity document.

**Credential presented in a group:** in a relationship group, the user's
devices present the contact key of that relationship, never the root. The
issuer validates who it is talking to without learning the root identity, and
two issuers cannot correlate the same user. In a personal group, and in a
project with invited people, the credential is the main identity.

Rotating a contact key ends the corresponding relationship group; migrating the
relationship creates a new group under the new key, and history follows the
re-sharing rules above.

Backup of group keys is what makes recovery possible after losing every device,
and it is also the point at which an implementation can accidentally hand the
host everything. A key backup encrypted to the recovery kit preserves the
property; a backup the host can decrypt does not, and an implementation offering
the latter MUST describe it as assisted mode regardless of what else it does.

**The backup holds the present, not the archive.** It MUST contain only the
current epoch secrets and the most recent snapshot, and the host MUST replace the
previous backup when accepting a new one, retaining no earlier versions. Clients
MUST rewrite the backup on every epoch change affecting them, and MUST offer its
destruction as a user action with immediate effect.

Otherwise the host accumulates years of ciphertext and stores beside it the key
that reopens all of it, so whoever obtains the kit once reads the whole archive
retroactively. The kit is paper held by non-technical people, which makes that
exposure a question of when rather than whether. Limiting the backup to the
present does not remove the risk; it reduces it to what a device would already
hold. This interacts directly with cryptographic erasure in the compliance
document, which only erases when no copy of the key survives in a backup.

## Ciphersuite floor

The mandatory suite is the floor. Implementations MUST NOT create or join a group
whose suite is weaker, and MUST reject a Welcome proposing one, even if that suite
appears in the MLS registry. "Best available suite" means the strongest all
members advertise, and the choice MUST be deterministic from the KeyPackages
rather than the preference of whoever creates the group. Downgrading an existing
relationship MUST require an explicit act from both sides, and clients MUST
present the change with its consequence written out.

Without a floor and without a selection rule, whoever creates a group chooses
everyone else's cryptography, and mandatory coexistence alone obliges the other
side to accept whatever is proposed.

# Security Considerations

The protocol's confidentiality claims hold only in end-to-end encrypted mode.
Every statement about compartments, sealed sections, and content privacy is a
statement about that mode, and this document is the place where that dependency
is recorded once rather than repeated in each document.

Commits are ordered by the Delivery Service. A malicious host can delay or
reorder delivery, a denial of service, but cannot forge membership, because an
Add requires the signature of a member authorised under the membership policy
above.

Signature verification requires resolving the signer's identity document.
Implementations MUST verify that the key carried in a proof belongs to the
declared signer, and MUST NOT accept a proof on the basis of the embedded key
alone. The embedded key exists so that verification can proceed before
resolution, not so that resolution can be skipped.

# Privacy Considerations

The exhaustive metadata list above is the honest bound on what EWP protects. A
host that carries only ciphertext still builds a social graph from routing
information. Distributing addresses across hosts, specified in the identity
document, partitions that graph, and it is the only mechanism the protocol
offers against it.

# Open Questions

1. Periodic epoch rotation without membership events, a key age limit? The
   backup rule, which keeps only the current epochs, makes that rotation more
   valuable: each rotation shrinks what a captured backup reopens.
2. Multi-device for an enterprise issuer, a fleet of servers as "devices": the
   practical limits of tree size.
3. Role validation requires every member to know the compartment's current
   Project. How does a member who just joined validate the Add that added
   them, before they had the Project to check the proposer's role against?
4. Clipping the snapshot per compartment calls for a snapshot format that
   declares which compartment it belongs to, so the receiver can refuse the
   excess instead of trusting the sender. That format is not yet specified.
5. The downgrade entry that assisted mode requires publishing in the affected
   projects is the first time the state of a personal box becomes a recorded
   fact in someone else's project. Does it deserve an entry type of its own,
   or is it a comment with a null action?
