# ADR-0006: Consent before delivery (consent-first) as a rule of the protocol

**Status:** accepted (2026-08-04)

## Context
Every open federated protocol born with "accept everything from everyone" turned into spam: email (a retrofit defence through SPF, DKIM and DMARC twenty years later, with reputation controlled by an oligopoly), Mastodon (the 2024 wave), Matrix (invite spam, policy servers in 2025). At the same time, the two best precedents for our use case are consent-first: Brazil's DDA (enrolling as an elected payer before any bill appears) and Open Finance (consent as a resource with scope, term and symmetric revocation). No federated protocol has this as a rule of the protocol. e-work will.

## Decision
1. **No full delivery without prior consent.** An unknown issuer reaches only a minimal `consent.request`, which lands in a silent quarantine (never raising an audible notification).
2. **Consent is a first-class object** (RFC 0007): id, scope (payload types, maximum rate, maximum urgency), the policy for status returned, validity, auditable states.
3. **Revoking is as simple as granting**, taking effect immediately: the user's host starts refusing at the edge, and the issuer receives a permanent error.
4. The send capability derived from the consent **travels with the user's identity**, not with the provider (unlike DDA, which ties it to the bank).

## Alternatives considered
- **The email model (accept and filter):** rejected. It is the root cause of spam; filtering becomes an arms race and centralises power in the large providers.
- **Economic cost or proof of work (Nostr):** rejected as the primary mechanism. It punishes legitimate users and does not model continuous user-issuer relationships.
- **A central directory of approved issuers (the Gmail Actions and AMP model):** rejected. Gatekeeping by a platform is the anti-pattern we want to kill; verification is open (domain plus key), and reputation is a pluggable layer.

## Consequences
- Issuer onboarding gains a step (the handshake), and the UX design of that step (a QR code in the shop, a link on the printed invoice, acceptance at signup) becomes central to adoption.
- Mass spam becomes structurally expensive: without consent there is no delivery, and consent is per user.
- The quarantine of requests needs careful UX so it does not become "a second spam box" (the rule: requests expire, do not accumulate without limit, and never interrupt).
