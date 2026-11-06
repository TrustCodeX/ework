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
  "requiresHumanAbove": [ { "amount": "200.00", "currency": "BRL" } ],
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
| `messageRequired` | Action types that MUST carry a non-empty message: a list, `[]` for none, or `["*"]` for every action |
| `requiresHumanAbove` | Thresholds, at most one per currency, above which execution MUST be signed by a `human` key |
| `autonomousAllowed` | Whether a key of class `autonomous` may act at all |

An absent or empty `confirmedBy` means **nothing declared**, and an undeclared
field does not compose: the other side's stands, because intersecting with an
absence would zero out the requirement of whoever did declare, with nobody having
said anything. `["executor"]` is the explicit statement that whoever executes
attests to their own work, is the field's default, and in that case there is no
third-party confirmation and no waiting state. **The default changes when the
`executor` is not the `owner`** (an executor proposed by the issuer, see the
data model document, or a `reassign`): with no `confirmedBy` declared,
`["owner"]` applies, the executor's `complete` takes the task to
`awaiting-confirmation`, and the owner confirms or contests. Letting whoever
does the work attest to it is only acceptable when whoever does it is the
owner, because then there is no third party to protect (ADR-0029). Third-party confirmation is what a
populated `confirmedBy` exists for: the carrier says it delivered, and the client
confirms it received. `evidence` demands an
evidentiary attachment of kind `proof`, specified in the data model document,
or the structured result of the machine-to-machine profile, together with the
completion.

`requiresHumanAbove` is a list here, and not the single object of the delegation
scope above, because amounts in different currencies do not compare: the
composition below MUST keep a threshold per currency, and keeping only one would
silently discard the requirement of whoever declared the other. The threshold that
applies is the one in the task's currency, and a currency with no declared
threshold has no ceiling.

`maySign`, specified in the history document, appears in this same object and is
**not** the issuer's to declare: widening authority belongs to the `owner`.
Receivers MUST NOT honour `maySign` arriving inside an offer and MUST drop it when
applying one. Without that rule the issuer wrote `maySign: { "accept": ["issuer"] }`
into its own offer and signed the acceptance of it, which is precisely the record
the authority rules exist to prevent, produced by the field meant to correct the
default.

## Enforcement

Enforcement is not trust in someone else's client, and it is not the host
either. The argument for the host is a good one: putting the rule in the server
would make it apply to every client, including the autonomous agent that never
renders an interface. It fails for the reason this document opens with, which is
that the server is blind. A rule that depended on the server would hold only
where the content is in the clear, and would vanish in the encrypted deployment,
which is the default.

What closes the gap without a server is the receiver. Whoever applies an entry
checks before applying, and what fails becomes a **void entry**, specified in the
history document: recorded, attributable, and without effect. This is the same
choice the authority rules make, for the same reason. Not depending on someone
else's client never meant depending on the host; it means every peer checking for
itself.

Each field closes by its own path, and none of them passes through the
recipient's server. With `humanApproval: "before"`, an action moving the task
toward completion MUST be signed by a key of class `human`. An `autonomous` key
attempting it MUST be refused, with a message naming the class of the offending
device: diagnosing this from a generic authorisation error is unnecessarily
hard.

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

It is checked by peers, in the two moments the history document sets out:
whoever composes the entry MUST NOT sign it without the required message, and
whoever receives one lacking that message MUST treat it as a void entry rather
than discard it. Discarding would break the `after` chain for every later entry
referencing it, manufacturing exactly the gap that verifiers are told to read as
host withholding. The recipient's host is not the checkpoint: under end-to-end
encryption it reads neither the message nor this policy, and granting it that
sight would add both to the exhaustive list of observable metadata, trading the
confidentiality of the whole history for a check the peer already performs.

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
autonomous permission, the union of the two `messageRequired` lists, the lower
threshold of each currency across both lists, and the **intersection** for who
may confirm, falling back to the recipient's list where that intersection is
empty.

That last case was missing, and without it the rule did not close: two disjoint
confirmer lists do not compare by strictness, and an issuer naming itself as
confirmer was left with no counterparty. Whoever receives decides who confirms in
their own name.

Composition **only tightens, and is idempotent**: composing again with a policy
already folded in changes nothing, and the result is no looser than either input
in any field, with the declared exception of that last row. That property is what
makes the composed policy checkable by third parties, and the section below
depends on it.

## The composed policy is published

The composition has an input only one side can see, and that decides where it can
apply. The issuer's part arrived inside the offer and is in plain view. The
recipient's part sits in their box.

If the recipient composed silently and judged by that private policy, two peers
applying the same history would reach different states. The worst case is not the
wasted record: it is the recipient's own agent signing `complete` on a task whose
private policy forbids agents. The issuer applies it, because the policy the
issuer can see allows it; the recipient does not. For one the bill is paid and for
the other it is not, and the requirement that the history be the same for
everyone, specified in the history document, is broken by an input that is not in
the history. A requirement nobody can see does not become protection; it becomes
state divergence.

