# ADR-0017: The recovery kit is twelve words, in the language of whoever keeps it

**Status:** accepted (2026-08-06)

## Context

EW-RFC 0003 §8 says what the recovery kit **does**: it is mandatorily generated when the identity is created, it derives the root key and the backup key, and the interface MUST force the safekeeping step before releasing use. It also says the format is born compatible with threshold splitting, for emergency access and inheritance.

EW-RFC 0003 §8 is blunt: without the recovery kit the identity is unrecoverable by design, and there is no second copy. The kit is therefore the only thing standing between a person and the permanent loss of their identity, and its shape is not presentation: it is the surface where the protocol touches the life of someone who is not technical.

Today it is 32 bytes in base64, which the MVP shows once. That fails at four concrete points, and none of them is aesthetic:

- it cannot be copied by hand without error;
- it does not detect error, so a wrong character produces a valid, different key that only fails months later;
- it cannot be read aloud, which rules out the phone call to a relative;
- it cannot be checked against a piece of paper, so nobody knows whether what they wrote down is right.

The design handoff asks for "twelve words on a tinted background, with a Print button". The question it does not answer, and which decides the implementation, is what those words encode and in which language they are generated. It is also why issue #26 was blocked: implementing the words without deciding the format would be defining protocol inside a form.

## Decision

**The kit is twelve BIP-39 words, encoding 128 bits of entropy, generated in the language of the interface at the moment of creation.**

1. **Word list: BIP-39.** We do not invent vocabulary where a standard exists, and this one has official lists in nine languages, a checksum built in, and two decades of interoperable implementations to check against.

2. **Words encode entropy, not the key.** The root key and the backup key are *derived* from that entropy. Encoding the key directly would leave the kit with nothing but the root, and the backup key of §8 would have nowhere to come from. Derivation also leaves room for the threshold splitting sketched in §8 without the kit having to change shape later.

2b. **The derivation uses its own salt.** BIP-39 derives the seed with `PBKDF2(phrase, "mnemonic" + password)`. Here the salt is `ework-mnemonic`. This is not decoration: it guarantees that the same twelve words produce **different** keys here and in a cryptocurrency wallet. Without it, a person who reused the phrase across both systems would tie the compromise of one to the other, and e-work would become a path to stealing a wallet. The full derivation, with domain separation between the root key and the backup key, is normative in EW-RFC 0003 §8.

    This item existed only in the English translation of this ADR, not in the original. The security review of 2026-08-08 found the divergence: the decision had been made, written in a single place, and absent from the canonical document and from every normative text. Two conforming implementations would derive different keys from the same twelve words, which in a protocol with no second copy is permanent loss of identity. Putting the item back is not changing our mind; it corrects a loss in the passage between languages.

3. **The language is the holder's, and it is detected on the way back.** The kit is generated in the language of the interface, because whoever writes the kit down is the owner of the account, at the moment she creates it. On entry the language comes from the words themselves: the official word lists share no word, so the interface never has to ask "which language is your kit in?" of someone who is trying to recover an account.

4. **The old format is accepted forever.** Accounts created before this decision have a base64 kit, and they do not stop existing. What changes is what the client *generates*, never what it *accepts*.

5. **Validation is local and immediate.** The BIP-39 checksum is checked in the browser before any call. A mistyped word is pointed out as a wrong word, right there, instead of turning into "invalid signature" later.

## Alternatives considered

**Keeping base64.** Rejected. It is the format that costs nothing to keep and everything to use: it cannot be dictated, checked or corrected, and its failure mode is silent. The argument in its favor was not adding a word list to the client, which is real weight (about 2 KB compressed per language). Two thousand words against the chance of someone losing their identity for reading an O as a 0 is not a hard trade-off.

**Twelve words always in English.** This was the first inclination, on the argument that whoever helps might not speak the owner's language. Rejected because it inverts who pays the cost: the person who writes the kit down is the owner, at creation time. Whoever helps is the rarer case and the better-prepared one.

**A passphrase chosen by the person.** Rejected: human-chosen entropy is weak in predictable ways, and here there is no server to limit attempts, because the key is used locally. A memorable phrase would be guessable offline.

**Our own words, instead of BIP-39.** Rejected by rule 5 of the repository: do not invent vocabulary where a standard exists. A list of our own would mean writing and reviewing the list, the checksum algorithm and the encoding scheme, and then convincing the second implementation to copy it all.

**Twenty-four words (256 bits).** Rejected. 128 bits of entropy is already beyond any brute-force horizon for this threat model, and doubling the length doubles the chance of the person giving up halfway through writing it down, which is the failure that actually happens.

**Encoding the key directly in the words.** Rejected for the reason in decision 2: it closes the door on the backup key and on threshold splitting, both of which the RFC already foresees.

**Optional custody at the host.** Out of scope here: EW-RFC 0006 §7 already provides for it, as an explicit choice by whoever does not want to keep anything, and it changes the threat model. It is not a substitute for the kit; it is the alternative for whoever decides to give up ownership.

## Consequences

- EW-RFC 0003 §8 gains the definition of the format, with the list of supported languages and the rule for detection on entry. #26 (building the seven-step box) stops being blocked.
- The client ships the word list of the active language, and only that one: carrying all nine in the bundle would double its size for a case that almost never occurs, and detection on entry loads the others on demand. The test suite checks the specification's own vectors: an implementation that is internally consistent and wrong would produce kits no other tool can read, and a round-trip test alone would not catch that.
- **A visual link to the crypto world.** Twelve BIP-39 words are recognizable as a wallet "seed phrase", and e-work says explicitly that it does not use blockchain. The text on the screen needs to say what the phrase is (the key to the account, kept by its owner) without inheriting the expectation that there is a coin behind it. That is copy work, and it is tracked in #48.
- **Kit migration does not exist, and that is deliberate.** Whoever has a base64 kit keeps it. Offering "generate a new kit in words" would create a moment when two valid kits exist at the same time, and no interface can guarantee that the person destroyed the old one.
- The confirmation step in account creation becomes possible: asking for three of the twelve words back is reasonable, while asking for three characters of a base64 string is asking someone to verify noise.
- The wrong-order case, which vocabulary checking alone lets through, is caught by the checksum with a message that says what happened.
- Nothing is promised about a leaked kit: whoever holds the words gets into the account. This decision does not improve that case; it refuses to pretend it does.
- The threshold splitting of §8 now operates on the entropy, not on the key. That is the correct shape: the shares reconstruct the secret that derives the keys, not a key already derived.
