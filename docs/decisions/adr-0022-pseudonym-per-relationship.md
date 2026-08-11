# ADR-0022: The issuer knows the contact key, never the root

**Status:** accepted | 2026-08-08 | Michel Wilhelm

## Context

ADR-0008 established the per-relationship address, and the argument that justifies it is explicit: bring down the database cross-referencing industry. EW-RFC 0006 §5 repeated the promise in so many words, saying that the issuer validates who it talks to without learning the root identity, and that two issuers cannot correlate the same user.

The security review of 2026-08-08 showed the promise was false in the very text that made it. The credential presented in the MLS group was only one of the paths, and the other three were open:

- the `Consent` object (EW-RFC 0007 §1) carried `user` and `proofs[].by` as the root `did:key`;
- the sending credential (EW-RFC 0007 §4) carried the same `did:key`;
- the `Entry` directed to the issuer (EW-RFC 0015 §1) carried `author` in the same format.

A global, stable identifier in any one of those places is enough. The root `did:key` is global, it is stable, and by decision of EW-RFC 0003 §7.5 it does not rotate in day-to-day use. Two issuers comparing databases, or a broker buying both, know they are talking to the same person, and the per-relationship address stops protecting what it exists to protect.

The choice was binary and could not be postponed: either unlinkability is a requirement and the objects change, or it leaves the documents and becomes a declared risk.

## Decision

**Unlinkability is a requirement, and no object destined for an issuer carries the root.**

1. `Consent.user` and `Consent.proofs[].by` come to identify the person by the contact key of that relationship, in the form `contact:<multibase>`, with the signature produced by that key.
2. The sending credential of EW-RFC 0007 §4 follows the same rule.
3. `Entry.author`, when the entry is published in a relationship group, is the identifier of the contact key, and `proof.by` is the contact key itself. In a personal group and in a project circle, it remains the main identity.
4. Implementations MUST NOT put the root `did:key` in any object destined for an issuer.
5. The contact key travels in the `consent.grant`, bound by signature to that consent, and it is what the counterparty uses to validate the credential of the relationship group. It stays outside the public identity document.

Item 5 resolves, as a by-product, a second finding of the same review: the contact key had no validation chain at all, and the only thing attesting who was on the other side of the group was the host that delivered it, which is exactly who the design wants to exclude.

## Alternatives considered

**Accept linkability and withdraw the promise from the documents.** It was the honest and cheap alternative: erase the claim in EW-RFC 0006 §5 and item 4 of EW-RFC 0003 §6, and record the correlation as a residual risk. Rejected because it contradicts the reason ADR-0008 and ADR-0009 exist, and because the damage is worse than in email: here the identifier is global, stable and does not rotate, while an email address can at least be replaced. Keeping the per-relationship address mechanism without the property it promises would be paying the complexity and buying nothing.

**Rotate the root frequently to shrink the correlation window.** Rejected because the root is the continuity anchor of the identity, and rotating it frequently breaks what it exists to sustain, without eliminating the correlation: two issuers that collect in the same period still cross their databases.

**Zero-knowledge proofs linking the contact key to the root without revealing it.** Rejected for v0.1 because the cost of implementation and of cryptographic review is disproportionate to the problem, and because the directed delivery in the `consent.grant` resolves the concrete case. It stays recorded as a possible evolution if a requirement appears to prove to the counterparty that two relationships belong to the same person without revealing which one.

**A random identifier per relationship, tied to no key at all.** Rejected because the issuer needs to validate the signature of whoever talks to it, and an identifier with no key behind it validates nothing. The contact key already existed in EW-RFC 0003 §6 and serves exactly that purpose.

## Consequences

The promise of EW-RFC 0006 §5 becomes true, instead of merely asserted.

The cost is real and worth stating clearly: **the issuer can no longer prove to a third party which global identity accepted that consent.** Dispute and audit come to rest on the relationship, not on the person. For billing that is enough, because what is disputed is the contractual relationship, and the contractual relationship is exactly what the contact key identifies. For a case that requires linking the acceptance to a civil registry, the answer remains the one in ADR-0010: the protocol does not index people by civil registry, and whoever needs that solves it outside the protocol.

Whoever stores users' `did:key` in an issuer database today needs to migrate. Since the specification is a draft and the reference implementation is the only one in production, the cost is that of one migration, not of an ecosystem.

The same user with two relationships with the same issuer, through two different addresses, appears as two people to that issuer. That is the desired behaviour, and clients need to explain it to whoever finds the duplication odd.

It remains open how a person proves continuity to the issuer when migrating hosts or rotating the contact key. EW-RFC 0003 §7.3 already covers selective migration of a relationship, and the continuity proof now needs to be produced by the old contact key, not by the root.