Therefore:

1. `ework:executionPolicy` is an **input to the composition, never a judge**. No
   entry MUST be judged by a policy that is not published.
2. Whoever hardens **publishes the result**: the composed policy enters the task
   through an entry carrying the `harden` action, specified in the history
   document, signed by the `owner`, in the group of the participants it binds.
   Clients MUST NOT publish it when the composition tightens nothing, and MUST
   publish it no later than acceptance, which is when it starts to bind the other
   end.
3. **An entry is judged by the policy published in its causal past.** Hardening
   published afterwards MUST NOT reach an earlier entry. This is the line that
   closes the case of being bound by an unknowable requirement: what is not in
   your causal past does not bind you, and the author's `after` proves what they
   knew.
4. The private policy binds where it can bind without hiding anything from
   anyone: **at signing time**. The recipient's client, and the recipient's own
   host, MUST NOT sign or publish an entry that violates it. It is the first of
   the two moments the history document sets out, applied to the whole policy.
5. `harden` **only tightens**: a `harden` entry whose policy is not at least as
   demanding as the published one MUST be treated as a void entry. Any
   participant checks that alone, because both policies are in the history.
6. `harden` is **not status**, it is a term of the deal: the status sharing
   policy of the consent document MUST NOT filter it, and it MUST reach every
   participant it binds. Filtering it would hand the whole problem back, with the
   issuer bound by a requirement the recipient's host chose not to deliver.

**What this costs, said up front:** the issuer now sees the projection of the
recipient's policy onto that task. The projection, not the rule: the issuer learns
nothing about what it says to other issuers, to other payloads or to other tasks,
and learns nothing at all when the composition tightens nothing. That is the price
of a requirement that binds someone who did not write it, and the alternative is
measured above.

**A revision MUST NOT loosen an accepted policy.** A revision within a series that
makes the execution policy less demanding in any of those fields MUST require fresh
explicit acceptance, and clients MUST NOT apply it silently. Otherwise the issuer
offers with human approval required, which the person reads as a safe task, and
after acceptance sends a revision in the same series turning it into one the agent
performs alone.

"Less demanding" is the inverse of the composition, field by field. A revision
loosens when human approval descends the order, when autonomous permission goes
from false to true, when a required-message list drops an action the published one
required, when a threshold rises for some currency or disappears, or when
confirmation by a third party is waived or given to a confirmer the published
policy did not name.

A revision carries the issuer's policy, and the hardening the recipient published
still applies: the task's policy after a revision is the composition of the two,
not the envelope's alone. This is not silent composition, and that is the
difference that matters: both parts are in the history, so every peer reaches the
same result. What a revision may move is the issuer's part, and that is what the
test above decides on.

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

Publishing the composed policy closes an attack that private composition opened on
its own: with the recipient's policy only in their box, the peer receiving an entry
and the peer who wrote it derive different states from the same history, and the
issuer takes as paid a bill the recipient never applied. The conformance suite MUST
test that two peers receiving the same entries reach the same state, and MUST test
that a `harden` which loosens enters the history without effect.

`maySign` arriving inside an offer is the other path by which the other side's
policy became one's own authority: the field exists for the `owner` to widen, and
honouring it from the issuer let the issuer sign the acceptance of its own offer,
blessed by the object the issuer itself wrote.

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

Publishing the composed policy reveals to the issuer the **projection** of the
recipient's policy onto that task, and only to the issuer and the other members of
that group. They learn that a human is required there, or a justification, or that
someone else confirms; they do not learn the rule behind it, nor that it holds for
anyone else, and they learn nothing when the composition tightens nothing. For the
hosts nothing changes: `harden` is an entry like any other, encrypted to the group,
and the exhaustive metadata list of the cryptography document is unchanged. Clients
SHOULD say, at the moment of hardening, that the other end will see it, because an
announced requirement is what this design trades for a requirement that would not
hold.

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
6. `evidence`, `confirmationTimeout` and `onTimeout` stayed out of the
   composition table, because "stricter" is not obvious for the three of them: a
   short timeout with `onTimeout: "fail"` is harder on whoever executes and
   softer on whoever waits, and the ordering depends on which of the two the
   rule protects. `evidence` (`none` < `optional` < `required`) looks like the
   easy case and joins the table without argument; the other two need that
   question answered first.
7. Is the private policy one per box, or one per relationship? One per
   relationship is what practice asks for, because what makes sense for the bank
   does not make sense for the school group, and it is also one more dimension
   for the person to configure without knowing it exists. A safe default for
   whoever configures nothing remains the requirement.
