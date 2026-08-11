---
docname: draft-wilhelm-ework-consent-00
title: "The e-work Protocol (EWP): Consent Before Delivery"
abbrev: EWP Consent
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC8259,RFC8032
---

This document specifies the consent layer of the e-work Protocol (EWP). In EWP,
an issuer cannot deliver a task to a recipient without a prior, scoped,
revocable grant from that recipient. Delivery without such a grant is refused at
the receiving host's edge, before it reaches the recipient's box.

The document defines the consent object, the scope that bounds it, the
capability that binds a grant to a single address, the knock-and-quarantine flow
by which a stranger requests contact, and the two distinct ways a relationship
ends: revocation, which informs the counterparty, and silent retirement, which
does not.

<!-- abstract -->

# Introduction

Electronic mail delivers first and filters afterwards. Anyone who learns an
address can place a message in the recipient's inbox, and the recipient, or the
recipient's provider, decides afterwards whether it should have arrived. Two
decades of spam filtering, reputation systems, and authentication extensions
have improved the filtering without changing the ordering.

EWP inverts the ordering. Consent is not an application-level courtesy layered
above delivery; it is the precondition for delivery, enforced by the receiving
host. This document specifies that mechanism.

The inversion has a cost, and this document states it plainly: first contact
requires an explicit act by the recipient. A protocol that made first contact
free would be reintroducing the problem it exists to solve, so the friction is
the feature. What the design can do, and does, is make that act cheap, informed,
and reversible.

## Conventions and Definitions

<!-- rfc2119 -->

**Grant.** A signed statement by a recipient authorising a specific issuer to
deliver, within a stated scope, to a specific address.

**Scope.** The bounds of a grant: purposes, payload types, volume, maximum
urgency, and whether recurring series are permitted.

**Capability.** The credential an issuer presents at delivery time, bound to one
address and one grant.

**Knock.** A `consent.request` envelope, the only envelope type a host accepts
from a party with no existing relationship to the recipient.

**Quarantine.** The holding area where knocks wait. Contents are visible to the
recipient on demand and produce no notification.

# The Consent Object

~~~
{
  "@type": "Consent",
  "uid": "019f3c00-9d2a-7c31-b5f2-0d1a44e0c9a7",
  "user": "contact:z6MkK7f3a2q9...",
  "alias": "k7f3a2q9@ework.example",
  "issuer": "billing@utility.example",
  "issuerDept": "billing",
  "scope": {
    "purposes": ["billing"],
    "payloadTypes": ["ework.dev/payload/payment@1"],
    "maxPerMonth": 4,
    "maxUrgency": "normal",
    "recurringSeries": true,
    "regulatoryProfile": "br-lgpd@1"
  },
  "statusPolicy": "receipt",
  "createdAt": "2026-08-04T11:58:00Z",
  "validUntil": null,
  "state": "granted",
  "stateHistory": [ { "state": "requested", "at": "..." }, { "state": "granted", "at": "..." } ],
  "proof": { "type": "ed25519-jcs-2026", "by": "did:key:z6MkCleia...", "sig": "..." }
}
~~~

The object is signed by the recipient, not by the host. A host that could forge
a grant could authorise delivery to its own users, which would make the whole
mechanism decorative.

A consent moves from `requested` to `granted`, may alternate between `granted`
and `paused`, and ends in `revoked`; `expired` is reached when `validUntil`
passes. Transitions to `granted`, `paused`, and `revoked` are acts of the
recipient, signed by the recipient's identity. **Revoking MUST be as simple in
the UI as granting**, and the effect at the edge is immediate.

## Scope

| Field | Meaning | Enforced by |
|---|---|---|
| `purposes` | Declared purposes, free-form but registered where possible | Recipient client |
| `payloadTypes` | Typed payloads the issuer may send | Receiving host |
| `maxPerMonth` | Volume ceiling for the series | Receiving host |
| `maxUrgency` | Highest urgency the issuer may claim | Receiving host |
| `recurringSeries` | Whether repeated deliveries under one `dedupKey` are allowed | Receiving host |
| `regulatoryProfile` | Regime the issuer declares it operates under | Displayed, not enforced |

