---
docname: draft-wilhelm-ework-crypto-00
title: "The e-work Protocol (EWP): Cryptography and Encryption Modes"
abbrev: EWP Crypto
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC9420,RFC8032,RFC8785,RFC8446
---

This document specifies the cryptographic layer of the e-work Protocol (EWP):
the signature scheme, the two encryption modes, the mapping of protocol groups
onto MLS {{RFC9420}} groups, and an exhaustive statement of the metadata a host
observes in each mode.

<!-- abstract -->

# Introduction

EWP defines two modes and requires that a host declare which one it operates.
The distinction is invisible in a user interface, so a protocol that allowed it
to be implicit would let every implementation claim the stronger one.

## Conventions and Definitions

<!-- rfc2119 -->

# Modes

**End-to-end encrypted mode.** Content is encrypted to a group; the host stores
and forwards ciphertext it cannot read. This is the target state and the default
the protocol is designed around.

**Assisted mode.** The host reads content. This exists because it is what makes
several things possible before group encryption is deployed: server-side search,
recovery when a user loses every device, bridges to systems that do not speak
EWP, and simpler operation for a host serving people who prefer that trade.

A host MUST advertise its mode in the discovery document. A client MUST show the
mode to the user. **An implementation MUST NOT describe assisted mode as
providing confidentiality against the host operator**, because that is precisely
what it does not provide.

# Signatures

Signing is Ed25519 {{RFC8032}} over the canonical form {{RFC8785}} of the object
with proof fields removed. This applies uniformly: identity documents, consent
objects, entries, and envelopes.

Independent implementations MUST produce byte-identical canonical forms.
Canonicalisation is the part of a signature scheme that fails silently in
practice, and the only reliable test is cross-implementation verification.

# Groups

In end-to-end encrypted mode, each context is an MLS {{RFC9420}} group.

| Context | Members | Purpose |
|---|---|---|
| Personal | All devices of one identity | Multi-device synchronisation |
| Relationship | Two parties | Everything in one consent relationship |
| Project compartment | Members of one circle | Compartmentalisation |

MLS provides forward secrecy, post-compromise security, and efficient membership
change. Adding a member is a group operation; removing one rekeys the group, so
the removed member cannot read subsequent content.

Whether history is re-shared with a joining member is a decision of the group
owner and MUST be explicit. Both behaviours are legitimate and they have
different consequences.

# Cryptographic Agility

The ciphersuite is negotiated through MLS. EWP does not fix one, and
implementations MUST be able to change it without a protocol version change.

A post-quantum hybrid suite, combining ML-KEM with X25519, is the target as soon
as it stabilises in the MLS ecosystem. Stating this here is not a promise of a
date; it is a statement that the design must not make the change hard, which is
why no ciphersuite is written into the protocol.

# Metadata Visible to the Host

This section is exhaustive by intent. A privacy claim that omits what leaks is
worse than no claim.

In **end-to-end encrypted mode** the host observes:

- Which addresses exchange envelopes, and in which direction.
- Timing and frequency of exchanges.
- Approximate envelope size, reduced by padding to size classes.
- Envelope type, since it is in the header and routing depends on it.
- The coarse `urgencyHint`, where present.
- For scheduled release, the pair (destination, release time), with neither
  content nor reason.
- Blob sizes and the set of hosts to which each blob has been granted.

In **assisted mode** the host additionally observes everything above plus the
content: task titles, descriptions, payloads, attachments, and entries.

Padding to size classes reduces the size channel. The remaining channels are
inherent to store-and-forward and cannot be removed by any cryptographic
measure, only by mixing or cover traffic, which EWP does not specify.

# Key Management

Root keys are derived from a recovery kit and stay offline. Device keys live on
devices. Contact keys derive relationship addresses. The layering, the rotation
authority, and the rule that a compromised key never authorises its own
replacement are specified in the identity document.

Backup of group keys is what makes recovery possible after losing every device,
and it is also the point at which an implementation can accidentally hand the
host everything. A key backup encrypted to the recovery kit preserves the
property; a backup the host can decrypt does not, and an implementation offering
the latter MUST describe it as assisted mode regardless of what else it does.

# Security Considerations

The protocol's confidentiality claims hold only in end-to-end encrypted mode.
Every statement about compartments, sealed sections, and content privacy is a
statement about that mode, and this document is the place where that dependency
is recorded once rather than repeated in each document.

Signature verification requires resolving the signer's identity document.
Implementations MUST verify that the key carried in a proof belongs to the
declared signer, and MUST NOT accept a proof on the basis of the embedded key
alone. The embedded key exists so that verification can proceed before
resolution, not so that resolution can be skipped.

# Privacy Considerations

The exhaustive metadata list above is the honest bound on what EWP protects. A
host that carries only ciphertext still builds a social graph from routing
information. Distributing addresses across hosts, specified in the identity
document, partitions that graph, and it is the only mechanism the protocol
offers against it.
