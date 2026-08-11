---
docname: draft-wilhelm-ework-compartments-00
title: "The e-work Protocol (EWP): Compartments and Sensitive Data"
abbrev: EWP Compartments
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC9420,RFC8259
---

This document specifies compartmentalisation in the e-work Protocol (EWP): how a
project shared among several parties limits what each of them can see, at the
granularity of the task and of the individual field.

The central rule is stated once and applies throughout: **visibility is a key,
never a permission**. A compartment is a cryptographic group, and a party
outside it cannot decrypt what belongs to it. Implementations that enforce
visibility at the server, without encryption, MUST say so, because the
difference matters to the user and is invisible in the interface.

<!-- abstract -->

# Introduction

A kitchen renovation involves the owner, a carpenter, an installer, and a
supplier. All of them share a project. None of them should see everything: the
installer needs the delivery date and has no business knowing the payment
schedule; the supplier needs the order and not the labour costs.

Access-control lists express this and fail in a specific way: they are a promise
by whoever holds the data. In a federated protocol the data sits on a host the
user does not operate, so a promise by that host is worth what the host's
operator is worth.

EWP therefore makes a compartment a cryptographic group. What the installer
cannot see, he cannot decrypt.

## Conventions and Definitions

<!-- rfc2119 -->

**Project.** A container grouping tasks and participants.

**Compartment**, presented to users as a **circle**: a set of participants
sharing a key. Everything addressed to that compartment is encrypted to that
key.

**Sealed section.** A part of an otherwise shared task, encrypted to a narrower
compartment.

**Public milestone.** A neutral task in a wide compartment, used as proxy for a
dependency on a task in a narrow one.

# Honesty Requirement

Implementations operating in assisted mode, where the host enforces visibility
rather than encryption, **MUST NOT present that arrangement as a privacy
guarantee.** They MUST state, at the point where the user configures visibility,
that the host operator can read everything.

This requirement exists because the interface for the two modes is otherwise
identical, and a user cannot tell them apart by looking. An implementation that
says "only the financial circle can see this", when in fact the server is
choosing to hide it, has told the user something false about who can read their
data.

The same honesty applies within a single client: presentation restrictions with
no key behind them (sorting, filtering, hiding on screen) are **hints**, and the
interface MUST label them as such, without security language.

# Project Object

~~~
{
  "@type": "Project",
  "uid": "019fd0...",
  "title": "Kitchen",
  "members": { "did:key:z6MkOwner...": { "role": "owner" },
               "did:key:z6MkBeta...":  { "role": "member" },
               "did:key:z6MkGamma...": { "role": "member" } },
  "compartments": {
    "general":   { "name": "Everyone on the project", "members": ["*"] },
    "financial": { "name": "Financial", "members": ["did:key:z6Mk..."] },
    "beta":      { "name": "Supplier Beta", "members": ["did:key:z6MkOwner...", "did:key:z6MkBeta..."] },
    "gamma":     { "name": "Supplier Gamma", "members": ["did:key:z6MkOwner...", "did:key:z6MkGamma..."] }
  },
  "defaultCompartment": "general",
  "sensitivityPolicy": { "money": "financial", "health": "financial",
                          "credential": "none" },
  "historyForNewMembers": "reshare-all",
  "statusVisibility": "members",
  "created": "2026-08-05T00:00:00Z"
}
~~~

Every project has a `general` compartment containing all members. Tasks default
there.

`members` maps each member to a role: `owner`, `admin`, who manages members and
compartments, `member`, or `observer`.

`historyForNewMembers`: `reshare-all`, the default, gives a new member the
snapshot, the kitchen furniture case; `from-join` shows the newcomer only what
happens from then on.

A compartment **by subject** (financial, legal) and **by relationship** (the
owner plus one supplier) are the same mechanism. The second is what guarantees
that Supplier Beta does not see Gamma's price, a common commercial requirement
that no tool solves today.

Compartments are created, their membership changed, and `sensitivityPolicy`
altered, by holders of the `owner` or `admin` role, under the Add and Remove
proposal policy of EW-RFC 0006 §2.1. Every change is an event in the project
log, visible to the members of that compartment.

## What Each Member Receives

**The `compartments` object delivered to a member MUST contain only the
compartments that member belongs to, and `sensitivityPolicy` MUST omit any entry
whose destination that member does not belong to.**

