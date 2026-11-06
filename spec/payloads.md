---
docname: draft-wilhelm-ework-payloads-00
title: "The e-work Protocol (EWP): Typed Payloads"
abbrev: EWP Payloads
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC8259
---

This document specifies typed payloads for the e-work Protocol (EWP): the
mechanism by which a task carries structured domain content, the extension
model, and the rule that a client encountering an unknown type degrades to
something useful rather than failing.

<!-- abstract -->

# Introduction

A task about paying a bill and a task about confirming an appointment share
title, deadline, and state, and differ in everything else. Modelling that
difference in prose puts the parsing burden on the recipient. Modelling it in a
fixed set of task subtypes puts the burden on the protocol, which then needs a
new version for every domain.

Typed payloads take the third path: the task is generic, and it carries typed
content the way an email carries typed parts.

## Conventions and Definitions

<!-- rfc2119 -->

# Payload Structure

~~~
"ework:payloads": [
  { "type": "ework.dev/payload/payment@1",
    "data": { "amount": "412.87", "currency": "BRL",
              "dueDate": "2026-09-10",
              "beneficiary": { "name": "Utility Co." },
              "reference": "8829-4471",
              "pix": { "brcode": "00020126580014br.gov.bcb.pix...6304ABCD" } } }
]
~~~

`type` is a versioned identifier. `data` is an object whose shape the type
defines.

A task MAY carry more than one payload. Payload order is not significant. A
task MAY also carry no payload at all: "take the dog to the vet" is a complete
and legitimate task of the protocol.

# The Degradation Rule

**A client encountering an unknown payload type MUST still present the task.**
Title, deadline, issuer, and available actions come from the task itself and do
not depend on the payload.

This is the property that makes the extension model viable. If an unknown type
broke the task, every new type would be a breaking change for every client that
had not yet implemented it, and in a federated system with independent
implementations that is the same as saying no new types.

The dependency does not run the other way either: no core document MAY assume
the existence of a specific payload type. A core rule that only makes sense
for one domain is in the wrong place and MUST become a payload.

A client MAY show the raw payload, and SHOULD indicate that it does not
understand the type rather than pretending the task has no content.

# Naming

| Origin | Form |
|---|---|
| Core types | `ework.dev/payload/<name>@<major>` |
| Third-party types | reverse domain, `com.example.payload/<name>@<major>` |

The major version is part of the identifier. A backward-incompatible change is a
new identifier, so a client that understands `@1` is never handed `@2` content
under the same name.

Third parties MUST use a reverse-domain prefix. Registration is not required to
use a type, and the registry exists so that widely used types converge rather
than to gate what parties may exchange. A type enters the `ework.dev/payload/`
namespace only when two independent implementations are using it.

# Core Types

The following are defined by this document.

## payment@1

| Field | Required | Meaning |
|---|---|---|
| `amount` | yes | Decimal string, never a float |
| `currency` | yes | ISO 4217 code |
| `dueDate` | yes | Date |
| `beneficiary` | yes | Object with at least `name`; `taxId` carries the holder's tax identifier |
| `reference` | no | Issuer's reference for reconciliation |
| `boleto` | see below | Object with `linhaDigitavel` (the digitable line) and `barcode` |
| `pix` | see below | Object with `brcode` and `txid` |
| `charges` | no | Object with `finePercent`, `interestMonthlyPercent`, `discountUntil`, `discountAmount` |
| `recurrenceOffer` | no | Object with `mode`, `brcode`, `note`: an offer to enrol in a recurrence |
| `paidProofRequested` | no | Whether the issuer asks for proof of payment |

At least one instrument, `boleto` or `pix`, MUST be present.

`amount` is a string because a float cannot represent decimal money exactly, and
a protocol that transports money as a float will eventually transport the wrong
amount.

`charges` mirrors the charges of a bill with a due date (fine, interest,
discount), so the client can show the cost of being late.

`recurrenceOffer` replicates the Pix Automático journey of a charge plus
recurrence enrolment; the authorization itself happens at the bank, outside the
protocol.

The UI MUST display `beneficiary` prominently, and MUST warn when the
beneficiary diverges from the verified issuer of the offer. This is the central
anti-fraud rule.

The payload describes what is owed. **It does not carry payment credentials, and
it does not perform payment.** EWP transports the task; settlement happens in
whatever system the parties use. The typical actions are `deeplink` and `copy`,
and `paidProofRequested` asks for, never requires, proof of payment as an
attachment of kind `proof`.

### The instrument is pinned to the series

Within one `dedupKey`, the beneficiary identifier and every instrument field are
pinned. A revision altering any of them MUST require fresh explicit acceptance with
a strong warning, and clients MUST NOT offer one-tap acceptance in that case.

Pinning the beneficiary alone left the shortest path open: keep beneficiary, amount
and due date identical and change only the instrument. Nothing fired, because the
holder had not changed and the amount had not worsened, and the person paid the
attacker inside a task they had themselves already checked and accepted. Changing
the instrument counts as a change for the worse by definition, because the money
goes somewhere else.

