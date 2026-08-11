---
docname: draft-wilhelm-ework-identity-00
title: "The e-work Protocol (EWP): Identity, Addressing, and Rotation"
abbrev: EWP Identity
category: exp
area: Applications and Real-Time
author: Michel Wilhelm
email: michelwilhelm@gmail.com
normative: RFC2119,RFC8174,RFC8032,RFC8259,RFC8615
---

This document specifies identity and addressing for the e-work Protocol (EWP).
An EWP identity is a key pair under the sole control of its holder; a host never
possesses the private key. Human-readable handles are verifiable aliases, never
the identity itself.

The document also specifies per-relationship addressing, in which a recipient
issues a distinct opaque address to each counterparty, and the rotation protocol
that allows an identity to replace addresses while choosing, relationship by
relationship, who is carried forward and who is silently left behind.

<!-- abstract -->

# Introduction

Systems that identify people by an address controlled by a provider make
switching providers equivalent to changing identity. Systems that identify
people by a key held by the person avoid that, at the cost of key management.
EWP takes the second position and spends the rest of this document on making the
cost bearable.

## Conventions and Definitions

<!-- rfc2119 -->

# Key Layers

| Layer | Count | Storage | Purpose | Rotation |
|---|---|---|---|---|
| Root | 1 per identity | Offline, derived from the recovery kit | Being the identity; signing the other keys | Rare, with a contestation window |
| Device | 1 per device | On the device | Day-to-day signing | On loss or compromise |
| Contact | 1 or more per relationship | On the device | Deriving relationship addresses | Freely |

The root key MUST NOT be used to encrypt content or to authenticate sessions:
it signs and steps away. Day-to-day operations belong to the device keys, and
the outward face is the contact key.

**The compromised key never authorises its own replacement.** If it sufficed,
whoever stole it would rotate first. Root replacement is authorised by the
recovery key, device replacement by the root or another verified device.

## Deriving keys from the recovery kit

The kit is a twelve-word BIP-39 mnemonic encoding 128 bits of entropy. Two
conforming implementations MUST derive the same keys from the same twelve words,
otherwise the kit recovers an account in one client and fails in another, which in
a protocol with no second copy is permanent loss of identity. The derivation MUST
be:

1. `seed = PBKDF2-HMAC-SHA512(password = NFKD-normalised phrase,
   salt = "ework-mnemonic", iterations = 2048, output = 64 bytes)`. This is the
   BIP-39 construction with its own salt: the `ework` prefix guarantees that the
   same twelve words yield different keys here and in a cryptocurrency wallet,
   without which reusing a phrase across both would tie one compromise to the
   other.
2. The optional BIP-39 passphrase MUST NOT be used, and implementations MUST NOT
   offer it.
3. `root = Ed25519(HKDF-SHA256(ikm = seed, salt = empty,
   info = "ework/v1/root", output = 32 bytes))`.
4. `backup = HKDF-SHA256(ikm = seed, salt = empty, info = "ework/v1/backup",
   output = 32 bytes)`.
5. Threshold splitting, where offered, operates on the 128 bits of entropy, never
   on `seed` or on a derived key.

The distinct `info` labels are domain separation, not decoration: deriving both
keys from the same material without separating them would make the leak of one the
compromise of the other. This document MUST publish test vectors, which is the
only way two implementations can prove they agree rather than discover the
divergence on the day someone needs to recover.

# Identity Document

The identity document is public and self-signed.

~~~
{
  "id": "did:key:z6MkCleia...",
  "seq": 12,
  "handles": ["cleia@ework.example"],
  "hosts": [ { "handle": "cleia@ework.example",
               "clientApi": "https://ework.example/ework/v1" } ],
  "devices": [ { "id": "dev-a1b2", "key": "ed25519:...", "actorClass": "human",
                 "validFrom": "2026-08-04T00:00:00Z", "validUntil": null } ],
  "recoveryKeys": ["ed25519:..."],
  "previousKeys": [],
  "updatedAt": "2026-08-04T00:00:00Z",
  "proof": { "type": "ed25519-jcs-2026", "by": "did:key:z6MkCleia...", "sig": "..." }
}
~~~

**Contact keys MUST NOT appear in this document.** Publishing them would link
every address a person holds to every other and destroy exactly the property
that justifies them. They are known to three parties and nobody else: the
user's client, the host that routes that address, and the counterparty of that
relationship.

Receivers MUST accept only documents whose `seq` exceeds the last one seen and
whose signature verifies against the root key, or against a valid rotation as
specified below.