Scope fields that the receiving host enforces MUST be checked at the edge.
Scope fields that only the client can evaluate MUST be presented to the
recipient before the grant is made, not after.

`maxPerWeek` MAY replace `maxPerMonth` where the series runs on a weekly
rhythm; either way it is a rate the recipient's host applies at the edge.

A `maxUrgency` above `normal` requires explicit mention on the granting screen,
and `critical` requires a screen of its own, as the urgency document specifies.

`regulatoryProfile` is a declaration by the issuer about itself. A host MUST
NOT present an unregistered profile identifier as if it carried meaning. An
identifier invented by the issuer, displayed alongside registered ones, would
borrow a credibility it has not earned.

## Status policy

`statusPolicy` controls what flows back to the issuer when the recipient acts.

| Level | The issuer receives |
|---|---|
| `none` | Nothing beyond edge-level delivery success or failure |
| `receipt` | Delivery receipt, plus acceptance or refusal of the offer |
| `milestones` | The above, plus completion, failure, or cancellation |
| `full` | The above, plus progress and comments directed at the issuer |

The protocol default is `receipt`. Status beyond the configured level MUST NOT
leave the recipient's host: the filter runs in the client, because the content
is end-to-end encrypted, and is checked again at the edge.

A task MAY carry `statusSharing`, which narrows the level for that task alone.
**The effective level is the lower of the two, and implementations MUST compute
it that way.** Two failure modes follow from getting this wrong, and both were
observed in the reference implementation:

1. **Using only the task field hands the decision to the issuer.** The field
   arrives inside the offer, so the issuer writes it. Without comparing against
   the relationship's policy, an issuer that requests `full` receives
   everything, against what the recipient configured.

2. **A default value in the task field clamps the level downward always.** A
   task created carrying `receipt` makes the minimum win even where the
   recipient granted `milestones`, so an authorised issuer never learns that the
   task completed. The field MUST be omitted when there is no override; absence
   means "the relationship's policy applies", never "the default applies".

# Knock and Quarantine

A party with no relationship to a recipient has exactly one envelope type
available: `consent.request`.

~~~
{
  "ewp": "0.1",
  "type": "consent.request",
  "from": "billing@utility.example",
  "to": ["cleia@ework.example"],
  "body": {
    "name": "Utility Co.",
    "purpose": "Deliver your monthly bill as a task",
    "scope": { "purposes": ["billing"], "maxPerMonth": 4 },
    "sample": "Pay electricity bill, due on the 10th, 412.87"
  }
}
~~~

The body is short and plain: the issuer's verified identity, the requested
scope, a brief purpose text, and a static sample of what would be sent. A
`consent.request` MUST NOT contain a real payload, a payment link, or an
attachment.

The receiving host MUST place the request in quarantine and MUST NOT generate a
notification. Quarantine is visible on demand; it does not interrupt. A
mechanism whose purpose is to protect attention cannot itself demand attention
on arrival.

The host MUST replace any earlier quarantined request from the same issuer
rather than accumulate them, so that repeated knocking gains the issuer nothing.
A request left unanswered expires after 90 days.

## Knocking again from inside

**A knock from an issuer that already holds a live relationship is a REVISION
request for that relationship, not a new one.** Hosts MUST NOT create a second
consent for the same issuer: the per-relationship address exists so that a leak
says where it came from, and two relationships with one issuer make the border
enforce two agreements in parallel, adding their quotas together. The
one-pending rule above governs concurrency, not accumulation: without this
rule, knocking again each month would double the ceiling, counting on the
distracted yes of someone looking at two identical cards.

Accepting the revision replaces the scope of the existing consent, **keeps the
address**, and issues a new capability. The newly requested scope becomes the
ceiling from then on, upward or downward: the issuer asked for it, and that is
what a ceiling records.

