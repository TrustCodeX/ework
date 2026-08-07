# ADR-0017: The recovery kit is twelve words, in the language of whoever keeps it

**Status:** accepted (2026-08-06)

## Context

EW-RFC 0003 §8 is blunt: without the recovery kit the identity is unrecoverable by design, and there is no second copy. The kit is therefore the only thing standing between a person and the permanent loss of their identity, and its shape is not presentation: it is the surface where the protocol touches the life of someone who is not technical.

Today it is 32 bytes in base64, which the MVP shows once. That fails at four concrete points, and none of them is aesthetic:

- it cannot be copied by hand without error;
- it does not detect error, so a wrong character produces a valid, different key that only fails months later;
- it cannot be read aloud, which rules out the phone call to a relative;
- it cannot be checked against a piece of paper, so nobody knows whether what they wrote down is right.

The design handoff asks for "twelve words on a tinted background, with a Print button". The question it does not answer, and which decides the implementation, is what those words encode and in which language they are generated.

## Decision

**The kit is twelve BIP-39 words, encoding 128 bits of entropy, generated in the language of the interface at the moment of creation.**

1. **Words encode entropy, not the key.** The root key and the backup key are *derived* from that entropy. Encoding the key directly would leave the kit with nothing but the root, and the backup key of §8 would have nowhere to come from. Derivation also leaves room for the threshold splitting sketched in §8 without the kit having to change shape later.

2. **The derivation uses its own salt.** BIP-39 derives the seed with `PBKDF2(phrase, "mnemonic" + password)`. Here the salt starts with `ework`. This is not decoration: it guarantees that the same twelve words produce **different** keys here and in a cryptocurrency wallet. Without it, a person who reused the phrase across both systems would tie the compromise of one to the other, and e-work would become a path to stealing a wallet.

3. **The language is the holder's, and it is detected on the way back.** The kit is generated in the language of the interface, because whoever writes the kit down is the owner of the account, at the moment she creates it. On entry the language comes from the words themselves: the official word lists share no word, so the interface never has to ask "which language is your kit in?" of someone who is trying to recover an account.

4. **The old format is accepted forever.** Accounts created before this decision have a base64 kit, and they do not stop existing. What changes is what the client *generates*, never what it *accepts*.

## Alternatives considered

**Keeping base64.** Rejected. It is the format that costs nothing to keep and everything to use: it cannot be dictated, checked or corrected, and its failure mode is silent.

**Twelve words always in English.** This was the first inclination, on the argument that whoever helps might not speak the owner's language. Rejected because it inverts who pays the cost: the person who writes the kit down is the owner, at creation time. Whoever helps is the rarer case and the better-prepared one.

**Twenty-four words (256 bits).** Rejected. 128 bits of entropy is already beyond any brute-force horizon for this threat model, and doubling the length doubles the chance of the person giving up halfway through writing it down, which is the failure that actually happens.

**Encoding the key directly in the words.** Rejected for the reason in decision 1: it closes the door on the backup key and on threshold splitting, both of which the RFC already foresees.

## Consequences

- The client vendors the two official BIP-39 word lists, and the test suite checks the specification's own vectors. An implementation that is internally consistent and wrong would produce kits no other tool can read, and a round-trip test alone would not catch that.
- The confirmation step in account creation becomes possible: asking for three of the twelve words back is reasonable, while asking for three characters of a base64 string is asking someone to verify noise.
- The wrong-order case, which vocabulary checking alone lets through, is caught by the checksum with a message that says what happened.
- Nothing is promised about a leaked kit: whoever holds the words gets into the account. This decision does not improve that case; it refuses to pretend it does.
