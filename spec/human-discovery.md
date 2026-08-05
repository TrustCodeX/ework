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

The mechanism is blind delivery against a salted, pepper-protected hash. A party
that already knows the identifier can reach its holder; a party that does not
cannot obtain a list.

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

# Verified Attributes

An identity MAY associate verified attributes with itself:

~~~
{
  "@type": "VerifiedAttribute",
  "kind": "phone",
  "hash": "sha256:...",
  "verifiedAt": "2026-08-04T00:00:00Z",
  "verifier": "did:key:z6MkHost...",
  "proof": { "sig": "..." }
}
~~~

The attribute value is never published. Only the hash is, and the identity
document lists which kinds exist rather than their values.

Verification is performed by a party the recipient trusts, typically the host,
through the ordinary channel: a code sent to the telephone number or email
address.

# Blind Delivery

To reach `+5511999999999` without learning who holds it:

1. The sender computes `H = hash(normalise(identifier), salt, pepper)`.
2. The sender POSTs a `consent.request` to a discovery endpoint, addressed to
   `H`.
3. The host matches `H` against its verified attributes. On a match, the request
   enters the recipient's quarantine. On no match, the response is identical.

**The response MUST be identical in both cases**, in code, body, and comparable
timing. A difference makes the endpoint an enumeration oracle, which is the
failure this design exists to avoid.

## Salt and pepper

The salt is per-identity and public. The pepper is per-host and secret.

The pepper is what makes offline enumeration infeasible. Telephone numbers
occupy a space small enough to hash exhaustively; with a secret pepper, an
attacker who obtains the host's attribute table still cannot invert it without
also obtaining the pepper, which is held separately.

Hosts MUST rate limit the discovery endpoint aggressively, by origin domain, and
SHOULD apply a deliberate constant delay. Online enumeration is bounded by rate
limiting; offline enumeration is bounded by the pepper. Both are required.

## Normalisation

Implementations MUST normalise before hashing: E.164 for telephone numbers,
lower-cased and trimmed for email addresses. Without normalisation the same
identifier hashes differently depending on how it was typed, and blind delivery
silently fails.

# What the Recipient Sees

A request arriving through blind delivery is indistinguishable, in the
recipient's quarantine, from one arriving to the principal handle, except that
it records which attribute matched. The recipient learns that someone who knows
their telephone number is asking for contact.

The recipient MAY disable an attribute for discovery at any time, in which case
matches stop, and MUST be able to see how many discovery attempts matched, so
that an increase is visible.

# Record of Attempts

Hosts SHOULD record discovery attempts that matched, under the recipient's
control, and SHOULD retain them for the minimum necessary.

The record serves the recipient: a sudden rise in attempts against a telephone
number is evidence that the number appeared in a leak. It also serves the
regime-mandated right to be informed about processing, mapped in the regulatory
document.

# Security Considerations

The design's security rests on three things together, and removing any one
defeats it: identical responses, aggressive rate limiting, and the secret
pepper.

A host that leaks its pepper turns its attribute table into a reversible index
of every telephone number and email address it holds. Hosts MUST store the
pepper separately from the attribute table, and rotating it invalidates existing
hashes, which requires re-verification.

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