### The action target comes from the payload

Clients MUST derive a payment action's target from the payload itself, and MUST NOT
offer a payment action whose target comes from the action list unless key,
beneficiary and amount match the payload. A mismatch MUST block the action rather
than merely warn.

Every defence here operates on the payload, and nothing bound the target of a
`deeplink` action to it. The issuer could send a correct beneficiary, correct tax
identifier and correct amount, with the primary action pointing at the attacker's
instrument: a conforming client displayed the legitimate beneficiary prominently,
found no discrepancy to warn about, and the button led to the wrong payment.

### Closed list of URI schemes

A `deeplink` target MUST use a scheme from the registered list (`https`, `pix`,
`tel`, `mailto`, `geo`), and clients MUST discard an action whose scheme falls
outside it while preserving the rest of the task. `javascript:`, `data:`, `file:`,
`intent:` and unregistered schemes MUST NOT be actioned under any circumstances.
Web clients MUST treat the target as data, never as a navigation attribute built
from sender-supplied text: in a client embedded in the host, `javascript:` in an
anchor element is script execution with access to key material and already
decrypted content.

## appointment@1

| Field | Required | Meaning |
|---|---|---|
| `slots` | yes | Array of proposed slots, each with `id`, `start` and `end` |
| `timeZone` | yes | IANA time zone name |
| `location` | no | Object with `name`, `address`, `geo` and `virtualUrl` |
| `professional` | no | Free text: who attends, and the specialty |
| `preparation` | no | Free text: fasting, documents to bring |
| `confirmBy` | no | Date-time limit for choosing a slot |
| `reschedulePolicy` | no | Free text: the issuer's rescheduling terms |

Acceptance picks a slot: the `accept` action carries the `slotId`. A counter
proposal (the `counter` action) MAY propose times outside the list.

Once accepted, the client SHOULD create the preparation subtask, with a
`finishtostart` dependency from the preparation to the appointment, when
`preparation` is present.

Rescheduling by the issuer is an `update` action on the same `dedupKey`, with
fresh acceptance.

## delivery@1

| Field | Required | Meaning |
|---|---|---|
| `window` | yes | Object with `start` and `end` |
| `timeZone` | yes | IANA time zone name |
| `address` | no | Free text: where to deliver |
| `items` | no | Array of items, each with `desc` and `qty` |
| `requiresPresence` | no | Whether someone must be there to receive |
| `contact` | no | Object with the carrier's `name` and `phone` |
| `trackingUrl` | no | Carrier tracking link |

## approval@1

| Field | Required | Meaning |
|---|---|---|
| `question` | yes | What is being decided |
| `options` | yes | Array of options, each with `id` and `label` |
| `attachmentRef` | no | Reference to the attachment under review |
| `decideBy` | no | Date-time limit for the decision |

The answer travels in an entry whose `complete` action carries the `optionId`.
It covers simple approvals: a design revision, a budget, a school
authorization. It is not a polling tool.

# Payloads and Consent

The consent scope lists `payloadTypes`: the types that issuer is allowed to
send. This is what keeps a grant meaningful, because consent to receive bills is
not consent to receive appointment reminders. What was missing was who checks,
and the question is not housekeeping: the type travels **inside** the task, the
task travels encrypted to the relationship's group, and the exhaustive list of
metadata a host observes does not include the payload type. Writing that the
receiving host refuses an out-of-scope payload would describe a host that reads
the task.

**Peers check it, in two moments that do not substitute for each other.**

- **Before sending**, an issuer MUST NOT emit an offer whose payload type falls
  outside the scope it holds. It knows that scope in full: it received the
  signed grant and the capability that summarises it in the `scopeHash`. The
  issuer's own host or gateway composes the offer in the clear, inside the
  signer's domain and before the envelope exists for anyone else, and it MAY
  refuse to emit by the same criterion. This is the only moment that spares the
  useless delivery, and it depends on the issuer being honest.
- **On receipt**, the recipient's client MUST refuse the out-of-scope payload
  and MUST NOT present the offer as an acceptable one. This depends on no
  issuer, and it is what makes the scope hold against one that ignores it.
  Accepting an out-of-scope offer regularises what the issuer had no right to
  send, so a screen presenting it with the usual button is itself the failure.

**Refusing is not making it disappear.** The refusal MUST be visible to the
recipient, naming the issuer, the type that arrived and the scope in force, and
clients SHOULD offer the two exits that already exist: revoking, and revising
the agreement to include that type. The common case is not a scam, it is the
clinic that started sending appointment reminders under a billing grant:
discarding in silence would leave the person without the appointment, unaware it
ever existed, and without the signal that this issuer sends outside what it
signed.

For the issuer the refusal produces no new signal: `receipt.delivered` already
left the edge, and what travels beyond it remains whatever the status policy
allows. The protocol emits no read receipts anywhere, and this is not where it
starts.

**Out of scope is not the same as unknown**, and conflating them undoes one side
or the other. An unknown type is a limitation of the reader, and the degradation
rule applies in full: the task appears, without the payload. An out-of-scope
type is a violation by the writer, and the task is not born. A client applying
degradation to the second case would deliver the task and call the violation an
incompatibility.