`actorClass` on each device is load-bearing and specified in the autonomous
actors document: it is what allows a task to require human approval and have
that requirement be verifiable rather than advisory.

**The `id` of each device MUST be unique within the document, and receivers
MUST refuse the whole document when two repeat.** The id is the key by which a
session declares which device is signing, and a document carrying the same id
twice has no right answer: any lookup returns the first entry, and the second
entry's signature ends up checked against the first entry's key. The symptom is
a rejected signature in a document where nothing looks wrong, and the signed
proof does not help the diagnosis, because the proof is correct and it is the
document that is ambiguous. Refusing on input turns a client with a weak id
generator into an immediate, legible error, instead of an account that starts
out working and stops opening on the second device. Implementations MUST NOT
derive the id from a clock alone: two devices added in the same millisecond
collide, and creating the account and then signing in again right away is the
common case.

## Freshness and rollback

A malicious host can serve an old document with a revoked device still listed,
and a peer that never saw the newer `seq` would accept it. Because the root
lives offline, the defence cannot be frequent re-signing by it. Instead:

1. Any current device SHOULD periodically publish a **freshness proof**
   `{ id, seq, at }` signed by the device key (SHOULD: every 7 days). A
   document whose most recent proof is older than 30 days MUST be treated as
   suspected rollback: clients warn and try other sources before trusting it.
2. Clients MUST **gossip the most recent `(id, seq)` pair** they know inside
   the MLS groups they belong to. Whoever learns a higher `seq` through gossip
   MUST refuse documents with a lower `seq` from then on, and treat the host
   that served them as suspect.

An honest limitation: whoever shares no group with the identity can be deceived
until the freshness proof expires. Key transparency, a public log of documents
in the spirit of WhatsApp's deployment, is the recorded evolution to close that
remainder; it is not a requirement of this version.

# Organisations

An organisation's identity is anchored in its domain: `GET
https://<domain>/.well-known/ework/org.json` publishes the identity document,
in the spirit of `did:web`.

Issuer verification is **open**: controlling the domain and publishing the key
is enough to exist. Reputation and extra attestations are pluggable extensions,
never a prerequisite.

Organisations do NOT use contact addresses to receive: their address is public
and stable by nature. The asymmetry is deliberate, and it is the one the
physical world already has: the company has a street address, you need not give
yours.

Clients MUST display the issuer's verified domain on every offer, and MUST
highlight issuers seen for the first time.

# Devices

Each device generates its own key pair, and the root signs its inclusion,
either from the recovery kit or from a device already authorised.

**Device-to-device verification** (a QR code or a short number comparison)
establishes trust with no server involved.

**Revocation** removes the device from the document (`seq` + 1) and from every
MLS group (a new epoch). Hosts MUST refuse sessions from a revoked device.

The device set is the same for every address: the person is one, it is the
faces that are many.

**Not every device is operated by a person.** Each key declares its
`actorClass` (`human`, `assisted`, `autonomous`, `system`), signed by the root.
Only a `human` key produces human confirmation, and that is what makes
human-in-the-loop a property of the system rather than a request to somebody
else's client. Agents get their own key with a delegated scope, never the root
and never a human key.

# Handles and Addresses

Two address types exist, with different purposes.

**Primary handle** (`cleia@ework.example`): public, stable, how people find a
person. Used for invitations between parties who know each other.

**Relationship address** (`k7f3a2q9@ework.example`): opaque, one per
counterparty, what organizations know.

## Bidirectional handle proof

A handle binds a name to a key, and the binding MUST be verifiable from both
directions.

1. **Domain to key:** `GET https://<domain>/.well-known/ework/handles/<local>`
   returns `{ "id": "did:key:z6MkCleia...", "seq": 12 }`.
2. **Key to domain:** the identity document lists the handle in `handles`.

A verifier MUST check both. Checking only the first lets a domain claim someone
else's key; checking only the second lets a key claim someone else's domain. A
handle is verified only when the two proofs match, and clients MUST treat an
unverified handle as a strong warning. Changing a handle does not change the
identity: contacts follow the key, not the name.

**The handle proof is not a lookup desk.** The first endpoint MUST require that
the querier already knows the target: `GET .../handles/<local>?id=<did:key>`
returns `204` when the pair matches and `unknown-recipient` when it does not,
always with the same latency, and MUST NOT return the `did:key` to a querier who
presented only a name. Hosts MUST rate limit this endpoint per network origin.