A compartment's name is part of its contents. "Supplier Gamma", listed for
someone who is only in the general compartment, does not leak Gamma's price. It
leaks that a Gamma exists, who they are, and that they are bidding on the same
project: precisely what a per-relationship compartment exists to hide. The
identifier does not save it, because implementations derive it from the name so
that it stays readable in `ework:compartment` and in graph queries.

**A role does not open a compartment.** `owner` and `admin` run the project and
still cannot see compartments they do not belong to. This is not gratuitous
strictness: visibility is a key and never a permission (ADR-0011), so in phase 2
someone outside the MLS group cannot decrypt, and no role invents a key. A host
that hands over the list because of a role is handing over what the cryptography
will refuse, and the gap shows up as lost functionality on migration day.
Administration remains possible because **whoever creates a compartment belongs
to it**, which is why that rule exists.

**A response about a compartment the requester does not belong to MUST be
indistinguishable from a response about a compartment that does not exist**, in
code and in body, as in EW-RFC 0004 §5. Trimming the list and then answering
"does not exist" and "you do not belong" in different words gives back on
request what the trim took away: guess names until one of the two sentences
changes. Both refusals say the same thing, and the single sentence is the truth
of both: the requester does not have that compartment.

# Tasks Restricted to a Compartment

~~~
"ework:compartment": "financial"
~~~

The task is encrypted to that group and to no other. A party outside it **never
receives the packet**: for that person the task does not exist, appears in no
list, and there is nothing to decrypt. This is the mechanism for the case "the
installer does not need to see anything about payments".

Without `ework:compartment`, the task goes to `defaultCompartment`.

# Sensitivity Policy

`sensitivityPolicy` maps a label to a destination compartment. A task or field
labelled `money` goes to `financial` automatically.

The author **labels**, the project's policy **decides the compartment**. Nobody
chooses a key by hand.

~~~
"ework:classification": {
  "/ework:payloads/0": ["money"],
  "/ework:attachments/1": ["health"],
  "/description": ["personal-data"]
}
~~~

The core label vocabulary is `money`, `health`, `personal-data`, `document`,
`location`, `credential`. It is extensible by third-party namespace, like the
payloads (EW-RFC 0008 §1).

**Defaults that protect on their own.** Every typed payload declares the default
classification of its fields, and clients MUST apply it without asking:

| Payload | Fields classified by default |
|---|---|
| `payment@1` | everything: amount, payee, codes and charges are `money` |
| `appointment@1` | `professional` and `preparation` are `health` when the payload comes from a health issuer |
| `delivery@1` | `address` and `contact` are `personal-data` |
| `approval@1` | nothing by default |

The author MAY loosen (move to a wider compartment) with an explicit act, and
the client MUST record that in the project log.

**Closing is automatic, opening is a decision.** A label mapped to `none` MUST
NOT be transmitted at all: the policy is the place where the project owner says
"credentials never travel here", and an implementation that treats an unmapped
label as "send to the default" would silently invert that.

An unknown label MUST route to the default compartment, never to the widest.

# Sealed Sections

A task can live in a wide compartment while a part of it belongs to a narrow
one.

~~~
{
  "@type": "Task",
  "title": "Deliver upper cabinets",
  "ework:compartment": "general",
  "ework:sealed": [
    { "id": "s1",
      "compartment": "financial",
      "covers": ["/description"],
      "content": "<encrypted to the financial compartment>" }
  ]
}
~~~

The covered paths are removed from the clear object and carried inside the
section. A reader inside the compartment reassembles them; a reader outside does
not.

**The existence of the section MUST remain visible to readers outside it**,
including which paths it covers. This is deliberate. The alternative, hiding the
fact that something is hidden, produces a worse outcome: a participant makes
decisions about a task without knowing that information bearing on it exists.
Seeing "there is a sealed section here, for the financial compartment, covering
the description" is the correct amount of disclosure.

## Binding a sealed section to its task

A sealed section MUST carry its own `nonce`, and the AEAD MUST use as associated
data the tuple (task uid, section id, destination compartment, canonical hash of
the cleartext object once the covered paths are removed). Clients MUST reject a
section whose decryption fails that check, and MUST present the task as tampered
rather than render what opened.

Without associated data the seal is a loose block: nothing ties it to the task
carrying it or to the version of the object. A member of the wide compartment, who
sees the task and cannot open the seal, cuts the block from another task of the
same narrow compartment and pastes it here in a revision. The narrow
compartment's client decrypts successfully, reassembles, and shows the right
content in the wrong context, with no sign that a swap occurred.

## What MUST NOT be sealed

