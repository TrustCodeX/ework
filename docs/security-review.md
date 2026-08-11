# Security review of the specification

Adversarial review of 2026-08-08, covering EW-RFCs 0000 to 0016 and ADRs 0001 to 0020. An analysis document, not normative: nothing here changes the specification. What it produces is a queue of corrections, each with the normative text that is missing.

The complete inventory of the 146 items lives in the repository, at `docs/08-revisao-de-seguranca-inventario.md`. This document analyses and prioritises.

## Status of the corrections

The two batches of this report were corrected on 2026-08-08, in the same pass in which it was written. Deliberately left out was the body of medium and low items of the inventory, which remains a work queue.

Fourteen RFCs changed (0001 to 0009, 0011 to 0015), two new ADRs ([ADR-0021](decisions/adr-0021-authority-per-action.md), authority per action, and [ADR-0022](decisions/adr-0022-pseudonym-per-relationship.md), pseudonym per relationship), ADR-0017 corrected, six new requirements (RNF-19 to RNF-24), six glossary terms, five mitigations revised in the threat model and five new adversaries (A30 to A34). The fourteen corresponding English drafts were updated in the same pass, by the DoD's translation rule.

Two choices were made by the author's decision, not by deduction from the text: preserving unlinkability rather than withdrawing the promise, which is ADR-0022, and propagating to the drafts now rather than declaring the divergence as a pending item.

Read what follows as the record of what was wrong and why. The line citations point to the text **before** the correction, and that is how they must be read: they are the photograph of the problem, not of today's document.

## What has been closed since the review

This review is from 2026-08-08, and the queue it opened moved the same day. The document stays as it was written, because its value is the snapshot: changing the text afterwards would hide what the specification used to be. The current state goes here.

