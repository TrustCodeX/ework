# ADR-0024: The identity vault is an application layer, and it stays outside the protocol

**Status:** accepted | 2026-08-11 | Michel Wilhelm

## Context

Signing in to e-work means proving possession of the kit. ADR-0018 settled that, and for a good reason: a second factor exists to shore up a password, the protocol has no password, and the kit is already "something you have". EW-RFC 0003 §8 goes further and forbids the optional BIP-39 passphrase with a MUST NOT, because it is one more secret to lose.

The problem is not in the protocol. It is in the distance between the protocol and what an ordinary person already knows how to do. Whoever is going to use this has spent their life signing in to sites with an email address and a password. Asking them to write twelve words on paper **before** they have seen the product work is asking for a key custody ceremony from someone who only wanted to pay a bill. And there is a case the protocol does not cover at all: the same person with more than one identity, one for work and one personal, each with its own twelve words, and nowhere for the two to live together.

A site implementing the protocol has to solve this. And in solving it, the obvious temptation is to put a password on the host, which is exactly what ADR-0018 forbids.

## Decision

**The vault is an application layer above the protocol. The password encrypts and decrypts the twelve words on the client, and the protocol underneath stays what it is.**

1. **The password is never a protocol credential.** It derives a key that encrypts the phrase, and nothing else. What the host receives is still a signature, exactly as EW-RFC 0004 §1 describes. The host does not know the vault exists.

2. **The encrypted vault lives in the application's backend, never in the e-work host.** They are two servers with two jobs: the host routes signed envelopes, and the application's backend holds a blob it cannot open. Merging them would make the host a keeper of credentials, which is what ADR-0018 refuses.

3. **The application's server never sees the password, the key, or the phrase.** It receives a proof derived from the master key, not the password; it stores ciphertext, not content. Whoever leaks its database gets back to no identity.

4. **One account holds several identities.** That is what makes this a vault rather than a login: `@imake.codes` for work and `@eworkprotocol.org` for personal life coexist, and the person picks which one they are acting as.

5. **No RFC changes.** This decision adds and alters no normative text. If one day it requires changing an RFC, that is the signal that it has leaked into the protocol and the boundary needs revisiting first.

## Alternatives considered

**A password on the e-work host.** Rejected. It contradicts ADR-0018 head on and would make the host hold an authentication factor, which changes what the host is. It would also erase the property that sustains federation: a host that only verifies signatures can be swapped for another; a host that holds your password cannot.

**A vault on the device only.** Rejected, with regret. It is the most private option, and nothing would leave the browser. But the vault exists for someone who expects to sign in from anywhere, and "export a file and import it on the other device" hands back the problem the vault came to solve, with one more step.

**The BIP-39 passphrase as the password.** Rejected. Beyond the MUST NOT in EW-RFC 0003 §8, it changes the derived key: forgetting the passphrase does not lock the vault, it produces **a different identity**. That is the difference between losing access and losing the person.

**Doing nothing.** Rejected, but it is the honest alternative and deserves the record: the twelve words work, and their cost is one of adoption, not of security. If the vault proves to be a worse vector than the barrier it removes, this decision becomes a new ADR.

## Consequences

**The vault is a convenience, and the kit remains the root of recovery.** Whoever forgets the password loses access through the vault, and recovers the identity with the twelve words, if they kept them. The interface MUST say this once, in those words, at the moment an identity enters the vault. Without that sentence the vault becomes a promise of recovery it does not keep, and the person finds out at the worst possible moment.

**The application's backend becomes a target, and a different target from the host.** It holds no identity; it holds the wrapping. Compromising it gives an attacker material for an offline dictionary attack against each account's password, which is why the derivation has to be expensive and its parameters stored per account, so they can be raised.

**A passkey fits later without a redesign.** The vault key is generated once and stored wrapped; adding a second wrapping through the WebAuthn PRF extension touches no stored phrase.

**The boundary needs watching.** The temptation to move a vault feature into the host will appear every time something is easier there. The test is item 5: if the change requires touching an RFC, it is on the wrong side of the line.

## References

ADR-0017 (recovery kit in words), ADR-0018 (no second factor at the host), EW-RFC 0003 §8, EW-RFC 0004 §1.