Returning the key to whoever knows the name made the endpoint a public oracle of
account existence, open to anyone, uncapped and unauthenticated. Each hit yielded
the `did:key`, which is the lookup key for the identity document, and that
document published complete `handles` and `hosts`: finding one handle yielded all
the others and every host, undoing the context separation this document promises.

## Keys are intervals, not presence

Each entry in `devices`, and each root, MUST carry `validFrom` and `validUntil`
(`null` while current). Revoking a device means filling in `validUntil`, and
implementations MUST NOT remove it from the list. Hosts MUST serve any earlier
version of the document when requested by `seq`, and signed entries MUST carry
the `seq` of the document they were signed under.

Verifiers MUST accept a signature when the key that produced it was valid at the
instant the act was recorded, and MUST reject it when that instant falls outside
the interval, even if the key still appears in the current document.

A presence list alone erred in both directions, and both were exploitable.
Repudiation was cheap: revoking the device that signed an act removed the key, and
a verifier holding only the current document could not tell "legitimate key at the
time" from "key that never existed", erasing verifiability without erasing the
entry. In the other direction, a verifier with a stale document kept accepting an
already revoked key, because nothing in the object said until when it was valid.

**Publication is minimal.** `handles` and `hosts` MUST list only the handle and
host through which the document is being served. Other bindings MUST be delivered
signed inside the MLS groups the person already belongs to, and MUST NOT be
published.

The handle proof lives at the **address domain**, not at the service domain.
This is deliberate: it is the one lookup that must work before anything is known
about where the service runs.

# Per-Relationship Addressing

**Default rule:** on granting consent to an organization, the client MUST
generate a new address for that relationship, unless the user explicitly chooses
to reuse an existing one.

~~~
{
  "@type": "ContactKey",
  "alias": "k7f3a2q9@ework.example",
  "key": "ed25519:...",
  "host": "ework.example",
  "peer": "utility.example",
  "consent": "ework:consent/019f3c00-...",
  "createdAt": "2026-08-04T11:58:00Z",
  "state": "active",
  "acceptsNewRequests": false,
  "boundBy": { "sig": "signed by the root" }
}
~~~

Four properties follow, and they are the reason the mechanism exists.

**Leak attribution.** Each issuer knows a different address. Spam arriving at
`k7f3a2q9` proves who leaked or sold the data. Clients MUST show where an
address came from, because the property is worthless if the user cannot connect
address to relationship.

**Surgical severance.** Retiring one address ends one relationship without
touching the others.

**A scraped address delivers nothing.** Delivery requires a valid capability for
that address. Without one, the most an attacker achieves is a
`consent.request` in quarantine, and with `acceptsNewRequests: false`, the
default, not even that.

**Non-linkability.** Two issuers cannot, by default, discover that they are
talking to the same person. This is what undermines cross-referencing by shared
identifier.

Registration is private: the host keeps the address list in private storage,
never in a public document. A host learns the addresses it routes and only
those; distributing addresses across hosts partitions that knowledge as well.

People invited personally use the primary handle by default, because there
linkability is desired: you want your sister to know it is you.

# Rotation

## Authority

| Rotating | Authorised by | Immediate effect |
|---|---|---|
| Contact key | Root or verified device | Old address dies; relationships migrate by choice |
| Device key | Root or another verified device | Removal from all encryption groups |
| Root key | Recovery key, with contestation window | Continuity chain in `previousKeys` |

## Rotation declaration

~~~
{
  "@type": "KeyRotation",
  "subject": "k7f3a2q9@ework.example",
  "oldKey": "ed25519:...",
  "newKey": "ed25519:...",
  "newAlias": "k9b1c4x2@ework.example",
  "reason": "compromise",
  "at": "2026-09-01T10:00:00Z",
  "graceUntil": null,
  "proofs": [ { "by": "old key or root" }, { "by": "did:key:z6MkCleia..." } ]
}
~~~

`reason` is one of `compromise`, `hygiene`, `migration`. With `compromise`,
`graceUntil` MUST be null: the old key dies immediately, because keeping the
address alive for convenience keeps alive the access of whoever stole it. With
`hygiene` or `migration`, the host SHOULD accept both addresses for up to 30
days so that envelopes in flight are not lost.

The continuity proof is what prevents hijacking: without a signature from the
old key or the root, nobody redirects another party's address.

## Selective migration

The declaration goes to **always** the identity's own hosts, and **only the
counterparties the user chooses**.

Those who do not receive it are left with a dead address. There is no
negotiation, no unsubscribe request, and no link confirming that the recipient
exists.