Covered paths MUST NOT reach `ework:execution` or any field governing the
authority of an action or the human-approval policy, and two sections MUST NOT
cover overlapping paths. Sealing the execution policy would produce one signed
task with two meanings, where the narrow compartment reads
`humanApproval: "none"` and the wide one reads `humanApproval: "before"`, each
side holding cryptographic proof that it is right. A decision rule is a contract
among all participants and therefore lives in the clear; the secret is the datum,
never the rule governing it.

# Cross-Compartment Dependencies

The problem: assembly depends on the instalment payment, and the installer
cannot see the payment. A `dependson` relation pointing directly at the
invisible task would leak its existence, its identifier, and its lifecycle,
turning the project graph into an inference channel.

**Normative pattern: the public milestone as proxy.**

~~~
  [Instalment payment]  --updates-->  (Milestone: payment released)
   compartment: financial              compartment: general
                                              |
                                              | dependson
                                              v
                                        [Assembly]
                                        compartment: general
~~~

- The milestone is an ordinary task in the wide compartment, with a
  deliberately neutral title and no sensitive payload.
- The private task declares `ework:updatesMilestone` with the milestone's
  identifier. On completion, the client of a party inside the narrow
  compartment completes the milestone too.
- Tasks in the wide compartment depend on the **milestone**, never on the
  private task.
- Clients MUST NOT create a `relatedTo` relation crossing into a compartment the
  reader does not belong to. On detecting such an attempt they MUST offer to
  create the milestone instead.

The refusal message MUST NOT confirm that the target task exists. Offering the
milestone is the correct response precisely because it resolves the user's need
without answering the question "is there something there?".

The installer sees "waiting on: payment released" and the date it unblocked. He
does not see the amount, the payee, or how many instalments there are. That is
the exact quantity of information his work requires.

# Projects Across Hosts

A project that only accepts members from the same host is not federated. The
reference scenario is the opposite: the owner, the carpenter, and the installer
are rarely with the same provider, and requiring that they be is asking everyone
to migrate before starting.

- The invitation travels as `project.invite`, carrying the project, the
  compartments that party belongs to, and who owns it.
- The receiving host stores a **replica**. The owner remains whoever created it,
  and the local copy exists so the person can see the project and receive what
  is theirs.
- **The invitation MUST NOT enter on its own.** Arriving at a host where nobody
  knows the sender is the same problem as an unknown issuer, and the answer is
  the same: without a prior relationship, the invitation waits in quarantine.
  Otherwise "add to project" becomes a way to place content in the box of
  someone who never authorised the sender.
- Tasks of a compartment MUST be delivered to the remote members **of that
  compartment**, not to all project members. A project member is not a member of
  every compartment, and treating the two as the same undoes compartmentalisation
  on the first federated delivery.
- On the receiving side, what authorises the delivery is **membership in the
  compartment**, not a relationship credential: the task arrives at the primary
  handle, which has no credential attached. This opens no hole, because the
  membership only exists if the person accepted the invitation, and the
  invitation went through the consent door.

# Governance and Retroactivity

Adding a member to a compartment grants access to what that compartment holds
from that point forward. Whether prior history is re-shared is a decision of the
compartment's owner and MUST be explicit, because the two behaviours have
different consequences and neither is universally correct. Re-sharing follows
EW-RFC 0006 §3: it MUST be trimmed to the destination compartment, and SHOULD be
done per task rather than in bulk.

**Removing a member: the list and the key move together.** Removal MUST be carried
out as an MLS Remove in that compartment's group, and the change to the member list
MUST NOT be presented as complete before the new epoch has been committed. Until
then, clients MUST show the member as still present. The Remove is issued by the
identity holding the `owner` or `admin` role that executed the change, and
implementations MUST NOT accept a removal that alters only the list.

Content the member already decrypted cannot be recalled, and implementations MUST
NOT suggest otherwise.

**Two sources of truth, one reconciliation.** The roster exists in the project
object and in the MLS tree, and it is the tree that decides who can open content.
The list describes; the tree authorises. Where they diverge, clients MUST treat the
tree as authoritative for visibility, MUST surface the divergence to owners and
admins, and MUST NOT present the list as the set of readers.

The rule exists because the natural action in an interface, removing the name from
the list, is exactly the one that takes no one's key away. Without it an admin
removed the supplier, the screen showed them gone, and they stayed in the current
epoch reading everything: the list would have become permission, and the key would
have stayed where it was.

