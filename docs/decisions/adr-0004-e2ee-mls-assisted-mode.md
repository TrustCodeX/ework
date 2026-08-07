# ADR-0004: E2EE by default with MLS; assisted mode as an explicit opt-in

**Status:** accepted (2026-08-04)

## Context
The originating requirement: "everything must be end-to-end encrypted", with the precise mental model The origin requirement describes: in a project of three people, each has their own key, the packet sent to everyone does not open for anyone outside, and adding someone starts letting that someone open the packets. That is literally group cryptography, and a ready-made IETF standard exists: MLS (RFC 9420), with real adoption in 2026 (RCS and the GSMA, Wire, Webex). The known cost of E2EE: a blind server cannot search, cannot run hosted automation and cannot escalate on its own.

## Decision
1. **E2EE is the default** for all content: personal boxes, user-issuer relationships and projects.
2. The primitive is the **MLS group**: (a) a personal group with all the identity's devices; (b) a relationship group (user plus issuer); (c) a project group (all the devices of all the members). Only members open the packets; Add and Remove change the epoch and the keys.
3. **History for new members is an explicit re-share** by an existing member (configurable per project), because MLS only grants access from the point of entry onwards.
4. Multi-device copies Matrix's tested package: **cross-signing, device verification and a key backup encrypted with a recovery key**.
5. **Assisted mode per collection, always opt-in:** the user may hand over the keys of one collection to the host in order to gain server-side search, hosted rules and escalation with the devices switched off. The UI must spell out the consequence; a silent downgrade is forbidden.

## Alternatives considered
- **Strict E2EE with no assisted mode:** rejected. It kills features the scenarios demand (escalation with a dead device, a simple issuer gateway) and would push implementations into unspecified workarounds.
- **TLS plus at-rest now, E2EE later:** rejected. It contradicts the requirement, and retrofitting E2EE hurts (Matrix can attest to that).
- **Megolm (Matrix) instead of MLS:** rejected. MLS is the IETF standard with momentum (RCS), it has better post-compromise security and mature libraries (OpenMLS).
- **Simple peer-to-peer encryption (PGP-style, per recipient):** rejected as a general primitive. It does not scale to groups with member churn and multiple devices; it remains as the degenerate case of a two-member group.

## Consequences
- Ordering and state merging under E2EE are our problem (MLS does not solve them): RFC 0004 defines an encrypted operation log with rebasing.
- Timers for critical tasks run on the clients by default; server-side escalation requires assisted mode (RFC 0009 documents the trade-off).
- Losing every device without the recovery kit means the content is lost; being honest about that is part of the product.
