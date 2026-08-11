---
docname: draft-wilhelm-ework-human-discovery-00
title: "The e-work Protocol (EWP): Discovery by Human Identifiers"
abbrev: EWP Human Discovery
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC8032
---

This document specifies how an e-work Protocol (EWP) identity can be reached
through identifiers people already know, such as a telephone number or an email
address, without those identifiers becoming a directory that can be enumerated.

The mechanism is blind delivery: a directory forwards a consent request
addressed to an attribute and never translates the attribute into an address. A
party that already knows the identifier can reach its holder; a party that does
not cannot obtain a list.

<!-- abstract -->

# Introduction

A protocol whose only address is a public key is unusable for first contact
between people. A protocol with a searchable directory of telephone numbers is a
harvesting target, and several messaging systems have demonstrated exactly how
that ends.

EWP takes a narrow position: verified attributes, blind delivery, no catalogue.

**Civil registry identifiers are out of scope.** National identity numbers,
taxpayer numbers, and their equivalents are deliberately not supported as
lookup keys. They are unique, permanent, and impossible to rotate, which makes
them the perfect correlation key across every database that holds them, and a
protocol that indexes people by them becomes the join key for everything else.

## Conventions and Definitions

<!-- rfc2119 -->

# Identifier Types

| Type | Example | Role |
|---|---|---|
| Root key | `did:key:z6Mk...` | the identity. Never shown to an ordinary user |
| Handle | `cleia@ework.example` | the readable form, hides the key, allows a managed server |
| Contact address | `k7f3a2q9@ework.example` | what each issuer knows, one per relationship (the identity document) |
| **Verified attribute** | telephone, email | what the world already knows, used only for first contact |

An attribute is **not an identity**: it is a revocable binding. Implementations
MUST NOT require an attribute to create or use an identity. Nobody needs to
hand over a telephone number to exist in EWP, a deliberate difference from the
messengers that tie an account to a number.

Attribute namespaces, a closed list in this version:

- `tel`: full E.164, with country and area codes (`+554999990000`).
- `email`: a normalised email address.

**Civil registry identifiers (national identity numbers, personal taxpayer
numbers, electoral registrations, and their equivalents) MUST NOT be used as
discovery attributes.** The reasoning is given later in this document.
Registering a namespace for them is non-conformance, not extension.

# Verified Attributes

Binding an attribute requires an attestation signed by a **verifier**, never
self-declaration. **Who counts as an acceptable verifier has to be written down, or
anyone is one.** Directories MUST maintain a published, auditable list of accepted
verifiers, and MUST NOT accept an attestation signed by a key outside it. Clients
MUST display which verifier attested a binding, and MUST treat an unknown verifier
as a strong warning.

Without that rule, anyone able to publish a domain declared themselves a verifier,
attested someone else's phone number, and hijacked first contact directed at them.
Requiring the attestation while leaving its issuer unconstrained is the same as not
requiring it.

The attestation format:

~~~
{
  "@type": "AttributeAttestation",
  "attribute": "tel",
  "valueHash": "sha256:...",
  "identity": "did:key:z6MkCleia...",
  "verifier": "verifier.example",
  "assurance": "substantial",
  "method": "otp-sms",
  "issuedAt": "2026-08-04T12:00:00Z",
  "validUntil": "2027-02-04T12:00:00Z",
  "proof": { "by": "did:key:z6MkVerif...", "sig": "..." }
}
~~~

- `assurance` is one of `low`, `substantial`, `high`, in the spirit of eIDAS. A
  one-time code by SMS or email is usually `substantial`.
- `valueHash` holds the hash of the value with a salt carried by the attestation
  itself: the attestation circulates without the number in the clear.
- **Attestations expire** (SHOULD: 6 months for `tel`, 12 for `email`).
  Telephone numbers are recycled by carriers, and an eternal binding to a
  recycled number is an account hijacking vector (threat model A15).
- The user's own host MAY act as verifier for `tel` and `email`, which avoids a
  third-party dependency for the simple case.

# Blind Delivery

The directory **never translates an attribute into an address**. It accepts a
consent request addressed to an attribute and forwards it, if it knows the
binding.

To reach `+5511999999999` without learning who holds it:

