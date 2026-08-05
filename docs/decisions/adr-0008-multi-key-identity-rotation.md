# ADR-0008: Multi-key identity with an address per relationship, and rotation as an anti-abuse tool

**Status:** accepted (2026-08-04, decided with Michel)

## Context

ADR-0003 fixed the root of the identity in a key belonging to the user, with a readable handle as a verifiable alias. What remained unanswered was what happens in adversarial daily life: an address leaks, is sold or lands in a data dump, and starts receiving spam, marketing and attempted fraud.

In email, that scenario is terminal: there is only one address, once leaked it is leaked forever, you never find out who leaked it, and the only way out is to change address and tell the entire world. It was the asymmetry that killed email as a trustworthy channel.

Michel put the requirement like this: connect the account to N servers, be able to hold N keys in the profile, and be able to recreate the key when somebody obtains it and starts sending spam, telling the servers you were part of which key was the old one and which is the new one, so as to stop receiving things from one specific place.

The observation embedded in that is stronger than "key rotation": if the change has to stop one specific place without affecting the rest, then what rotates cannot be the whole identity. There has to be one key per relationship.

## Decision

1. **Three layers of key, with distinct roles.** A root key (offline, the identity, signing the rest), device keys (session and group cryptography, revocable one by one) and **contact keys** (one per relationship, each with an address of its own).
2. **An address per relationship is the default for organisations.** Each issuer receives a distinct address, derived from a contact key of its own. For people invited personally, the default is the primary handle.
3. **Contact keys do not go into the public identity document.** Listing them publicly would link them all together and destroy the very property that justifies them. They are known only to the user's client, to the host that routes that address, and to the counterparty.
4. **Rotation with selective migration.** On rotating, the user chooses who receives the continuity statement (old key to new key, signed). Whoever does not receive it is left behind, with no negotiation and no notice.
5. **Silent retirement is not an oracle.** A retired address answers exactly like an address that never existed (`unknown-recipient`). The sender does not learn whether they were cut off, mistyped the address or the account vanished.
6. **The send capability is bound to the address.** An address scraped from a leak delivers nothing: without a valid credential for that address, the most the world reaches is a consent request in quarantine, and even that the user can switch off per address.
7. **Rotating the root does not force changing addresses.** The layers are independent: replacing the root updates bindings, it does not oblige anyone to notify every issuer.

## Alternatives considered

- **A single key, rotated whole** (the initial mental model): rejected. Rotating cuts everyone off together, forces re-notifying every legitimate relationship and turns an incident with one issuer into a global event in the user's life. It also makes it impossible to discover who leaked.
- **Subaddresses in the style of `name+shop@domain`**: rejected as the main mechanism. It is trivially strippable by any script (`+shop` comes off with one regex), it has no key of its own, and it does not survive anyone who wants to work around it.
- **Disposable addresses with no link to the identity** (anonymous email relays): rejected. It loses continuity: with no proof that the new address is the same person, a legitimate relationship cannot be migrated without re-signing everything.
- **Reputation and statistical filtering as the primary defence**: rejected as primary. That is email's route, which ended in opaque reputation controlled by two providers. It remains as an optional pluggable layer (EW-RFC 0011).
- **Leaving the mechanism to the product layer**: rejected. If it is not in the protocol, every client invents its own way and the cutting-off property is lost at the first bridge.

## Consequences

- **Leak attribution becomes a feature.** Spam arriving at an address given to a single issuer proves who leaked or sold it, and the client shows that.
- **Cutting off is local and instant.** There is no asking to be removed, and no unsubscribe link that confirms you are alive.
- **A new UX cost.** An ordinary person cannot be shown "twenty-seven addresses". The interface speaks of relationships ("Acme Energia"), never of keys, and rotation appears as "stop receiving from this company" or "change my address with them".
- **Legitimate issuers have to implement continuity.** Honouring a rotation statement becomes a conformance requirement and goes into the reference issuer gateway.
- **A real implementation cost.** The client now manages N keys, the host now keeps a private registry of addresses per box, and the MLS group model presents the contact key as the credential in the relationship, not the root.
- **Recovery gets heavier.** The recovery kit has to restore the address registry too, otherwise the person recovers their identity and loses their relationships. Handled in EW-RFC 0003 §8.