**Corrected in the normative text** (PR #114): the authority model per action, which became ADR-0021 and the "Who MAY sign" column of EW-RFC 0015 §2; unlinkability, which became ADR-0022 and took the root out of every object destined for an issuer; the recovery kit derivation, now complete and with test vectors in EW-RFC 0003 §8; the equality of edge responses, with the fifth case that was missing; the handle proof, which stopped being a lookup counter; history tie-breaking by hash instead of by clock; and the floor of components covered by the transport signature. Thirteen RFCs changed in all.

**Corrected in the reference implementation** (versions 0.3.0 and 0.3.1): the five items of batch 0 of the correction plan, all reproduced against real hosts before and after. Among them, one this review did not catch because it is not a specification failure: the federated edge recorded an incoming entry without verifying its signature, and an identity with no relationship at all wrote into anyone's history in the name of that inbox's owner.

**Still open**: batch 2 of this document, almost in full, and the low-severity items of the inventory. The complete inventory and the code correction plan stay in the repository, and not on this site: the plan describes implementation holes by file and line, and publishing them while open is publishing the map.

## Method

Seven independent reviews, one per area of the specification plus one cross-cutting, each followed by an adversarial verification whose job was to refute the findings of the previous one. The verifier had three obligations: check that the citation really exists in that file and line, search the whole specification for a mitigation living in another document, and downgrade inflated severity. When in doubt, refute.

A finding only entered the list with three things: the literal citation of the text, a concrete attack scenario (who the adversary is, what they do using only what the specification permits, what they obtain) and the proposed correction in normative language.

The "already acknowledged" criterion was applied strictly. `docs/04-modelo-de-ameacas.md` catalogues 29 adversaries and most RFCs have honest "Open questions" sections. An item already there only stayed on the list when the declared mitigation does not close the case, and it is marked as such.

## The result in numbers

134 raw findings from the area reviews, 4 refuted in verification, 130 kept, plus 16 findings from a final completeness pass whose only question was what everyone else had let through. 146 in total.

Of the 130 verified: 30 high, 62 medium, 38 low, none critical. The zero deserves explaining. The reviewers marked 19 as critical and the verifiers downgraded all of them, with a consistent argument: almost every serious attack here requires a malicious host or an already-consented issuer, and the threat model declares both as expected adversaries (A1, A2, A8). Downgrading was correct by the adopted yardstick, and should not be read as comfort. The priority queue in this document is mine, not the average of the scores, and the first two items of batch 1 came precisely from the completeness pass, which ran without that yardstick.

By verdict, across the 130: 49 confirmed (the protection does not exist) and 81 partial (the protection exists but is insufficient, or lives only in `docs/`, or only with SHOULD where security depends on MUST).

That proportion is the most useful finding of the whole review, and it is worth more than any single item.

## The underlying pattern

The specification thinks about security with uncommon seriousness. What fails, almost always, is not the idea: it is the journey from the idea to the normative sentence an implementer is obliged to obey.

Three forms of the same problem, and they explain most of the 146:

**The mitigation lives in `docs/` and not in `rfc/`.** The threat model asserts protection that no RFC requires. Whoever implements reads the RFC, and rule 7 of `CLAUDE.md` says the RFC wins. A defence that exists only in the threat model does not exist.

**The rule exists in one RFC and is missing from the sibling RFC that defines the same thing.** The clearest case is attachments: RFC 0005 §5 says, for federation, that knowing a hash is not authorisation, and requires a list of authorised hosts per blob. RFC 0004 §5, which defines the same API for clients, has no equivalent sentence. The defence was written once and the other side stayed open.

**The sentence exists, with SHOULD.** Message padding, beneficiary divergence checking, gap signalling in history. These are exactly the points where the guarantee depends on the behaviour happening, and SHOULD obliges nobody.

None of these three is a design failure. All are closing failures, and that is good news: the fix is writing, not redesigning.

## Batch 1: fix before writing more code

### 1. There is no authority model per action: any participant signs any transition

The action vocabulary table of RFC 0015 §2 has three columns: `type`, effect and parameters. It does not have the fourth, which is who MAY sign each one. I searched the whole specification and the only statement of authority over an action is in `rfc/0015-comentarios-historico.md:84`, and it covers `attach` alone. The filter of RFC 0004 §4.1 is by state, never by author: "An action incompatible with the current state MUST be recorded as an entry without effect".

Consequence: inside a group they belong to by right, any participant signs any transition, and the entry is valid by every criterion the specification enumerates. The signature verifies, `after` closes the chain, and the action is compatible with the current state. Every conforming client applies it.

Issuer Acme signs `accept` on its own offer and produces a signed record that the charge was accepted, usable in a dispute and in collection. Supplier Beta signs `complete` to unblock its own delivery, or `reassign` pointing at whoever it likes, or `cancel` on someone else's task, or `update` rewriting the deadline and the amount. RFC 0015 §8 sells the list of signed entries as "the audit artefact", and it is exactly that artefact which is forged here.

It is worth understanding why the per-document review missed this and the completeness pass caught it: the failure is in no sentence, it is in the column the table does not have. Hash chaining and immutability, which RFC 0015 has and defends well, protect against erasing the past, and say nothing about writing in the present in someone else's name.

Correction: add to RFC 0015 §2 the normative authority column, with the explicit default for each action. Proposal: `accept`, `decline` and `counter` only by the recipient of the offer; `start`, `complete` and `fail` only by the executor; `confirm` and `contest` only by whoever `ework:execution.confirmedBy` authorises; `update` only by the offerer; `cancel` by the owner or by the issuer of their own offer; `reassign` only by the owner; `snooze` and `ack` only by a notified participant. And rewrite the rule of RFC 0004 §4.1 as "An action incompatible with the current state OR signed by an identity without authority for it MUST be recorded as an entry without effect", so that the attempt stays in the history and stays out of the state, which is the right answer and is already the spirit of the current sentence.

### 2. History re-sharing ignores the compartment and hands over the whole project

`rfc/0006-criptografia.md:39` is normative and mandatory: when adding a member to a project with `historyForNewMembers: "reshare-all"`, an existing member "MUST send to the group, in the new epoch, a snapshot of the state **of the project** (tasks + metadata + keys of the referenced attachments)". The default is `reshare-all`, declared in RFC 0002 §9 line 168.

But RFC 0013 defines one MLS group per compartment, not per project. So adding the supplier to the narrow compartment triggers, to the letter, sending the whole project's state to that compartment's group, including the keys of the referenced attachments. It is the literal negation of the promise that opens ADR-0011 and RFC 0006 §1, that "the assembler does not see the client's payment" stops being an interface promise and becomes a cryptographic property.

The aggravating factor is in `rfc/0013-compartimentos-dados-sensiveis.md:149`, and it is this report's underlying pattern in its sharpest form: RFC 0013 noticed the problem and wrote that re-sharing "here SHOULD be per task rather than in bulk". The strong word is on the dangerous behaviour, in one RFC, and the weak word is on the protection, in the other. An implementer who follows both documents to the letter obeys the MUST and leaks.

No adversary is even needed: a conforming client and an admin doing the most ordinary thing in the world, which is adding a supplier to a circle.

Correction: change the object of the snapshot in RFC 0006 §3 from "the project's state" to "the state of the compartment whose group receives the snapshot", with "The snapshot MUST NOT contain a task, metadata or attachment key belonging to a compartment the destination group does not own". Promote the SHOULD of RFC 0013 §7 to MUST in the same pass, and add to the conformance suite a test that adds a member to a narrow compartment in a project with a wide compartment and verifies that nothing from the wide one crossed over.

### 3. The recovery kit derivation is specified nowhere

`rfc/0003-identidade.md:186` says the kit is a twelve-word BIP-39 phrase, that "the entropy derives the root and the backup key", and RFC 0006 §5 repeats that the backup key is "derived from the kit". No RFC says how. There is no named derivation function, no domain separation, no ordering, no decision on the BIP-39 passphrase, no test vectors.

There is an aggravating factor that only appears when comparing languages. The English translation of ADR-0017 (`docs/en/decisoes/adr-0017-kit-de-recuperacao-em-palavras.md:24`) carries an item the Portuguese version does not have, and it is precisely the security item: "The derivation uses its own salt", with `PBKDF2(phrase, "mnemonic" + password)` and a salt starting with `ework`, so that the same twelve words produce different keys here and in a cryptocurrency wallet. The decision is good and the reason is correct, because reusing the phrase between the two systems would tie the compromise of one to the other. Except it exists only in the translation, and Portuguese is canonical for now. I compared the 28 translated document pairs in the repository and this is the only structural divergence: it is an isolated case, not a systemic problem, which makes the fix trivial.

So the real state is worse than a simple gap: the decision was made, it is written somewhere, it is not in the canonical document and it is not in any normative text.

This is the only item on the list that needs no adversary. Two conforming implementations derive different keys from the same twelve words, and the kit stops recovering the account in the second client. It is literally the scenario of ADR-0017, the person dictating the words to someone using another app. In a protocol whose RFC 0003 §8 declares there is no second copy, that is definitive loss of identity and of all encrypted content. And without domain separation, an implementation may derive root and backup from the same material, turning the leak of one into the compromise of the other.

Correction, in two parts. First, bring into ADR-0017 in Portuguese the item that exists only in the translation, so that the salt decision exists in the canonical document. Second, write the complete derivation in normative text in RFC 0003 §8, starting from the decision already made and completing what it does not fix: the exact salt string, the derivation order, the domain separation between the root and the backup key (proposal: distinct labels per purpose, `ework/v1/root` and `ework/v1/backup`, so that leaking one does not hand over the other), the iteration count, the output size, the mapping to Ed25519, and the explicit decision on the BIP-39 passphrase. Close with normative test vectors in the RFC itself, which is the only way two implementations can prove they agree.

I put it at the top of the items that are neither about authority nor compartments because it is the cheapest to fix, the one with the most irreversible consequence, and the only one on the whole list that dispenses with an adversary.

### 4. The contact key has no validation chain, and the host can be the other side of the relationship

`rfc/0003-identidade.md:78`: "Contact addresses have no public proof: whoever needs to validate them is the host that routes them, and the counterparty, who received them from the user themselves at the moment of consent."

The contact key is the credential presented in the relationship's MLS group (RFC 0006 §5). By design decision it does not appear in the public identity document. The `ContactKey` object has `boundBy` with a root signature, but no RFC defines that this object reaches the counterparty: the `Consent` of RFC 0007 §1 carries the `alias`, which is the address, and does not carry the contact public key. The handshake of §3 never transports it.

Consequence: when Acme receives the group's `Welcome`, the only thing attesting that the tree leaf is Cléia is the host that delivered it. A malicious host inserts its own key, sits permanently in the middle of the relationship, and reads and alters the content that passes through: invoice, payment slip, test result, appointment. RFC 0006 §5 states the opposite in so many words, that "the issuer validates who it is talking to".

Correction: the `consent.grant` MUST carry the contact public key of that relationship, with proof binding it to the signed consent; the credential presented in the relationship group MUST be exactly that key; and implementations MUST NOT accept a credential presented only by the host. The care to take is not to reintroduce the link to the root while doing so, which leads to item 8.

### 5. The transport signature does not fix what it covers

`rfc/0004-sincronizacao-cliente-servidor.md:39` requires the client to sign the session request with RFC 9421. RFC 0005 §2 does the same for federation. Neither says which components enter the signature base, and RFC 9421 is only safe when the specification using it fixes that.

Without `@target-uri` or `@authority` covered, a signature produced for one host works at another: host A asks host B for the session challenge in the victim's name, returns B's nonce to their client as if it were its own, and receives back a signature valid at B. Since RF-06 foresees an identity across several hosts, that scenario is the normal one, not the exotic one. Without a mandatory `content-digest`, any intermediary swaps the body of a signed request in federation.

Correction: a normative list of covered components in RFC 0004 §1 and RFC 0005 §2 (`@method`, `@target-uri`, `content-digest`, `content-type`, `created`, `expires` with a maximum window, `nonce`), with "Receivers MUST refuse a signature whose set of covered components does not contain exactly these", and `Content-Digest` (RFC 9530) mandatory on every POST with a body.

### 6. In the client blob API, knowing the hash is the credential

`rfc/0004-sincronizacao-cliente-servidor.md:77` defines attachment download by hash without a single authorisation rule: no link between the blob and the session requesting it, no mention of an inbox, not even a requirement that the endpoint be authenticated. Line 75 even pushes the implementer in the wrong direction by celebrating that "the id IS the hash: idempotent and deduplicating by construction".

RFC 0005 §5 solved this for federation and the rule was not mirrored here. Any user of the same host, including a free account at a public provider, reads someone else's attachment if they know the hash. In assisted mode, those attachments are in the clear. Even with full E2EE, the endpoint confirms possession of a specific file, which is a leak in itself.

Correction: mirror the federation rule in RFC 0004 §5, with "Knowing the hash is NOT authorisation", an `unauthorized` response indistinguishable in code, body and time from that of a non-existent blob, and an upload response indistinguishable between a new blob and one that already exists.

### 7. The address validation oracle the specification swears it does not have

Three independent findings, from three different reviewers, on the same mechanism. It is the theme with the most convergent evidence in the review.

RFC 0011 §4.2 (`rfc/0011-anti-abuso.md:60`) requires answering indistinguishably for an address that is "non-existent, retired and revoked". RFC 0003 §7.4 repeats the same list of three. Missing from that list is the most common state of all: a live address, with no relationship with whoever is knocking. And for that case RFC 0005 §4 requires answering `no-consent`.

They are two different permanent codes. The collector fires any `task.offer` at each address of a scraped list, from a disposable domain, and reads the answer: `no-consent` is a live address, `unknown-recipient` is a dead one. One request per guess, and out comes the verified list that RFC 0003 itself calls "exactly the product the spam industry buys and sells". It is the direct negation of ADR-0008 and of the promise in RFC 0007 §4 that a scraped address delivers nothing.

The third finding is worse because it does not even need an account: the handle proof in `rfc/0003-identidade.md:73` answers `GET /.well-known/ework/handles/<name>` with the `did:key` of whoever exists, and `unknown-recipient` for whoever does not, to anyone on the internet, without a quota. Each hit returns the search key of the identity document, which publishes complete `handles` and `hosts`: discovering one handle of a person hands over all the others and all of their hosts, which undoes the context separator sold in RFC 0003 §9 and contradicts RFC 0012 line 150, which declares it impossible to produce the list of who has an account.

Correction: unify the edge response. `unknown-recipient` for every envelope the edge does not accept, without distinguishing non-existent, retired, revoked, blocked or live-without-relationship; `no-consent` only when the sender already holds a credential for that address, that is, when they already knew the relationship existed. Add the missing state to the lists of RFC 0003 §7.4 and RFC 0011 §4.2, and include latency equality in the requirement. On the handle proof, require the querier to already know the target `did:key` and answer only confirmation, and restrict the document's `handles` and `hosts` to the handle and host through which it is being served.

### 8. The payment-slip scam comes in through the task revision

Scenario A3 of the threat model is the project's most concrete promise: the fake payment slip scam "dies by construction". Four findings show that it does not die, and none of them needs to break cryptography. All use an already-consented issuer, or a compromised issuer gateway, which RFC 0010 §3 concentrates in a single key.

**The payment instrument is not pinned to the series.** `rfc/0008-cargas-tipadas.md:64` pins only `beneficiary.taxId`. The attacker sends an `update` revision on the same `dedupKey`, keeps beneficiary, amount and deadline identical, and swaps only the `pix.brcode` and the digitable line. Nothing fires: the `taxId` did not change, the amount did not get worse (RFC 0007 §5 requires a new acceptance only for amount or deadline "for the worse"), and the client may offer one-tap acceptance. The person pays the scammer inside a task they had already checked and accepted.

**The action target is not bound to the payload.** `rfc/0002-modelo-de-dados.md:130`: every defence in RFC 0008 §2 operates on the `payload` object, and no rule ties the `target` of a `deeplink` action to it. The issuer sends the correct `payload.beneficiary`, the correct tax id, the correct amount, and the primary action pointing at the scammer's brcode. The conforming client shows the legitimate beneficiary prominently, finds no divergence at all to warn about, and the button leads to the wrong payment.

**The URI scheme is unrestricted.** `rfc/0008-cargas-tipadas.md:65` and RFC 0002 §6 require only "displaying the destination". There is no closed list of schemes. In the reference web client, embedded in the host binary, a `target` with `javascript:` or `data:` in an anchor element is script execution in the application's context, with access to key material and to already-decrypted content.

Correction: pin the whole instrument to the series (`pix.brcode`, `pix.txid`, `boleto.barcode`, `boleto.linhaDigitavel`) and not only the `taxId`, with an explicit new acceptance required and one-tap acceptance forbidden when any of them changes; treat an instrument swap as a change "for the worse" in RFC 0007 §5; derive the payment action's target from the payload itself, with divergence blocking the action rather than warning; and a closed list of schemes (`https`, `pix`, `tel`, `mailto`, `geo`), with `javascript:`, `data:`, `file:` and `intent:` refused under any circumstances.

### 9. The SRV target does not prove the domain authorised it

`rfc/0001-nucleo.md:96` argues that "a poisoned SRV leads to a host that cannot produce a valid signature for the sender it claims to represent, so the attack fails at validation, not at resolution".

The argument does not hold for inbound. Whoever controls DNS publishes `_ework._tcp.victim.com` pointing at their own host, serves `/.well-known/ework` with a legitimate TLS certificate for their own name, and declares `addressDomain: "victim.com"`. The check required at line 69 passes, because whoever declares the `addressDomain` is the target itself, and nothing obliges the target to prove the domain authorised it. By the rule at line 86 the discoverer stops there and never queries the apex.

The attacker then receives every POST destined for the domain: the complete routing header of every user (who talks to whom, type, size, cadence) and the plaintext body of every `consent.request`, which by line 124 travels unencrypted. They cannot forge an outbound signature, and they do not need to: the attack is interception, not impersonation.

There is a second layer, and it explains why the argument at line 96 fails at the root: the `hostKey` that the transport signature of RFC 0005 §2 uses as an anchor is born from that same unsigned document. The defence against hop forgery is bootstrapped by the document the attacker serves, so it does not protect the case §3.4 says it protects.

Correction: define `delegatedFrom` as signed proof and not as text. The domain publishes the delegation key in DNS (for example a TXT at `_ework.<domain>`, cheap for the same reason that justified the SRV), and the document at the target MUST carry an object signed by that key covering the tuple (`addressDomain`, target name, `hostKey`, validity). Discoverers MUST refuse a document whose `addressDomain` differs from the target's own domain without that proof, and MUST NOT use `hostKey`, `clientApi` or `federationInbox` obtained by that route before validating it. Add trust on first use: a change of `hostKey` for an already-known domain MUST be treated as suspicious and flagged. And fix the text of §3.4, which today asserts a property the design does not have.

## Batch 2: close before calling v0.1 stable

**A relationship between people is born without a revocable object.** `rfc/0007-consentimento-emissores.md:61` with §3.5: a relationship between people creates no `Consent`. The two termination paths then do not work. Polite revocation operates on a `Consent` that does not exist; silent retirement operates on the contact address, but RFC 0003 §6 says people invited personally use the main handle, and retiring it would cut every personal relationship at once and take down the handle proof. The adversary here is the ex-spouse, the ex-partner, the harasser, and the protocol has no specified exit door for them.

**There is no urgency ceiling for whoever holds a personal relationship.** `rfc/0011-anti-abuso.md:50`. The only ceiling is the `maxUrgency` of the `Consent` scope, and a relationship between people creates no scope. After first contact, `critical` is free: maximum sound, pierces silent mode, demands ack, triggers escalation, in series, in the middle of the night. Added to the previous item, the protocol hands over a harassment tool with no off switch. Correction: default `maxUrgency` of `normal` for a personal relationship, raised only by an act of the recipient, with mandatory downgrade by host and client.

**Removing a member from the project does not remove them from the group.** `rfc/0013-compartimentos-dados-sensiveis.md:150` is a descriptive sentence: it does not say MUST, does not say who issues the MLS Remove, does not say within how long, and does not tie editing `compartments.<id>.members` to issuing the commit. The admin takes the supplier off the list, the interface shows them out, and nobody issues the Remove. The ex-member stays in the current epoch and keeps reading everything. This is the most direct hole in the central thesis of ADR-0011, because the list became permission and the key stayed where it was. The sibling finding (`compartimentos-conformidade-01`) shows the root: the roster exists in two places, the `Project` JSON and the MLS tree, and no text requires reconciling them.

**Who may Add and Remove is an open question.** `rfc/0006-criptografia.md:92`. The Security considerations of the same file (line 84) claim that a malicious host "cannot forge membership, because Adds require the signature of a member authorised by the group policy". The policy backing that sentence is precisely what the document admits it has not written. Add to that `reshare-all` being the default of §3, and a single member who can add someone hands over the compartment's entire history along with them. The verifier classified it as medium; I consider the combination worse than that, because both ends are in the same file.

**Assisted mode on one inbox hands over someone else's compartment.** `rfc/0006-criptografia.md:72`. The mode is per inbox, and the host's device joins that inbox's personal group. Except the inbox contains the tasks of the projects the person takes part in, and server-side querying requires reading a decrypted Task. One member turning on assisted mode is enough for their host to start seeing that circle's content in the clear, and the other members have no way of knowing. The threat model's security anti-requirement says "never silently degrade E2EE", and here the degradation is silent for everyone except whoever turned it on.

**The sealed section is not bound to the task carrying it.** `rfc/0013-compartimentos-dados-sensiveis.md:87`. The structure carries neither a nonce nor associated data, and nothing ties the sealing to the task's `uid` nor to the object version. A member of the wide compartment, who sees the task and does not open the seal, cuts the block from another task and pastes it into this one with an `update` revision. The narrow compartment's client decrypts successfully and shows the content in the wrong context.

**The actor class is self-declared.** `rfc/0014-atores-autonomos-confirmacao.md:87`. The `Entry` carries `actorClass`, and the only normative obligation written down is to display it when it is not `human`. Nothing requires the receiver to resolve the key in the identity document and use the class signed by the root. A compromised agent signs an entry declaring itself human, with a valid identity signature, and satisfies `humanApproval`. The central property of ADR-0012, that human approval is a signature by a human-class key, depends on a verification no RFC requires.

**The identity document has no validity per key.** `rfc/0003-identidade.md:53`. Each device has `addedAt` and none has `validUntil`; revoking is removing from the document, and nothing requires keeping previous versions. Two opposite consequences, both exploitable: yesterday's legitimate signatures become unverifiable (cheap repudiation, and RFC 0015 §10 promises immutability), and a verifier with an old document keeps accepting an already-revoked key.

**History chaining only looks backwards.** `rfc/0015-comentarios-historico.md:130`. `after` declares what the author knew, so withholding an entry together with all the ones referencing it produces, for the victim, an internally coherent suffix indistinguishable from legitimate concurrency in the DAG. The host shows different stories to different members without leaving a trace. Correction: include in `after` the known current head of each co-author, publish signed heads periodically, and swap the SHOULD of flagging gaps for a MUST.

**The sealed section can cover the authorisation fields.** `rfc/0013-compartimentos-dados-sensiveis.md:89`. Nothing restricts which paths may be sealed, and nothing forbids two overlapping sections. Sealing `/ework:execution` produces a signed task with two meanings: whoever is in the narrow compartment reassembles it and reads `humanApproval: "none"`, and whoever is in the wide one reads `humanApproval: "before"`. The same signature holds for both readings, which is worse than divergence, because each side has proof of being right.

**There is no ciphersuite floor and no negotiation rule.** `rfc/0006-criptografia.md:78` requires supporting suite coexistence and says new groups are born "on the best available suite", without defining what best means nor who decides. Whoever creates the group chooses, and there is no sentence requiring refusal below a floor. Downgrade becomes the adversary's choice, which is the classic form of this failure, and it contradicts the legitimate concern with "harvest now, decrypt later" in the same section.

**The key backup does not rotate, does not expire and does not destroy itself.** `rfc/0006-criptografia.md:54`. The host accumulates ciphertext for years and also keeps the backup that reopens it. Nothing obliges it to replace the previous backup when receiving a new one, nor to destroy the old one. Whoever obtains the kit a single time, and it is paper kept by non-technical people, reopens the entire archive retroactively. This interacts badly with the cryptographic destruction that RFC 0016 uses as an erasure mechanism.

**A pending consent request can be replaced.** `rfc/0005-federacao.md:47` allows a resend to replace the pending request, and RFC 0007 §3.2 gives 90 days of validity to a deliberately silent inbox. The issuer sends a modest scope, the person reads it and leaves the decision for later, and the issuer swaps the request for another with a wide scope. What they accept is not what they read. Correction: replacement MUST preserve the scope, or reset the read state and flag the change.

**History ordering is decided by fields the author chooses.** `rfc/0015-comentarios-historico.md:128` breaks ties by `(sentAt, uid)`, and no RFC limits `sentAt` nor requires checking it against the moment of receipt. With `sentAt` in the future, a revision wins the last-write-wins of RFC 0004 §4.1 forever, and no legitimate later edit can override it. The threat model has no clock adversary. Correction: refuse `sentAt` too far ahead of the local instant, break ties by the entry's own hash instead of by `sentAt`, and compare by `min(sentAt, receivedAt)` in last-write-wins.

**Delivery gives up after three days and nobody finds out.** `rfc/0005-federacao.md:39`. After 72 hours each sending host gives up and informs only its own sender. The victim's host comes back online and has no way of knowing what it missed, because no resynchronisation mechanism exists. Against a self-hoster, which is the project's reference design, 72 hours of unavailability are cheap to cause and happen on their own during a trip. For the threat model's "availability of critical reminders" asset, this is silent loss.

**The promised unlinkability is false today.** `rfc/0006-criptografia.md:51` claims two issuers cannot correlate the same user. But the `Consent` of RFC 0007 §1 carries `user: did:key:...` of the root, the sending credential of §4 does too, and the `Entry` of RFC 0015 §1 has `author` in the same format. Two issuers cross databases by the global, stable identifier that RFC 0003 §7.5 says is not rotated day to day. A side has to be chosen: either these objects come to use the identifier of that relationship's contact key, or the claim leaves RFC 0006 and RFC 0003 §6 and becomes a declared risk.

## What held up

Worth recording, because a list of only holes distorts the portrait.

The compartment-per-MLS-group design, with one group per compartment rather than one group per project, resisted every angle of attack tried: the thesis of ADR-0011 is solid, and the problems found are closing failures at the edge, not design failures. The absence of a civil identifier (ADR-0010) did in fact eliminate the entire class of attacks it meant to eliminate, and the argument that you cannot be coerced into linking a bond the protocol does not implement held up. The refusal to have a second factor on the host (ADR-0018) produced no authentication finding that a second factor would solve. The threat model itself is honest above average: the 37 items marked as already acknowledged are proof that the authors saw the problems before this review, and the reason they stayed on the list is the distance between seeing and obliging, not a failure of perception.

## How to use this queue

The RFCs that concentrate the work, in order of volume: 0003 (identity), 0008 and 0002 (payloads and data model, the payment theme), 0004 and 0005 (the client and federation pair, where the rules need mirroring), 0013 (compartments) and 0007 (consent, with the missing personal-relationship object).

I suggest three batches, in this order.

The first is items 1 and 2, and they come before everything because they are not missing sentences: they are properties the specification promises and does not deliver, each one holding up an entire ADR (0014 in the first case, 0011 in the second). While they stay open, any new implementation builds on a guarantee that does not exist.

The second batch is items 3, 5 and 6, which are isolated sentences, with no side effects in another document, and each resolves in one writing session.

The third batch is items 7, 8 and 9, and each must be treated as a coordinated change, because it touches three RFCs at once and a partial correction leaves the hole open from the other side. The oracle in particular only closes when all three endpoints answer alike, and RFC 0003 §7.4 already says it better than I would: "The absence of an oracle is a property of the set of endpoints, never of one of them in isolation".

Each change closes through the DoD of [06-definicao-de-pronto.md](definition-of-done.md), and those that alter a recorded decision require a new ADR by rule 6 of `CLAUDE.md`, never a rewrite of the existing ADR. At least three should become ADRs: the authority model per action, the choice between unlinkability and issuer-verifiable authorship, and the Add and Remove policy per group type, which is today an open question of RFC 0006 and holds up the security claim in the same file.

## References

- [04-modelo-de-ameacas.md](threat-model.md), the 29 catalogued adversaries, against which each finding was checked
- `docs/08-revisao-de-seguranca-inventario.md`, in the repository, the 130 items with location and verdict
- [06-definicao-de-pronto.md](definition-of-done.md), the process that closes each correction