**Declining a revision ends nothing.** The agreement already in force stays in
force, and the issuer may keep sending within it. That is what separates this
refusal from the first one: refusing someone who never came in keeps the door
shut, while refusing someone already inside merely says no to asking for more.
Clients MUST make that explicit, because the word "decline" on its own suggests
the wrong consequence.

## Rate limits on knocking

Hosts MUST rate limit `consent.request` by origin domain rather than by IP
address. In federation what matters is who is sending, and the address is
commonly a shared proxy.

Refusal MUST be explicit, carrying `over-rate` and `retryAfter`. Silent discard
would leave the sender unable to distinguish a limit from a loss.

## Knocking between people

The same knock serves when first contact goes from a person to a public handle:
the self-employed installer inviting a client. It travels as a
`consent.request` from a personal identity, MAY carry the context of a project
invitation, and lands in the same quarantine, under the same rules of silence,
limit, and expiry.

Acceptance establishes the contact relationship, with an MLS group of its own
and the `Relation` object described below, and issues no sending capability:
capabilities remain a thing for organisations. The real `project.invite` flows
after the acceptance.

# The In-Person Offer

The URI `ework:consent-offer/<payload>` carries, in unpadded base64url, a
`consent.offer` envelope. It is the counter path: presented as a QR code or a
link at the moment of signing up, on the printed bill, at the desk. The person
went looking for it, so the acceptance is born `granted` and never passes
through quarantine: quarantine protects from those who arrive uninvited, and
here the trust already exists in the physical world.

The offer is issued by the issuer's host at the request of an organisation
identity, and MUST NOT be signed by an account key: readers verify it against
the domain's published `hostKey`, and a signature by any other key produces an
offer the protocol itself says to refuse.

`to` is empty: the offer is bearer by nature, because whoever presents it does
not know the address of whoever will read it. That is what makes it useful at
the counter, and what makes it dangerous. The rules below exist because of that
trade.

Clients MUST, before showing any accept button:

1. **Verify the signature against the key the `from` domain publishes** as
   `hostKey` in discovery. It is the issuer's host that issues and signs the
   offer, with the domain's published key: that is what lets any reader verify
   by consulting only discovery, which is public by design and tells nobody who
   has an account where. An offer whose key is not the one published by the
   domain MUST be refused, not merely flagged.
2. **Show the verified domain**, not the `name`. The name is written by whoever
   produced the QR; the domain is what the signature proves. A screen that
   displays "Acme Energy" in large type with the domain in small print hands
   the attack over for free: all it takes is a QR pasted over the original at
   the counter.
3. **Refuse an expired offer.** `expiresAt` is mandatory and MUST NOT exceed 30
   days, because a printed QR outlives the contract that justified it.
4. **Show the whole scope** before acceptance, in the same words as the
   quarantine screen. The in-person path is faster, and cannot be less
   informed.

Hosts MUST apply the same checks when receiving the acceptance, and MUST NOT
trust the verification done by the client: a modified client is precisely the
case that verification exists to cover.

# Declining a Request: Discard and Block

The sections above say what happens when a request is accepted, and were silent
about the more frequent path, which is the other one. Declining has two forms,
opposite in reversibility and identical in silence.

**Discard** takes the request out of the box and SHOULD keep it recoverable for
a period, 30 days as the recommended term, after which it disappears. It is the
default, and it exists because declining by mistake is the common error: a list
of silent requests is exactly where a tap misses its target, and without a way
back the person is left depending on the issuer knocking again, with the issuer
having no signal that it should.

**Block** is explicit and permanent. Knocks from that issuer MUST be discarded
on arrival, without entering quarantine. It is the answer to someone who
insists, and it MUST NOT be the default: blocking someone who merely got the
address wrong is a cost the person should not pay for a mistaken tap.

**The response to the issuer MUST be identical in all four cases**: request
discarded, issuer blocked, nonexistent address, and account that never existed.
It is the same `unknown-recipient` as everywhere else, and the equality is not
code economy: a differentiated response teaches whoever knocks that the account
exists, that somebody read, and that the address belongs to a person. A block
that announces itself is an oracle, and the property that per-relationship
addressing exists to protect dies with it.