This is the mechanism that gives the recipient leverage. In one operation, an
identity can replace every address it holds and carry forward only the
relationships it wants, leaving the rest with identifiers that resolve to
nothing. Clients MUST make the set of those left behind explicit before
executing, by name, because the operation is not reversible from the recipient's
side: restoring a relationship requires the counterparty to knock again.

## No oracle

A retired address enters state `retired`. The host MUST respond to envelopes
addressed to it exactly as it would for an address that never existed:
`unknown-recipient`, a permanent error. The host MUST NOT distinguish, in its
response, between retired, nonexistent, revoked, blocked, frozen for deletion,
**or a live address whose sender holds no relationship with it**.

That last case was missing and was the commonest of all. While a live address
without a relationship answered `no-consent` and a dead one answered
`unknown-recipient`, one request per guess separated them, producing the verified
list of live addresses that is precisely the product the spam industry buys. The
`no-consent` code MAY be used only where the sender already holds a capability
issued for that address, meaning it already knew the relationship existed.

Equality MUST hold in code, body and timing, and implementations MUST test
latency equality rather than status codes alone.

The rule applies to **every public surface**, not only to delivery. In
particular, the handle proof of this document MUST return `unknown-recipient`
for a retired, frozen, or destroyed address. A host that refuses delivery while
continuing to publish the identity document delivers the same information by a
different door, and anyone wishing to validate a list compares the two responses
instead of one. Absence of an oracle is a property of the set of endpoints,
never of one in isolation.

## Root rotation

A new document signed by the old key or by a `recoveryKey`, with `previousKeys`
recording the chain, and a contestation window (SHOULD: 30 days) during which
the old key can dispute a fraudulent rotation.

Because the layers are independent, **rotating the root does not force a single
address to change**: the bindings are re-signed, and the relationships
continue. This is deliberate: replacing the identity cannot cost notifying
every company in the person's life.

# Recovery

The recovery kit is generated at identity creation, without exception, and
derives the root key and the backup key. The interface MUST force the user to
save it before releasing the account for use.

**The kit IS a twelve-word BIP-39 mnemonic**, encoding 128 bits of entropy. The
entropy derives the root and backup keys; the kit MUST NOT be the root key in
the clear.

**The derivation is normative and complete.** Two conforming implementations
must derive exactly the same keys from the same twelve words, or the kit
recovers the account in one client and fails in another, which in a protocol
with no second copy is the permanent loss of an identity.

BIP-39's optional passphrase MUST NOT be used, and implementations MUST NOT
offer it: it is one more secret to lose, with no clear owner, and recovery is
already the hardest moment the person will have.

The distinct `info` labels are domain separation, not decoration: deriving both
keys from the same material without separating them would make the leak of one
the compromise of the other. This specification MUST publish test vectors,
because that is the only way two implementations can prove they agree instead
of discovering the divergence on the day somebody needs to recover an account.

**Test vectors.** Two implementations only prove they agree against numbers
neither produced alone. The phrase is BIP-39's canonical one, the 128 bits all
zero, so that the entropy step can be checked against the original
specification before reaching the part that belongs to this document.

~~~
phrase: abandon abandon abandon abandon abandon abandon abandon abandon
        abandon abandon abandon about
seed:   1ac64832a93def83...          (64 bytes, hex, first eight)
root:   LI3K91PIbMAkwI93GynLXxRUuzUCmcP2zGCoO7Tr3c4=   (base64, 32 bytes)
backup: hmlZJ4yjv6ECSzem2Ru0rgtWgkUcDsBqHF3Nh8MvHZE=
did:    did:key:z6MkgkJbJuF93YL7dpChw5ZKPA8aobXSmeyPnxffy5m6BERo

phrase: legal winner thank year wave sausage worth useful legal winner
        thank yellow
root:   hsRws2q/kklMBBq28bgLH9jrhjU0Eh6Pa3V87YPq07k=
backup: Cj6D+CD4XQU4S5n88w+s9X8YbbxALe59kNELqZR6Q/I=
did:    did:key:z6MkgSrVEjLTohWt7as6x4mhbXhNcjaFjnmC4LLhg2dt4cta
~~~

An implementation that does not reproduce these values is NOT conforming, and
the consequence of discovering that late is a person holding the kit in front
of an account that will not open.

Implementations MUST generate in the language of the interface, and MUST detect
the language on input from the words themselves, never by asking.
Implementations MUST keep accepting kits in earlier formats on input: what
changes is what is generated, never what is accepted. The BIP-39 checksum MUST
be verified locally before any call to the host, so that a wrong word is
reported as a wrong word instead of becoming a signature failure later.