**Nothing is retroactive.** Reclassifying a field protects the future. What has
already been delivered and decrypted by someone does not come back, and the
interface MUST say so, in those words, before the person believes they have
fixed a leak.

# What Remains Visible

An honest list, for this version:

1. **To members of the wide compartment:** that a sealed section exists in a
   task, its destination compartment, and its approximate size.
2. **To members of any compartment:** the list of the project's compartments and
   their names, because that comes from the project configuration. Compartment
   names SHOULD NOT describe sensitive content ("Jane's employment lawsuit" is a
   leak in the name itself).
3. **To hosts:** the opaque group ids, the epochs, who receives an envelope of
   which group, sizes and cadence. An attentive host infers that certain members
   take part in fewer things.
4. **To everyone:** the existence of the public milestone and the moment it was
   completed, which is exactly what it exists to reveal.
5. **To all project members:** that a member downgraded to assisted mode a box
   holding content of that project (EW-RFC 0006 §7). This is visible on purpose:
   the decision is one person's, the consequence is everyone's, and hiding it
   would be the silent downgrade the threat model forbids.
6. **To members of the wide compartment:** that the member list and the tree of
   the narrow compartment have diverged, when they diverge (see Governance and
   Retroactivity). The divergence signals an error or an incomplete removal, and
   is worth more seen than hidden.

What is NOT visible: the content of a task in a compartment the reader does not
belong to, the content of a sealed section, amounts, attachments, and the
corresponding typed payloads.

# Security Considerations

In the target state, each compartment is an MLS {{RFC9420}} group. Membership
changes are group operations, with the forward secrecy and post-compromise
security properties that protocol provides.

In assisted mode, compartments are server-enforced access control. This protects
against other participants and not against the host operator. The honesty
requirement above is what keeps the difference visible to the user.

Sealed sections in assisted mode have the same limitation, and additionally the
host learns the shape of what is sealed: which compartment, which paths. That
metadata is unavoidable in this mode, since the host performs the assembly.

The model depends entirely on the author sealing at the moment of writing. A
client that generates the task with the amount in the clear in the wide
compartment has created a leak that no later configuration repairs, which is why
the default classifications are normative and not suggestions. The conformance
suite MUST test that a payment payload created inside a multi-member project is
born sealed.

A malicious member inside the compartment can still pass on what they see. That
has no cryptographic solution, it is the limit of every sharing system, and it
is the reason the right granularity is "who needs it", not "who is trusted".

Identifier collision remains an oracle, and the trimming rule above does not
close it. Creating a compartment whose identifier already exists has to fail,
or the new one would erase the old, and failing reveals that the old one
exists. Someone who administers a project without belonging to a compartment
learns one name per attempt. There is no way out within the present design: the
identifier is the object's key and is derived from the name, so creation either
overwrites or tells. Escaping it requires opaque identifiers, at which point
`ework:compartment` stops being readable and the property that motivated
name-derived identifiers is gone. This is recorded as a residual risk until
there is a proposal that does not trade one leak for another.

# Privacy Considerations

Compartmentalisation is minimisation at the source, and it is the cheapest
privacy mechanism in the protocol: content that never reaches a party cannot
leak from that party. The sensitivity policy exists so that the decision is made
once, at the project level, rather than repeated by each participant on each
task, where it will eventually be forgotten.

The list in "What Remains Visible" is the contract: if it is not there, it does
not leak. Host metadata remains subject to EW-RFC 0006 §6, which now includes
the compartment and the presence of a sealed section.

# Open Questions

1. **UX cost:** how many compartments does an ordinary person tolerate before
   giving up and sending everything to the general one? Decided on 2026-08-04:
   the interface speaks of **circles** with a name ("just me and the carpentry
   shop"), never of compartments, and automatically creates the per-relationship
   circle when a supplier joins. "Compartment" stays as the specification term,
   "circle" as the product term. How many circles a person tolerates remains a
   question for user testing in phase 3.
2. **Default for personal projects** with a single member: a compartment is pure
   overhead there, and the client SHOULD hide compartments entirely.
3. Automatic public milestone: should the client create the milestone on its own
   when it detects a cross-compartment dependency, or always ask?
4. The health default classification (see Sensitivity Policy) depends on knowing
   that the issuer is a health issuer, which today is not declared anywhere.
   Does it take an issuer category field, or is it left to the manual label?
5. Partial sealing of an attachment (hiding pages of a PDF): out of scope, or
   worth a derivation mechanism?