Implementations MUST NOT notify anyone on either action, nor on unblocking. The
protocol produces read receipts nowhere, and this is not the place to open an
exception.

The block list SHOULD be visible and reversible for whoever blocked. A list
that cannot be seen is a list that decides on its own, and the day to unblock
is precisely the day nobody remembers having blocked.

None of this travels: discard and block are local state of the recipient. A
host that implements them differently, or not at all, remains interoperable,
because whoever knocks cannot tell the difference. That is the requirement, and
the only one.

# Granting

On granting, the recipient's client:

1. Generates a new address for this relationship, opaque and unlinkable to the
   recipient's handle.
2. Constructs and signs the consent object.
3. Issues a capability bound to that address and that grant.
4. Sends `consent.grant` to the issuer.

The grant is sent **from the newly created address**, not from the recipient's
primary handle. This prevents an issuer from learning the primary handle
merely because contact was accepted.

## Revising the terms without ending the relationship

Granting again is also how a live agreement is **tightened**, not only how a
capability is rotated. Without it, changing one's mind about scope had only the
path that ends everything: someone who wanted to keep receiving the bill and
stop accepting rescheduling had to cut the whole relationship and wait for the
issuer to knock again.

**The permanent ceiling is the scope originally REQUESTED, not the one
granted.** Below it the recipient moves freely in both directions: tightening
after granting broadly is the common case, and loosening back up to what was
asked creates no authorisation the issuer had not already requested. Above it,
never: granting what nobody asked for invents permission the recipient has no
way to evaluate, because there was no request to read. Hosts MUST therefore
retain the requested scope alongside the granted one.

**Revision issues a new capability and the issuer is told**, with the revised
consent and the replacement capability, by the same path as the original grant.
This is not an exception to the protocol's silence: the issuer already knows
the relationship exists, the border would already answer its first out-of-scope
send with a refusal, and letting it find out by trial would turn a legitimate
tightening into blind debugging. Whoever wants the other side to know nothing
has the silent retirement path, which is where silence lives.

The previous capability dies immediately, by `scopeHash`: it describes an
agreement that no longer holds.

That choice has a consequence the specification must handle. The knocking party
receives a grant arriving from an opaque identifier and has no way, on its own,
to know whose it is. Therefore:

- The grant MUST carry `inReplyTo` with the identifier of the original
  `consent.request`.
- The knocking party's host MUST record, when sending a knock, which address it
  knocked.

Without both, the relationship ends up labelled by the opaque address instead of
by the counterparty, which defeats leak attribution: that property works only if
the user knows which relationship each address belongs to.

Clients MUST distinguish the two directions when displaying. The relationship
address belongs to the granting party's host, so on one side it is "the address
I issued" and on the other it is "the address I was given". Saying the same
sentence in both cases confuses precisely the user who is trying to work out
which is which.

# Delivery Enforcement

At delivery time the receiving host MUST verify, before the envelope reaches the
box:

1. The target address exists and is `active`.
2. A grant exists for that address and is in state `granted`.
3. The envelope carries a capability matching that grant.
4. The payload type is within scope.
5. The volume and urgency ceilings are not exceeded.

Failing 1, 2, or 3, the host MUST respond `unknown-recipient` or `no-consent`
according to the rule in the security considerations below.

An address configured with `acceptsNewRequests: false`, which is the default for
relationship addresses, MUST also refuse `consent.request`. This is what makes a
scraped address worthless: without a capability it delivers nothing, and with
this flag it does not even accept a knock.

## The issuer never learns the root

`user`, and the signer in `proofs`, identify the person by that relationship's
**contact key**, in the form `contact:<multibase>`, with the signature produced by
that key. Implementations MUST NOT place the root `did:key` in the consent object,
in the capability, or in an entry addressed to the issuer.

The root `did:key` is global, stable, and by design does not rotate day to day.
While it travelled in those three objects, two issuers comparing records knew they
were talking to the same person, and the per-relationship address protected
nothing: the object the relationship creates undid the guarantee the relationship
existed to provide.

