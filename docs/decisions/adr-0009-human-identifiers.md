# ADR-0009: Human identifiers as verified attributes, with blind delivery and no reverse lookup

**Status:** superseded (2026-08-04) by [ADR-0010](adr-0010-no-civil-identifier.md), which keeps the mechanism and removes civil identifiers (tax IDs) from scope. The text below is preserved because it contains the analysis that led to the exclusion, and because the idea of indexing people by tax ID will be proposed again.

## Context

ADR-0008 solved how an existing relationship is protected: an address per relationship, rotation, a silent cut-off. A hole was left at the start of the story. Acme Energia has Cléia's tax ID, her phone number and perhaps her email. It does not have, and could not have, her e-work address. Without a bridge, initial consent can only be born in person (a QR code at the counter, a link on the printed invoice), which limits adoption exactly where it matters most.

The origin proposal was to accept the identifiers the world already uses: a phone number with country and area code, a tax ID (CPF, SSN, whatever each country has, unique per person and therefore excellent for government use), an email address, and `user@server`, noting that this last form also serves to hide the identity key behind a managed server.

The risk is proportional to the usefulness. A directory that answers "what is the address of whoever holds tax ID X" is an enumeration oracle over a space of roughly a billion numbers with a check digit, in a country where that database has already leaked several times. Built naively, it hands over a national map of who has an account, and turns the anti-fraud protocol into fraud infrastructure. Signal took years and a hardware enclave to mitigate the same problem with phone numbers; Matrix's identity servers tried to solve it with hashing and did not, because a small space is enumerable even when hashed.

## Decision

1. **A human identifier is an attribute, not an identity.** The identity remains the root key. Phone, email and tax ID are attributes that **bind** to it, with a signed attestation, and can be revoked.
2. **`user@server` remains the handle**, which has existed since ADR-0003 and fulfils exactly the role The origin requirement describes: a readable form that hides the key and allows using a managed server.
3. **The directory is a doorbell, not a catalogue.** It never translates an attribute into an address. It accepts a consent request addressed to an attribute and forwards it if it knows the binding. Whoever queries receives no address at all.
4. **The answer is always the same**, whether or not a binding exists. No existence oracle, consistent with EW-RFC 0003 §7.4.
5. **Discovery is born switched off.** Nothing is discoverable until the user switches it on, per attribute and per requester class.
6. **No reverse lookup, no export, no list checking.** There is no address-for-attribute, and no "which of these ten thousand tax IDs have an account".
7. **A tax ID requires an attestation from an accredited verifier.** Possession of the number is never proof of anything, because in Brazil the number is public in practice.
8. **After first contact, the directory leaves the path.** The relationship starts using the direct contact address, forever.

## Alternatives considered

- **A classic lookup directory** (attribute to address, like a DNS for people): rejected. It builds a queryable census of who has an account, enumerable in days, and a gift to the fraud industry.
- **A hash of the attribute as the search key** (the route of Matrix's identity servers): rejected. The phone and tax ID spaces are far too small; hashing does not protect an enumerable domain, and it gives a false sense of security.
- **The phone as the identity** (the Signal and WhatsApp route): rejected. It ties the identity to a resource the carrier controls, dies on number portability and SIM cloning, and excludes anyone who does not want to give a phone number.
- **A secure enclave for private lookup** (Signal's current route): rejected for now. It solves a problem that blind delivery dissolves without special hardware, and it would create a dependency on an enclave vendor.
- **No bridge at all, only in-person QR codes**: rejected. Pure and useless: it kills the use case that justifies the protocol.

## Consequences

- **Remote first contact works** without exposing an address, which unlocks issuers at scale.
- **Government use becomes possible and framed:** authenticated official notification, with a receipt, without creating a queryable register of citizens.
- **The directory becomes sensitive infrastructure** with governance of its own: who operates it, who audits it, who answers for a breach. It is the only part of the system that is not indifferent to who hosts it, and that has to be written down.
- **Implementation cost:** accredited verifiers, an attestation format, a quota and a record of attempts per requester.
- **Accepted residual risk:** a compromised directory can test binding guesses and observe who tries to reach whom. Mitigated by splitting (several directories, including national and sectoral ones), by the binding never becoming an address, and by the user seeing every attempt.
