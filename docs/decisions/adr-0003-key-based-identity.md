# ADR-0003: Root identity by the user's key, with the readable address as an alias

**Status:** accepted (2026-08-04)

## Context
The originating requirement is explicit: every client can connect to multiple servers with a single identity, and everything is end-to-end encrypted. An identity tied to the server (the email and Matrix model) contradicts both points: migrating servers becomes losing the account, and real E2EE requires the root of trust to belong to the user. The research confirmed it: identity coupled to the homeserver is Matrix's number one public regret, and AT Protocol (a stable DID plus a DNS handle) is the best design in its class.

## Decision
1. The root of the identity is an **Ed25519 key pair belonging to the user** (compatible with did:key in its representation).
2. The **readable address `name@domain` is a verifiable alias**, with proof in both directions (the domain points at the key; the identity document lists the handle), in the style of NIP-05 and atproto.
3. The **identity document** (signed by the root) lists: handles, current hosts, device keys (cross-signing), recovery keys and rotation history.
4. **Organisations** are anchored in a domain (a key published at `/.well-known/ework/org.json`, in the spirit of did:web): issuer verification is open, with no central gatekeeper.
5. A raw key never appears to the end user: the UX speaks of addresses; keys are infrastructure.

## Alternatives considered
- **An address tied to the server:** rejected because of the portability requirement and the Matrix lesson.
- **A full W3C DID:** rejected for now. Too complex, and the ecosystem is immature for the ordinary user. The format is born DID-compatible for future interoperability at no cost today.
- **A raw key as the visible identity (pure Nostr):** rejected. Without rotation and without recovery it is no good for ordinary people.

## Consequences
- Migrating hosts means updating the identity document, not rebuilding a life.
- Account recovery becomes a key custody problem (a mandatory recovery kit, optional custody): handled in RFC 0003 and RFC 0006.
- A neutral identity directory may emerge later (the PLC association lesson); v0.1 lives without one, with discovery by domain.