The cost is real and worth stating: the issuer can no longer prove to a third party
which global identity accepted. Dispute and audit rest on the relationship rather
than on the person, which suffices for billing, because what is disputed is the
contractual relationship and that is exactly what the contact key identifies.

# Series and Deduplication

`dedupKey`, chosen by the issuer, opaque, and stable per logical series, for
example `bill-<period>-<customer>`, carries semantics:

- A new offer with a `dedupKey` already seen, while the task is open, is a
  **revision**, equivalent to the `update` action: the reissued bill, the
  rescheduled visit. It is presented as an update, not as a new task.
- A revision MUST require fresh explicit acceptance when it alters the amount,
  the due date, the beneficiary, **or any payment-instrument field**.
  Registered payloads MUST declare which of their fields are immutable within a
  series.

  The earlier trigger was only "amount or due date changed for the worse", and
  it let through exactly the fields the person uses to pay. Swapping only the
  instrument kept amount, due date, and holder intact, fired nothing, and
  entered as an update to a task already checked and accepted. Changing the
  instrument counts as a change for the worse by definition, because the money
  starts going somewhere else.
- A completed or cancelled task, plus the same `dedupKey` in a new billing
  period, is a new task of the series, September's bill after August's,
  inheriting the `next` relation from the previous one.
- An identical resend, with the same envelope `id`, is discarded by
  idempotency.

# Ending a Relationship

Three paths exist, and each exists for a reason.

**Ending a relationship between people.** A person-to-person relationship creates
no consent object, so neither path below reached it, and it was left with no
specified way out at all. Every acceptance of a personal relationship MUST create a
`Relation` object holding `peer`, `alias`, `maxUrgency`, `escalation` and `state`.
A `relation.revoke` signed by whoever accepted ends it, with immediate effect at the
edge and a Remove from that relationship's MLS group, after which the host MUST
answer that peer exactly as it would a nonexistent address. Ending is silent, like
blocking.

`escalation` says whether this person MAY name you as their escalation contact.
It starts **false**, and only whoever accepted the relationship raises it, by the
same principle as `maxUrgency`: whoever bears the consequence decides. The two are
separate on purpose, because one means "may wake me" and the other means "may wake
me on someone else's behalf".

Whoever revokes MAY, in the same act, generate a dedicated contact address for
future personal relationships, so as not to depend on the primary handle.
Clients SHOULD offer this to someone revoking because of harassment.

Retiring the address does not substitute for this: people invited personally use
the primary handle, so retiring it would sever every personal relationship at once
and break the handle proof. The adversary here is the ex-partner and the harasser,
and the protocol was offering them a channel with no off switch.

The two paths for issuers:

**Polite revocation** (`consent.revoke`): the issuer is informed, stops sending,
and the relationship can be resumed later. This is the normal path with a
counterparty that behaves.

**Silent retirement**: the address is retired without notice. Nothing is
communicated, there is no unsubscribe request, and there is no link that
confirms the recipient exists. The host answers as though the address had never
existed.

The distinction matters because the two situations are not the same. Against a
counterparty that respects revocation, informing it is correct and preserves the
option to resume. Against one that ignores revocation, that leaked the address,
or that should never have received it, an unsubscribe request is an interaction,
and an interaction confirms the address is live. Silent retirement is the
difference between asking to leave and simply ceasing to be there.

Silent retirement composes with address rotation, specified in the identity
document: rotating all addresses while carrying forward only selected
relationships accomplishes, in one operation, what unsubscribing from each
counterparty individually cannot.

# Issuer Duties

The general list of issuer duties lives in the anti-abuse document. Two belong
here, because they are about the consent relationship itself:

- **Respect scope and rate.** Violations produce `over-rate` and `no-consent`,
  and they are a signal to the recipient, not only an error to the issuer:
  clients SHOULD suggest revocation when they occur.
- **Honour a `decline` of a recurring series by offering to end it.** Three
  declines in a row SHOULD trigger a `consent.pause` suggested to the user.

# Security Considerations

## Identical responses

