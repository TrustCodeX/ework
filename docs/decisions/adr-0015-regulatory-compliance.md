# ADR-0015: Regulatory compliance by design, with profiles over native mechanisms

**Status:** accepted (2026-08-04)

## Context

The origin requirement sets out: the protocol has to support Brazil's LGPD, HIPAA, the GDPR and other regimes. The practical reason is twofold. The issuers who give e-work its value are regulated (banks, clinics, government), and without a clear answer their legal department vetoes adoption before any technical evaluation. And the data subject's rights, which today are exercised through a customer service form, can become buttons: the protocol already has consent as an object, a complete export, symmetric revocation and a signed trail.

There are three real tensions that cannot be wished away: the right to erasure against the immutable history of ADR-0014; the issuer's mandatory legal retention against immediate revocation; and the fact that compliance is a property of a deployment (who operates what, where, under which contracts), never of a protocol.

## Decision

1. **Rights become native mechanisms, not a layer on top.** Access and portability are the export; rectification is a new entry with a visible correction; objection is revocation plus silent retirement; review of an automated decision is the actor class with human approval by key (EWP implements "human in the loop" literally); the Consent object is itself the processing record of the relationship.
2. **Erasure by cryptographic destruction at container level.** Never selective editing of the middle of the history: the task, the relationship or the whole account is erased by destroying the context's keys. Auditing and erasure coexist: either the context exists intact, or it ceases to exist entirely.
3. **A normative account deletion flow**, with an export offered, revocations issued, a grace window and a final response indistinguishable from an account that never existed.
4. **Registrable regulatory profiles** (`br-lgpd@1`, `eu-gdpr@1`, `us-hipaa@1`), declarable in the consent, naming legal retention periods on the issuer's side and extra requirements (such as a business associate agreement in the HIPAA case).
5. **The specification never promises compliance; it promises capabilities.** No official material may say "compliant because it uses EWP".
6. **Specialist legal review before the public opening** is an exit criterion of phase 4, and the language about cryptographic destruction is the first item on the agenda.

## Alternatives considered

- **Ignoring regulation in the specification:** rejected. It kills adoption by the issuers that matter and pushes every implementation to invent its own answer.
- **Treating it as a product layer:** rejected in part. Organisational policies really are a matter for the deployment, but rights exercised between federated parties (revoking, erasing, exporting, contesting an automated decision) require a protocol mechanism in order to be exercisable at all.
- **Designing for one law only (the LGPD):** rejected. Designing by profiles covers the analogous regimes without rewriting the core.
- **Relaxing the immutability of the history to satisfy the right to erasure:** rejected. It would destroy the auditing property that immutability exists for; container-level erasure satisfies the right without opening the door to selective editing.

## Consequences

- EW-RFC 0016 is born with the rights-to-mechanisms map, the deletion flow, retention by profile and the position on cryptographic destruction, with the legal debate honestly noted.
- The immutable history becomes an adoption argument in healthcare: an audit trail is a requirement of HIPAA's technical safeguards.
- Gateways and directories, which concentrate PII outside E2EE, gain an obligation to publish a minimal retention policy.
- A real cost: account deletion, incident notice and cryptographic destruction enter the implementation sequence, and the legal review becomes a dependency of the opening schedule.
