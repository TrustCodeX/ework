# Definition of done (DoD)

This document defines what "I am finished" means for changes to the specification. It applies to humans and to agents alike. The rule exists because a suite of RFCs is a coupled system: touching one document without propagating the change produces a specification that contradicts itself, and a specification that contradicts itself is not implementable.

Most of these items are verified by `uv run scripts/check_spec.py`. Whatever the machine cannot check is marked as human judgement.

## DoD common to any change

1. **Header updated.** Any document touched has had its status and its date reviewed. An RFC carries `**Status:**` and `**Created:**`; an ADR carries `**Status:**` with a date and who decided.
2. **No em dashes.** No long dashes (U+2014 em dash, U+2013 en dash) in any file of the repository. Prefer a comma, a colon, a full stop or a new sentence.
3. **Links resolve.** Every relative link points at a file that exists.
4. **Indexes up to date.** A new RFC goes into the table of [RFC 0000](../spec/0000-index-and-process.md); a new document in `docs/` goes into the map in the README.
5. **Vocabulary.** A new term goes into the [glossary](glossary.md) with its English equivalent, because the canonical translation comes later.
6. **Site regenerated.** `make site` runs without error (the site reads the markdown directly, so a new document appears on its own; what breaks is a link or malformed markdown).
7. **Validator clean.** `make check` exits with code 0.

## DoD for a change to an RFC

8. **The seven sections exist:** Summary, Motivation, Specification, Security considerations, Privacy considerations, Open questions, References. An empty section is a sign that the change is not ready, not that the section does not apply.
9. **Deliberate normative words.** MUST, MUST NOT, SHOULD, SHOULD NOT and MAY in capitals mean what [RFC 0000](../spec/0000-index-and-process.md) §2 says they mean. If the sentence is not normative, do not use the capitalised word.
10. **Requirements traced.** A requirement cited (RF-xx, RNF-xx) exists in [02-requisitos.md](requirements.md); a new requirement goes in there with a priority and a "Where" column pointing at the RFC.
11. **Neighbouring RFCs checked.** A change that alters the envelope, identity, cryptography or consent requires rereading the RFCs that depend on it and fixing whatever became inconsistent. The known coupling is mapped in the "Neighbourhood" section below.
12. **Reference scenarios revalidated** (human judgement). The six scenarios of the [vision](vision.md) remain executable with what the specification says, with no magic step.
13. **Security and privacy reassessed** (human judgement). If the change creates new metadata visible to the host, it goes into the exhaustive list of [RFC 0006](../spec/crypto.md) §6 and into the [threat model](threat-model.md). New metadata that is not listed is a violation of the specification.
14. **Open question handled.** Every remaining doubt becomes an item under "Open questions" in the RFC itself. A doubt that is not written down is hidden debt.

## DoD for a change to an ADR

15. **The four sections exist:** Context, Decision, Alternatives considered, Consequences.
16. **Alternatives with a reason for rejection.** An alternative listed without why it was refused does not count: that is where the value of an ADR lies.
17. **An ADR is never edited to change your mind.** A revised decision becomes a new ADR with status `accepted`, and the old one moves to `superseded` pointing at its successor. The history is the product.
18. **Affected RFCs updated.** An ADR that changes a founding decision forces a review of the RFCs that implement it; leaving an RFC contradicting an ADR is the worst possible state.

## DoD for a new document

19. It goes into the README map, into the appropriate index, and into the site (automatically).
20. It declares at the top what it is: context, norm or research. A document in `docs/` is never normative; norms live in `rfc/`.
21. Research making a claim about the world (adoption, standard status, how a system works) cites a source with a URL. A claim without a source is an opinion and must be marked as such.

## The specification in English

The set in `spec/drafts/` is a translation, not a fork. If an EW-RFC changed, the corresponding draft changes with it, or the divergence is entered as a declared pending item. `spec/build_draft.py --check` validates that they all still compile, and the one-to-one mapping is in the table in `spec/README.md`.

The em dash rule applies there too: `make check` already sweeps `spec/` because it sweeps every markdown file in the repository.

## Neighbourhood (who breaks whom)

| If you touched | You must reread |
|---|---|
| RFC 0001 (envelope, signature) | 0004, 0005, 0006, 0007 |
| RFC 0002 (data model) | 0004, 0008, 0009, 0010 |
| RFC 0003 (identity) | 0005, 0006, 0007, 0010 |
| RFC 0004 (sync, blobs) | 0002, 0006, 0010 |
| RFC 0005 (federation) | 0001, 0003, 0007, 0009 |
| RFC 0006 (cryptography) | 0004, 0005, 0009, 0010, the threat model |
| RFC 0007 (consent) | 0002, 0005, 0008, 0009 |
| RFC 0008 (payloads) | 0002, 0007 |
| RFC 0009 (urgency) | 0001, 0002, 0006, 0007 |
| RFC 0010 (bridges) | 0002, 0004, 0006, 0007 |
| RFC 0011 (anti-abuse) | 0003, 0005, 0007, 0012, the threat model |
| RFC 0012 (discovery) | 0003, 0007, 0011, the threat model |
| RFC 0013 (compartments) | 0002, 0006, 0008, the threat model |
| RFC 0014 (autonomous actors) | 0002, 0003, 0007, 0009, 0013, the threat model |
| RFC 0015 (entries) | 0001, 0002, 0009, 0011, 0013, 0014 |
| RFC 0016 (compliance) | 0003, 0004, 0006, 0007, 0012, 0015, the threat model |
| Any ADR | the RFCs listed in the "Where" column of 02-requisitos.md |

## Language policy

Drafts in Brazilian Portuguese until the public opening. In phase 4 of the [roadmap](../ROADMAP.md), the suite is translated and English becomes canonical. Until then: write in Portuguese and keep the glossary with the English equivalents, so that the translation is mechanical rather than archaeological. The site is already bilingual.

## How to run it

```bash
make check
```

```bash
make site
```

```bash
make serve
```
