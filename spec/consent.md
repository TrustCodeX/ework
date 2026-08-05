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
  "user": "did:key:z6MkCleia...",
  "alias": "k7f3a2q9@ework.example",
  "issuer": "billing@utility.example",
  "scope": {
    "purposes": ["billing"],
    "payloadTypes": ["ework.dev/payload/payment@1"],
    "maxPerMonth": 4,
    "maxUrgency": "normal",
    "recurringSeries": true,
    "regulatoryProfile": "br-lgpd@1"
  },
  "statusPolicy": "receipt",
  "grantedAt": "2026-08-04T11:58:00Z",
  "expiresAt": null,
  "state": "granted",
  "proof": { "type": "ed25519-jcs-2026", "by": "did:key:z6MkCleia...", "sig": "..." }
}
~~~

The object is signed by the recipient, not by the host. A host that could forge
a grant could authorise delivery to its own users, which would make the whole
mechanism decorative.

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

The protocol default is `receipt`.

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

The receiving host MUST place the request in quarantine and MUST NOT generate a
notification. Quarantine is visible on demand; it does not interrupt. A
mechanism whose purpose is to protect attention cannot itself demand attention
on arrival.

The host MUST replace any earlier quarantined request from the same issuer
rather than accumulate them, so that repeated knocking gains the issuer nothing.

## Rate limits on knocking

Hosts MUST rate limit `consent.request` by origin domain rather than by IP
address. In federation what matters is who is sending, and the address is
commonly a shared proxy.

Refusal MUST be explicit, carrying `over-rate` and `retryAfter`. Silent discard
would leave the sender unable to distinguish a limit from a loss.

# Granting

On granting, the recipient's client:

1. Generates a new address for this relationship, opaque and unlinkable to the
   recipient's handle.
2. Constructs and signs the consent object.
3. Issues a capability bound to that address and that grant.
4. Sends `consent.grant` to the issuer.

The grant is sent **from the newly created address**, not from the recipient's
principal handle. This prevents an issuer from learning the principal handle
merely because contact was accepted.

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

# Ending a Relationship

Two paths exist, and both exist for good reasons.

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

# IANA Considerations

This document requests the creation of a registry, "EWP Consent Purposes", with
Specification Required as the registration policy, initially containing
`billing`, `appointment`, `delivery`, `document-review`, and `incident`.

Purposes outside the registry remain usable as free-form strings. The registry
exists so that clients can present familiar purposes consistently, not to
restrict what parties may request.
