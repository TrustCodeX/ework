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

# Project Object

~~~
{
  "@type": "Project",
  "uid": "019fd0...",
  "title": "Kitchen",
  "owner": "did:key:z6MkOwner...",
  "compartments": {
    "general":   { "name": "Everyone on the project", "members": ["*"] },
    "financial": { "name": "Financial", "members": ["did:key:z6Mk..."] }
  },
  "defaultCompartment": "general",
  "sensitivityPolicy": { "money": "financial", "health": "financial",
                          "credential": "none" },
  "created": "2026-08-05T00:00:00Z"
}
~~~

Every project has a `general` compartment containing all members. Tasks default
there.

# Sensitivity Policy

`sensitivityPolicy` maps a label to a destination compartment. A task or field
labelled `money` goes to `financial` automatically.

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

# Governance and Retroactivity

Adding a member to a compartment grants access to what that compartment holds
from that point forward. Whether prior history is re-shared is a decision of the
compartment's owner and MUST be explicit, because the two behaviours have
different consequences and neither is universally correct.

Removing a member removes access to future content. Content the member already
decrypted cannot be recalled, and implementations MUST NOT suggest otherwise.

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

# Privacy Considerations

Compartmentalisation is minimisation at the source, and it is the cheapest
privacy mechanism in the protocol: content that never reaches a party cannot
leak from that party. The sensitivity policy exists so that the decision is made
once, at the project level, rather than repeated by each participant on each
task, where it will eventually be forgotten.
