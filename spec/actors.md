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

Clients MUST display the actor's class on every recorded action, so that "it
was me" and "it was my agent" are never indistinguishable after the fact.

# Delegation

An agent acts under a delegation, a scoped credential signed by the root key
and referenced from the identity document:

~~~
{
  "@type": "Delegation",
  "uid": "del_018f...",
  "principal": "did:key:z6MkCleia...",
  "agentKey": "ed25519:...",
  "actorClass": "autonomous",
  "scope": {
    "circles": ["financial"],
    "actions": ["accept", "schedule", "complete", "snooze"],
    "payloadTypes": ["ework.dev/payload/payment@1"],
    "maxValue": { "amount": "500.00", "currency": "BRL" },
    "maxPerDay": 20,
    "requiresHumanAbove": { "amount": "200.00", "currency": "BRL" }
  },
  "validUntil": "2026-11-04T00:00:00Z",
  "proof": { "by": "did:key:z6MkCleia...", "sig": "..." }
}
~~~

`circles` limits the agent to the circles it needs, specified in the
compartments document: delegating is not handing everything over. A payments
agent sees the financial circle and does not see the rest of the project.

Revocation is immediate and uses the same mechanics as device revocation,
specified in the identity document: removal from the document and exit from the
groups, with a new epoch.

Actions outside the scope are refused by peers when they validate the signature
against the delegation.

# Execution Policy

A task may state what its resolution requires.

~~~
"ework:execution": {
  "humanApproval": "before",
  "confirmedBy": ["issuer"],
  "confirmationTimeout": "P3D",
  "onTimeout": "escalate",
  "evidence": "required",
  "messageRequired": ["decline", "fail"],
  "autonomousAllowed": false
}
~~~

| Field | Meaning |
|---|---|
| `humanApproval` | `none`, `before`, `after`, or `both` |
| `confirmedBy` | Who may confirm completion: `executor` (the default), `issuer`, `owner`, `any-human`, or specific identities |
| `confirmationTimeout` | How long completion may wait for confirmation |
| `onTimeout` | `escalate`, `fail`, or `auto-confirm` |
| `evidence` | `none`, `optional`, or `required` |
| `messageRequired` | Actions that MUST carry a non-empty message |
| `autonomousAllowed` | Whether a key of class `autonomous` may act at all |

Third-party confirmation is what `confirmedBy` exists for: the carrier says it
delivered, and the client confirms it received. `evidence` demands an
evidentiary attachment of kind `proof`, specified in the data model document,
or the structured result of the machine-to-machine profile, together with the
completion.

## Enforcement

The receiving host MUST refuse an entry that violates the policy. Refusing at
the host, rather than merely disabling a button in one client, is what makes the
rule apply to every client, including the autonomous agent that never renders an
interface.

With `humanApproval: "before"`, an action moving the task toward completion MUST
be signed by a key of class `human`. An `autonomous` key attempting it MUST be
refused, with a message naming the class of the offending device: diagnosing
this from a generic authorisation error is unnecessarily hard.

This holds only if someone checks the declaration at the right source. Receivers
MUST resolve the signing key in the author's current identity document and use
**only** the `actorClass` signed there by the root key. The class carried inside an
entry is display redundancy, MUST NOT satisfy any policy, and a mismatch between
the two MUST make the entry void.

Without that obligation the human/machine distinction was declarative: the agent
composed an entry claiming `actorClass: "human"`, signed it with its own key,
produced a valid signature for the identity, and the only written rule about the
field said to display it.

**The delegation MUST be retrievable by whoever must validate it.** The identity
document carries only an opaque identifier, and an identifier nobody can resolve
lets nobody validate anything: the party expected to refuse the out-of-scope action
was precisely the one profiting from it. The delegation object MUST be served by
the identity's host at `GET <clientApi>/delegation/<id>`, signed by the root, and
peers MUST fetch and check its scope before accepting an action from an
`autonomous` key. An action whose delegation cannot be retrieved, or whose
delegation does not cover it, MUST be treated as a void entry.

With `autonomousAllowed: false`, a key of class `autonomous` MUST NOT act on the
task at all.

`messageRequired` exists because "why?" is the question an audit asks first. A
refusal or failure without a stated reason is a record that satisfies the form
and not the purpose. The message travels atomically with the transition it
justifies, as the history document specifies: it is what makes "the agent
completed and explained why" a guarantee instead of a courtesy.

**An honest limit, and it MUST appear in any material about this mechanism:**
the key proves **who** signed, never that the person paid attention. A human who
approves everything on autopilot is still a human in the loop in the protocol's
eyes. Against inattention there is no cryptography, only interface design:
clients SHOULD display what is being approved with prominence proportional to
the risk (value, urgency, irreversibility), and MUST NOT offer "approve all".

## Composing two policies