1. The sender submits a `consent.request` addressed to the attribute (as a
   hash), together with its own verified identity.
2. The directory answers "accepted for forwarding", always the same answer.
3. If the directory knows the binding, it forwards the request to the holder's
   host, signing as directory, and the request lands in the recipient's
   quarantine marked "arrived through your telephone number". If not, it
   discards the request.
4. On acceptance, the grant creates a fresh contact address and the sender
   receives a send capability for it. From then on the parties talk directly,
   and the directory leaves the path for good.

Normative rules:

1. The directory **MUST answer identically** whether the binding exists or not:
   same code, same body, same latency. The answer means "accepted for
   forwarding", never "this person exists".
2. The directory **MUST NOT** expose reverse lookup (address to attribute),
   bulk export, or list checking ("which of these ten thousand numbers have an
   account").
3. Only **verified requesters** (an organisation with a domain-anchored
   identity, as in the identity document) MAY submit, with a quota per period,
   and the directory MUST log every submission.
4. The forwarded request carries the same minimal `consent.request` body of the
   consent document: no real payload, no payment link, no attachment.
5. After acceptance, the relationship uses the direct contact address. The
   directory **MUST NOT** be consulted again for that relationship, and takes
   no part in anything from then on.
6. Refusal and silence are indistinguishable to the requester: whoever asked
   cannot tell whether they were refused, whether nobody looked, or whether
   the binding does not exist.

## Storage at the directory

The directory stores `HMAC(directory_key, attribute_value)` pointing to a
**routing token encrypted** to the public key of the user's host. A leak of the
database reveals neither the attributes in the clear nor the destinations, and
the directory cannot assemble the list of who has an account without also
holding the HMAC key.

Declared limitation: a directory compromised **while operating** can test
guesses (the space is small) and observe who tries to reach whom. No magic
fixes that. The mitigations are structural: splitting across several
directories, the binding never becoming an address, the directory leaving the
path after first contact, and the user seeing every attempt.

The most defensible design is the **user's own host as their directory**:
whoever controls the host already knows the addresses of that box, so the
binding creates no new knowledge anywhere. That works when the requester can
discover the host from the attribute, which is not always the case, and so
independent directories remain part of the design. See the open questions.

## Normalisation

Implementations MUST normalise before hashing: E.164 for telephone numbers,
lower-cased and trimmed for email addresses. Without normalisation the same
identifier hashes differently depending on how it was typed, and blind delivery
silently fails.

## Forwarding is the one transport exception

The federation document binds the transport signature to the `hostKey` of the
domain in `from`, and in blind forwarding the envelope arrives with the issuer's
`from` and the directory's transport signature. Left unwritten, implementations
would open that hole on their own, each differently, and such a hole accepts an
envelope from anything presenting itself as a directory.

The exception is normative and narrow. Hosts MUST accept a third party's transport
signature **only** for `consent.request` envelopes arriving by the forwarding path,
and **only** from directories the host recognises, by its own operator-configured
list. The forwarded envelope MUST carry the directory's identity in `forwardedBy`,
and the author signature inside the object remains mandatory and is what proves
origin. Hosts MUST NOT accept any other envelope type from a directory, and MUST
NOT accept a forwarded `consent.request` addressed to a contact address, only to
the attribute bindings of this specification. Clients MUST display which
directory forwarded.

# What the Recipient Sees

A request arriving through blind delivery is indistinguishable, in the
recipient's quarantine, from one arriving to the primary handle, except that
it records which attribute matched. The recipient learns that someone who knows
their telephone number is asking for contact.

The recipient MAY turn an attribute back off at any time, in which case matches
stop, and MUST be able to see every attempt, so that an increase is visible.

# Off by Default

No attribute is discoverable until the user turns it on, per attribute and per
requester class:

~~~
"discovery": {
  "tel": { "enabled": true, "allow": "existing-relationship", "directories": ["directory.example"] },
  "email": { "enabled": false }
}
~~~

`allow` is one of `nobody` (the default), `existing-relationship` (the requester
presents proof of a prior relationship), `verified-org` (any verified
organisation), `anyone`.

# Choice of Directories

The user chooses which directories each binding is published to, including
none. Splitting bindings across different directories also splits the
knowledge: none of them sees the whole set.

# Record of Attempts

Hosts MUST keep, and clients MUST display, the record of attempts: who tried,
through which attribute, when, and what happened. The record is what turns an
attempted abuse into evidence, and what lets the user turn an attribute off
before the problem grows.

The record serves the recipient: a sudden rise in attempts against a telephone
number is evidence that the number appeared in a leak. It also serves the
regime-mandated right to be informed about processing, mapped in the regulatory
document.

# Revocation

Unbinding an attribute is immediate and does not require the verifier. From
revocation on, requests addressed to that attribute have nowhere to go.

# Why Civil Registry Identifiers Stay Out

A civil registry number would be, at first sight, the ideal attribute: it is
what the company actually holds when it issues the bill, and it is unique per
person and per country. The exclusion is deliberate and has three reasons, in
order of weight:

1. **It is permanent and immutable.** The whole identity design of EWP
   (per-relationship addresses, cheap rotation, unlinkability) exists so that
   no unique identifier follows the person everywhere. Binding the identity to
   a number that accompanies them until death recreates exactly the crossing
   point the protocol tries to eliminate.
2. **Incident mitigation does not exist.** A compromised telephone number can
   be replaced; an email can be replaced. A civil registry identifier cannot.
   A mechanism whose incident response plan is "replace the identifier" cannot
   rest on something that cannot be replaced.
3. **It is the key through which the state sees the person.** A binding
   between an EWP identity and a civil registry, even confined to one
   directory, is a lever whose pressure to be used would always come from
   whoever has more power than the holder.

This is not the enumeration argument: blind delivery already neutralises that,
and by that metric a civil identifier would be as safe as a telephone number.
It is the nature of the identifier that decides.

Assumed consequences, stated plainly: remote first contact does not work when
the company only holds the civil number, and direct governmental use by civil
registry is out. A public body participates like any issuer, obtaining consent
through the same paths as everyone else. A jurisdiction that wants compulsory
delivery by civil identifier will have to do it in a national profile of its
own, outside the core, publicly owning the cost.

This does not affect the tax ID of **organisations** (the identity document,
the payloads document): a company registration number identifies a legal
entity in a public registry, appears on the bill, and is the payee's
anti-fraud check. A company has a street address; a person does not need to
have one.

# Security Considerations

The central risk is enumeration, and the defence is not cryptographic, it is
design: since no query returns an address, enumerating produces nothing beyond
requests thrown into the void. Equality of responses has to include time, and
implementations MUST test for it, because a latency difference is as good an
oracle as a code difference.

A binding to a recycled telephone number is the classic hijacking vector.
Mitigations: attestations expire, and a telephone binding MUST NOT, on its own,
authorise account recovery, which continues to depend on the recovery kit (the
identity document).

The directory is the most sensitive piece of the system and the only one where
the choice of operator matters. It deserves written governance, auditing, and
partitioning, and none of that is a problem the specification solves alone.

Discovery does not authorise delivery. A successful match places a
`consent.request` in quarantine and nothing more. Everything the consent
document specifies still applies.

# Privacy Considerations

The mechanism deliberately trades a small disclosure for usability: a sender who
already knows an identifier learns whether contact is possible. That is the
minimum required for the mechanism to work at all, and it is disclosed to
someone who already had the identifier.

What it refuses to trade is enumeration. There is no endpoint that returns a
list, no endpoint that confirms existence without a matching identifier, and no
identifier that cannot be disabled or rotated. The exclusion of civil registry
identifiers follows the same reasoning, and is recorded as a design decision
rather than an oversight.

# Open Questions

1. Does a directory operated by the user's own host solve more than it
   complicates? It needs a way for the requester to discover the host from the
   attribute without recreating the oracle, which may be impossible. This is
   the most interesting open question in this document.
2. Governance of independent directories: who operates, who audits, who
   answers for a leak. The answer changes the threat model.
3. Proof of a prior relationship (`allow: "existing-relationship"`): what form
   it takes without becoming a new vector, since a contract number is
   guessable in many cases.
4. Cost per submission for requesters without history: is a quota enough, or
   is a real cost needed?
5. Telephone re-verification and portability: at what interval, and what
   happens to the relationships when the binding expires without renewal.
6. The legal basis (under the LGPD) for the directory operator's processing of
   the binding.