**The recipient's host is not where this is checked.** Under end-to-end
encryption it does not read the payload, and it does not judge what it cannot
see. Granting it that sight would put the payload type on the exhaustive
metadata list, and the price is in the privacy considerations below: the type
name, next to the sender's domain, is the subject matter. Where the host is a
member of the group by the owner's explicit act, in assisted mode, it is a
receiver like any other and what binds every receiver binds it, without that
becoming a guarantee: a check that exists only in some deployments is not a
property of the protocol.

**What the edge still enforces** is what it can see, and it remains what makes
consent a delivery gate: a live address, a grant in state `granted`, a
capability compared against the one issued for that relationship, and the volume
for the period. None of it depends on reading content, which is why none of it
disappears when the ciphertext arrives.

Conformance suites MUST test that an offer carrying an out-of-scope payload type
does not become a task and surfaces to the recipient as a violation of that
relationship, and MUST test it with the host outside the group.

# Payloads and Compartments

A payload MAY be sealed into a narrower compartment, as specified in the
compartments document. The sensitivity policy maps labels to compartments, and a
payment payload in a project labelled `money` routes to the financial
compartment automatically.

Every payload declares the default sensitivity of its fields, and clients MUST
apply it without asking when creating a task inside a project: the whole of
`payment@1` is `money`; `delivery@1` classifies `address` and `contact` as
`personal-data`; `appointment@1` classifies `professional` and `preparation` as
`health` when the issuer is a health issuer; `approval@1` classifies nothing.
Closing is automatic; opening is an explicit and recorded act.

# Security Considerations

Payload content is attacker-controlled input from the recipient's perspective.
Clients MUST validate against the type's schema before rendering, and MUST NOT
execute anything a payload contains.

A payload MUST NOT carry credentials, tokens, or secrets. The protocol offers no
mechanism to protect them beyond the transport and the encryption mode, and a
type that requires them is misdesigned.

Payment is the target. Beyond the beneficiary rule, clients MUST NOT accept a
`payment` payload under a consent that does not include that scope, and SHOULD
validate internal consistency, checking that the digitable line, the barcode
and the amount agree, before displaying payment buttons. That is the consent
rule above applied to the sharpest case, and it was already written this way,
at the client, before the rule existed: the payload that carries money is the
one that can least afford a check that vanishes when the ciphertext arrives. In
a multi-party project, a payment payload created in the clear in the broad
compartment is a leak no configuration repairs afterwards, and the conformance
suite tests that it is born sealed.

Scope enforcement that works only in an unencrypted deployment is worse than no
enforcement: it is true in the demo and false in production, and it teaches
clients to rely on a control that disappears exactly when the protocol's
guarantees arrive. It is the same reason the required justification lives with
peers in the history document.

# Privacy Considerations

The payload type travels in the task, and in assisted mode the host reads it. A
type name can itself be sensitive: `ework.dev/payload/appointment@1` from a
clinic reveals something even without the content. The compartments document
specifies how to seal payloads away from parties who should not see them, and
the cryptography document states what the host observes in each mode.

**The type name is content, which is why it stays off the observable metadata
list.** Next to the sender's domain it is the subject matter of the task. Putting
it in the clear so the host could enforce the scope would hand over the category
of everything that arrives, in every relationship, permanently, and would buy
with it one identified peer's check against another, inside a relationship the
person already authorised and can sever. The host routing the bill does not need
to know it is a bill.

# Open Questions

1. International `payment`: `pix` and `boleto` are Brazilian. Generalize with
   per-country sub-schemas, or keep separate national payloads?
2. A `medication@1` payload (structured dosage) for the medication scenario:
   core, or an extension from the health community?
3. Formal validation (a JSON Schema published per payload) as part of the
   registry.
4. Can the check be returned to the edge without naming the type? The obvious
   construction is an opaque per-relationship commitment: the grant issues one
   value per permitted type, the envelope carries the one it uses, and the host
   compares without knowing what it compares. It looks attractive and probably
   is not: the host stores the grant, so it knows the list of permitted types,
   and the presented value becomes an index into it. With one type in scope the
   commitment tells the host nothing it does not already know and catches no one
   the capability comparison does not already catch; with several, it starts
   telling the host which one each envelope is, which is exactly what the rule
   above declines to hand over. What stays open is whether a better construction
   exists, not the proposal.
5. The refusal offers "revise the agreement to include this type" one tap away,
   and that tap is a scope widening decided under the pressure of a task the
   person wants to see. It is consent bundling through the back door, with the
   issuer choosing the moment. What remains to be decided is whether revising
   from a refusal requires the full granting screen, with the whole scope in
   view, or whether the warning suffices.

# IANA Considerations

This document requests the creation of a registry, "EWP Payload Types", with
Specification Required as the registration policy, initially containing
`payment@1`, `appointment@1`, `delivery@1`, and `approval@1` under the
`ework.dev/payload/` prefix.
