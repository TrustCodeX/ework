# ADR-0010: Civil identifiers out of the scope of discovery

**Status:** accepted (2026-08-04, decided with Michel). Supersedes [ADR-0009](adr-0009-human-identifiers.md).

## Context

ADR-0009 designed discovery by human identifiers and included tax IDs (Brazil's CPF, an individual's CNPJ, the US SSN and equivalents) among the bindable attributes, with an accredited verifier's attestation and blind delivery. The justification was good: it is the identifier the company actually has when it issues a charge, it is unique per person and per country, and it would unlock government use.

Michel decided to remove it. The decision is right, and for a stronger reason than the one discussed in ADR-0009.

There, the problem addressed was **enumeration**, and blind delivery already solved that: since no lookup returns an address, sweeping the tax ID space produces nothing but requests thrown into the void. By that measure, a tax ID was as safe as a phone number.

The real problem is another one, and it is different in nature: **a civil identifier is permanent, immutable, unique per country and is the key by which the State sees a person**. Binding the e-work identity to it reintroduces exactly the single global identifier that ADR-0008 exists to eliminate. All the work of an address per relationship, unlinkability and cheap rotation serves to stop anyone from cross-referencing databases and assembling a portrait of a person. A binding to a tax ID, even if only inside a directory, recreates that cross-reference point and puts it in the most sensitive place possible.

And there is the asymmetry that cannot be corrected: a phone number and an email address are disposable, rotatable and chosen by the person. A tax ID is not. If the binding is compromised, you change your phone number; a tax ID accompanies the person until death. A mechanism whose incident mitigation is "change the identifier" cannot rest on an identifier that cannot be changed.

## Decision

1. **Civil identifiers are out of the scope of EWP as discovery attributes.** No `taxid.*`, no `gov.*`. The exclusion also covers the non-fiscal civil identifiers ADR-0009 envisaged (voter registration, equivalent public registers), because they share exactly the same property.
2. **What remains** are the attributes chosen and replaceable by the person: phone (E.164) and email, plus the handle `user@server`, which was never an attribute but an address.
3. **Everything else from ADR-0009 still holds:** blind delivery, no reverse lookup, no bulk export, discovery off by default, a visible record of attempts, and the directory leaving the path after first contact.
4. **An organisation's tax ID still exists**, and that is not a contradiction: it identifies a legal entity in a public register, it appears on the bill, and it is the anti-fraud check on the payee (EW-RFC 0008 §2). A company has a street address; a person does not have to.
5. A jurisdiction that wants compulsory delivery by civil identifier will have to do so in a national profile of its own, outside the core, publicly assuming the cost. The protocol does not open the door and then pretend it did not.

## Alternatives considered

- **Keeping tax IDs with strong safeguards** (accredited attestation, a restricted `allow`, never `anyone`): rejected. A safeguard protects against misuse, not against the nature of the identifier. The permanent binding still exists, and pressure to relax the safeguards would come from those with more power than the user.
- **Tax IDs only for public bodies**: rejected. It is the same door with a doorman, and it turns government into a privileged actor in the protocol, contradicting the principle that an issuer is an issuer.
- **Tax IDs encrypted or derived per directory** (a sectoral identifier, in the spirit of some European schemes): rejected for now. It reduces linkability between directories, but keeps the immutable anchor and adds cryptographic complexity that is hard to audit. It is recorded as a possible path if the use case comes back with force.

## Consequences

- **We lose** remote first contact in the scenario where the company only has the tax ID. In practice, whoever issues a bill almost always also has a phone number or an email address, so the loss is smaller than it looks, and the in-person path (a QR code at signup) remains the strongest of all.
- **We lose** the direct government use case. That is an assumed cost, and EW-RFC 0012 now says so to your face, instead of leaving the expectation hanging.
- **We gain** a property that can be stated without an asterisk: no part of e-work indexes people by civil identifier. That is auditable, easy to explain to non-technical people, and a real difference from practically everything that exists in Brazil.
- **We gain** simplicity: no accredited verifier, no assurance levels for tax IDs, no national validation profiles, and none of the discussion about a legal basis under the Brazilian data protection law for a civil binding.