Where issuer and recipient declare different policies, the stricter prevails.
Nobody loosens the other's requirement, and one practical consequence is
normative: a user's auto-accept rule, specified in the consent document, MUST
NOT apply to a task whose issuer requires `humanApproval: "before"`.

"Stricter" has to be computable, or it is not a rule. The recipient's policy lives
in their own box and never in the arriving object, and composition MUST be
performed field by field: the most demanding value for human approval, in the
order `none` < `after` < `before` < `both`, false winning over true for
autonomous permission, the union of required-message lists, the lower of two
thresholds, and the **intersection** for who may confirm, falling back to the
recipient's list where that intersection is empty.

That last case was missing, and without it the rule did not close: two disjoint
confirmer lists do not compare by strictness, and an issuer naming itself as
confirmer was left with no counterparty. Whoever receives decides who confirms in
their own name.

**A revision MUST NOT loosen an accepted policy.** A revision within a series that
makes the execution policy less demanding in any of those fields MUST require fresh
explicit acceptance, and clients MUST NOT apply it silently. Otherwise the issuer
offers with human approval required, which the person reads as a safe task, and
after acceptance sends a revision in the same series turning it into one the agent
performs alone.

The consent scope composes from outside the task as well: it MAY require that
offers from that issuer always demand human approval, regardless of what the
issuer declares.

# Execution Confirmation

`humanApproval: "after"` introduces a state between doing and done. The agent
performs the work and moves the task to `awaiting-confirmation`; a human then
signs a `confirm` action.

The distinction from acknowledgement, specified in the urgency document, matters
and is easy to blur: **acknowledgement is "I saw it", confirmation is "it was
done, and someone attests to that"**. A critical task MAY require both, and in
that case remaining in `awaiting-confirmation` is a valid escalation trigger:
the medication was marked as taken, and the carer has not yet confirmed.

`confirmationTimeout` bounds the wait. When it expires, `onTimeout` decides:
`escalate` triggers the urgency mechanics, `fail` fails the task, and
`auto-confirm` is permitted only when `evidence` is `none` and the policy
requires no human.

Whoever confirms may also refuse, returning the task to `in-process` with a
reason. Contestation is what gives third-party confirmation teeth: the word of
whoever executed stops being final.

# Machine-to-Machine Profile

When there is no human at either end, the task must be executable without
interpretation:

~~~
"ework:machine": {
  "acceptance": { "type": "ework.dev/acceptance/json-schema@1", "schema": { } },
  "result": { "type": "ework.dev/result/structured@1" },
  "retry": { "max": 3, "backoff": "PT5M" },
  "idempotencyKey": "catalog-sync-2026-08-04"
}
~~~

`acceptance` defines the success criterion in a form the other end can verify,
instead of prose. `result` carries structured, signed output, which becomes the
evidence the execution policy demands. Idempotency reuses the deduplication key
of the consent document: re-execution does not duplicate effect.

Machine-to-machine tasks are still tasks: they have deadlines, dependencies,
urgency and status, and they appear to their human owner whenever the owner
wants to look. The protocol does not become a message queue.

A peer that does not understand `ework:machine` treats the task as an ordinary
task, and someone performs it by hand. Graceful degradation applies here as in
every extension.

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

Delegation must never allow implicit sub-delegation: an agent MUST NOT issue a
delegation to another agent unless the original scope authorises it explicitly,
or the chain of responsibility dissolves.

The subtlest new risk is the empty human stamp: flows that ask for confirmation
so often that the person approves without reading. That degrades the protection
without any technical control noticing. The countermeasures live in interface
design, which is why they became rules in the enforcement section.

# Privacy Considerations

Actor class is visible to counterparties, and that is deliberate: a party
negotiating with an autonomous agent has a legitimate interest in knowing it.
The class does not reveal which software, which vendor, or which model, and
implementations MUST NOT add such detail to the class field. Regimes granting a
right to review automated decisions are served by this: the decision is
attributable and the demand for human review is expressible, which the
regulatory document maps out.

Actor class and delegations live in the identity document, which is public:
they reveal that the person uses automation, and how many agents they have.
Implementations that consider this sensitive MAY keep the agent as an identity
of its own, acting through a credential, at the cost of losing the direct chain
of responsibility. Execution signatures live inside the encrypted content, so
which key executed what does not leak to the hosts.

# Open Questions

1. Human presence attestation (a human key in a secure enclave, requiring
   physical interaction) for high-risk cases: worth an extension, or do the
   cost and the exclusion of ordinary devices kill the idea?
2. Controlled sub-delegation: what form lets a scope authorise a chain of
   agents without losing traceability?
3. Is `auto-confirm` on timeout too dangerous to exist, even restricted?
4. The `acceptance` vocabulary of the machine-to-machine profile: is JSON
   Schema enough, or does it need something genuinely executable?
5. How to present, to ordinary people, that a task was done by their agent and
   not by them.