An address that was retired, revoked, frozen, or never created MUST produce
identical responses: same code, same body, comparable timing. Any distinction
becomes a validation tool for address lists.

This property belongs to the **set** of public endpoints, not to any one of
them. The reference implementation leaked here: delivery correctly refused for
an account undergoing deletion while the handle-proof endpoint continued serving
that account's identity document, so comparing the two revealed the account's
existence and state. One honest endpoint and one distracted endpoint add up to a
whole oracle.

## The capability is a bearer credential

A capability presented at delivery authorises delivery. Implementations MUST
bind it to a single address and MUST verify that the presented capability is the
one issued for that relationship, not merely that some capability is present.
Checking only for presence permits any party that reaches the address to
deliver.

This requirement is stated forcefully because the reference implementation
violated it while the text above was already written. The edge checked that the
`capability` field existed and did not compare its value, and a benchmark
measuring attacker economics delivered 119 of 300 tasks into a victim's box
using an invented string. The property that a harvested address is worthless
without the capability was absent for as long as that check was a presence test.

Two further requirements came out of the same measurement:

1. **Comparison in constant time.** Returning at the first differing byte makes
   the time to refusal depend on how many leading bytes the guess got right,
   and a bearer credential recovered byte by byte costs sixteen guesses per
   position.
2. **Never comparison by substring.** A query with `LIKE '%value%'` makes a
   one-character credential match any credential that contains it, and becomes
   an oracle: it answers "is there a live credential with this sequence?" and
   lets the credential be rebuilt piece by piece.

Refusal MUST return the same error as a nonexistent relationship. Distinguishing
a wrong capability from an absent relationship tells a prober that the address is
live.

## Consent is not permission to enumerate

A grant authorises delivery to one address. It does not authorise the issuer to
probe other addresses at the same host, and hosts MUST NOT reveal, in any
response, whether a different address exists.

# Privacy Considerations

The consent record is, simultaneously, the auditable processing record that data
protection regimes require: specific purpose, granular scope, term, revocation
as simple as granting, and a signed trail of state changes. Accountability falls
out of the design rather than being bolted on.

Two-issuer linkage is prevented by default: each issuer knows a different
address, so two issuers cannot determine that they are talking to the same
person without out-of-band information. This is the property that undermines the
data-broker practice of cross-referencing databases by shared identifier.

# Open Questions

1. Delegated consent, the adult child managing an elderly parent's consents,
   depends on the management delegation left open in the identity document.
2. Transfer of issuer ownership, a company merger or a domain change, needs a
   chain of continuity proofs.
3. Whether the `receipt` level includes the reason for a refusal. The proposal
   is that refusal is always without a reason by default.
4. Blocking an identity that already holds a granted relationship through
   another address. The provisional reading is that blocking does not cut what
   was already granted, because cutting is a different action with a different
   consequence, and mixing the two would make one tap do two things. What
   remains open is whether the interface should offer both together in that
   case.
5. **In which field the offer's purpose travels.** The anti-abuse document
   requires every offer to declare the purpose it was sent under, and calls
   sending outside it a violation, not a grey area. No document defines where
   the purpose goes: the origin block in the data model document carries
   `issuer`, `consent`, `offeredAt`, and `envelope`, and not the purpose.
   Without the field, the edge can enforce payload type, urgency ceiling, and
   volume, but not `purposes`, which is exactly the item the conformance rule
   calls a violation and not a grey area. The proposal is
   `ework:origin.purpose`, a string from the registered vocabulary, mandatory
   when the consent declares more than one purpose. It stays open because it
   touches the data model, and deserves deciding together with whether the
   purpose also enters the capability's signature, the `scopeHash`.

# IANA Considerations

This document requests the creation of a registry, "EWP Consent Purposes", with
Specification Required as the registration policy, initially containing
`billing`, `scheduling`, `delivery`, `service-order`, `legal-notice`, and
`document-renewal`.

Purposes outside the registry remain usable as free-form strings. The registry
exists so that clients can present familiar purposes consistently, not to
restrict what parties may request.
