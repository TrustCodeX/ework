---
docname: draft-wilhelm-ework-actors-00
title: "The e-work Protocol (EWP): Autonomous Actors and Execution Confirmation"
abbrev: EWP Actors
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC8032
---

This document specifies how the e-work Protocol (EWP) treats non-human parties.
A client may be a person, a script, a scheduled job, or an autonomous agent, and
the protocol makes that distinction explicit rather than assuming a human is
present.

The mechanism is a class attached to each device key. A task that requires human
approval is satisfied only by a signature from a key of class `human`, so the
requirement is verifiable by any party rather than being an application-level
convention.

<!-- abstract -->

# Introduction

Task systems generally assume a person at the other end. That assumption is
already false, and becoming more so: bills are paid by scheduled jobs,
deployments are confirmed by pipelines, and increasing numbers of routine
decisions are taken by software acting on a person's behalf.

Two problems follow. First, when something goes wrong, attribution is
impossible: a log entry saying the account acted does not say whether a person
was involved. Second, some tasks genuinely require a human, and a system with no
way to express that has no way to enforce it.

## Conventions and Definitions

<!-- rfc2119 -->

# Actor Classes

Every device key in an identity document carries a class.

| Class | Meaning |
|---|---|
| `human` | A person operates this device directly |
| `assisted` | Software acts, with a person supervising in the loop |
| `autonomous` | Software acts without supervision |
| `system` | Infrastructure: the host itself, a gateway, a bridge |

The class is a property of the **key**, not of the request. It is recorded in
the identity document, signed by the root key, and a party changing it must
re-sign that document.

This is what makes the mechanism work. A class asserted per request could be
claimed by anything; a class bound to a key means that acting as `human`
requires possessing a key registered as `human`, and possession of that key is
the thing being protected.

Entries carry `actorClass`, and receivers MUST verify it against the identity
document rather than trusting the entry's own claim. An entry whose declared
class differs from the registered class of the signing device MUST be rejected.

# Execution Policy

A task may state what its resolution requires.

~~~
"ework:execution": {
  "humanApproval": "before",
  "confirmedBy": [],
  "messageRequired": ["decline", "fail"],
  "autonomousAllowed": false
}
~~~

| Field | Meaning |
|---|---|
| `humanApproval` | `none`, `before`, or `after` |
| `confirmedBy` | Identities whose confirmation is required |
| `messageRequired` | Actions that MUST carry a non-empty message |
| `autonomousAllowed` | Whether a key of class `autonomous` may act at all |

## Enforcement

The receiving host MUST refuse an entry that violates the policy. Refusing at
the host, rather than merely disabling a button in one client, is what makes the
rule apply to every client, including the autonomous agent that never renders an
interface.

With `humanApproval: "before"`, an action moving the task toward completion MUST
be signed by a key of class `human`. An `autonomous` key attempting it MUST be
refused, with a message naming the class of the offending device: diagnosing
this from a generic authorisation error is unnecessarily hard.

With `autonomousAllowed: false`, a key of class `autonomous` MUST NOT act on the
task at all.

`messageRequired` exists because "why?" is the question an audit asks first. A
refusal or failure without a stated reason is a record that satisfies the form
and not the purpose.

# Execution Confirmation

`humanApproval: "after"` introduces a state between doing and done. The agent
performs the work and moves the task to `awaiting-confirmation`; a human then
signs a `confirm` action.

The distinction from acknowledgement, specified in the urgency document, matters
and is easy to blur: **acknowledgement is "I saw it", confirmation is "it was
done, and someone attests to that"**. A critical task MAY require both, and in
that case remaining in `awaiting-confirmation` is a valid escalation trigger:
the medication was marked as taken, and the carer has not yet confirmed.

# Adding an Agent

An agent is a device of the identity, added like any other:

1. The identity generates a new key pair for the agent.
2. It adds the device to the identity document with the appropriate class and
   increments `seq`.
3. It signs the document with the root key and publishes it.

**The agent never receives the root key**, and receives only its own. An agent
key is compromised more often than a personal device, because it lives on
servers, in CI configuration, and in container images; the blast radius must be
one key.

Revoking an agent is removing its device from the document. Entries it signed
remain valid and remain attributable, which is the point: the history says what
the agent did, and that record is not erased by the agent ceasing to exist.

# Security Considerations

The mechanism protects against an agent exceeding its remit, and against a
system misrepresenting a machine decision as a human one. It does not protect
against a person delegating their human-class key to software, which no protocol
can prevent and which the audit trail at least renders visible after the fact.

Class is not privilege. An `autonomous` key is not a lesser key: it can do
everything a `human` key can, except satisfy a policy that demands a human. The
separation is about attributability, not about restriction, and implementations
that treat it as a permission tier will produce surprising behaviour.

# Privacy Considerations

Actor class is visible to counterparties, and that is deliberate: a party
negotiating with an autonomous agent has a legitimate interest in knowing it.
The class does not reveal which software, which vendor, or which model, and
implementations MUST NOT add such detail to the class field. Regimes granting a
right to review automated decisions are served by this: the decision is
attributable and the demand for human review is expressible, which the
regulatory document maps out.