**Kits that predate this derivation.** The reference implementation used to
derive with the salt `ework` and split the 64 bytes in half, with no HKDF, and
identities created that way exist. Because the root IS the identity, applying
this section's derivation to them migrates no key: it produces a different
person. Implementations MUST accept the earlier derivation on input, choosing
between the two by the `did:key` published in the account's document, and MUST
generate only with this section's derivation. Migrating from one to the other
is a **root rotation** as specified above, which preserves the handle and the
relationships and is an act of the owner, never automatic.

**No second factor at the host.** The host MUST NOT store or require an
authentication factor beyond the signature. The second proof, where an
implementation wants one, is a signature from another already-authorised
device. Hosts MUST NOT lock an identity after failed attempts: the rate limit
is per network origin, and locking an identity would hand anyone who knows an
address the ability to deny service to its owner.

The kit alone recovers the identity, not the relationships. The **address
registry** is part of the encrypted backup: without it the person exists again
but loses the map of who knows which address. Implementations MUST include the
registry in the backup.

**Total loss without the kit:** the identity is unrecoverable by design. A new
identity is created and the relationships are rebuilt. Optional custody, as
specified in the cryptography document, changes the trade-off.

**Emergency access and inheritance, a sketched extension.** The kit MAY be
split among guardians by threshold (Shamir): reconstruction requires K of N
shares and opens a waiting period (SHOULD: 30 days) with a notice on every
device, cancellable by the owner with one tap on any verified device.
Inheritance uses the same mechanism with a longer wait, and the legal effects
are handled in the compliance document. The kit format is born compatible with
splitting; the full ceremony is left to an extension document. The use case is
the target audience in its purest form: a person's bills keep coming due when
she dies or becomes incapacitated.

# Multiple Hosts

An identity MAY register at more than one host. The identity document lists all
of them, and delivery to a handle at any of them reaches the same identity.

The property this buys is that provider migration does not change identity: the
key stays, the document is updated, and correspondents follow the document
rather than the address.

Two further consequences are the point of the design. **Knowledge
partitioning:** the bank's addresses on the home host, the shops' addresses at
a public provider. No host sees the whole set. **Context separation:** spaces
(personal, family, work) can live on different hosts, with different policies.

**Migration** consists of creating a mailbox at the new host, importing the
export, and publishing a document with `seq` + 1. The old host SHOULD serve a
redirect for at least 30 days and forward what arrives. Relationship addresses
migrate along, with a rotation declaration carrying `reason: "migration"` sent
to the counterparties the user keeps. Consents remain valid because they point
at the identity, not at the host.

# Security Considerations

Keeping the root key out of the hot path limits the damage from a compromised
device. The requirement that replacement be authorised by a superior key, never
by the suspect key itself, prevents an attacker from rotating first. The
contestation window limits malicious rotation via a stolen recovery key.

The bidirectional handle proof prevents both a domain claiming another party's
key and a key claiming another party's domain.

The absence of an oracle is a security decision, not a usability one. Any
differentiation in response would be turned into a list-validation tool.

# Privacy Considerations

Per-relationship addressing is the principal privacy mechanism of the protocol
and its cost is real: the user holds more identifiers than a single handle, and
clients must present them in a way that keeps the mapping legible. A client that
displays an opaque address without saying which relationship it belongs to has
removed the benefit while keeping the cost.

The public identity document carries the operational minimum, but the device
list still exposes cardinality; opaque device ids are permitted for that
reason.

# Open Questions

1. **Cognitive cost:** the interface can never speak of keys. The product
   vocabulary for the relationship address is not yet decided (candidates: "an
   address only for this company", "direct line", "alias"). It needs testing
   with real users.
2. **Relationship addresses for people:** should the default change in some
   cases (a marketplace seller, an occasional service provider)?
3. **Emergency access and inheritance:** an extension document for the guardian
   ceremony, guardian replacement and revocation, and the interaction with
   account deletion.
4. **Minors and managed accounts** (the elderly mother of the reference
   scenarios): managed delegation needs a design of its own, and interacts with
   who may rotate.
5. **A neutral identity directory** (the lesson of AT Protocol's PLC
   association): when, and under what governance.

# IANA Considerations

This document requests registration of the well-known URI suffix
`ework/handles` in the "Well-Known URIs" registry {{RFC8615}}, with change
controller IETF and this document as reference.
