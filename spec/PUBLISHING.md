# Publishing this officially: what exists, what it costs, what is missing

Three paths, and they do not compete: they serve different audiences and can run
in parallel. What follows is the real state of each, not an optimistic plan.

## 1. An Internet-Draft at the IETF

**This is the right path for a protocol.** HTTP, SMTP, IMAP and TCP are IETF
RFCs, and that is where a protocol specification receives the kind of scrutiny
that matters.

What many people do not know: **publishing an Internet-Draft has no gatekeeper.**
Anyone creates a datatracker account and submits. The document becomes public,
receives a number, and expires in six months if it is not renewed. That alone
gives citability and a stable address.

What does have a gatekeeper is becoming an RFC, and there are two routes:

- **A working group.** The normal path. It requires an existing WG to adopt the
  work, or a new one to form (a BoF at an IETF meeting). It is the slowest and
  it produces the best document.
- **The Independent Submission Stream.** It goes through the Independent
  Submissions Editor, needs no WG, and produces an actual RFC, marked as not
  being IETF consensus. It is the realistic route for new work with no community
  formed around it yet.

**Current state.** The drafts in `spec/` already come out in the official format
(`build_draft.py` generates `.txt` and `.html` through xml2rfc). What is missing:

- [ ] A datatracker account (<https://datatracker.ietf.org/accounts/create/>)
- [ ] Deciding which documents to submit first. Submitting sixteen at once is
      not how it works. The pair `core` plus `consent` is the minimum that
      stands on its own and carries the new contribution.
- [ ] Choosing which WG to notify. Candidates: `calext` (calendar extensions,
      home of JSCalendar and RFC 9253) and `dispatch` (for work that does not
      have a home yet).
- [ ] IANA registrations: `.well-known/ework` (RFC 8615), the service name
      `_ework._tcp` (RFC 6335) and the media type. The sections are already
      written in the drafts; the request itself only happens at publication.

**Cost:** zero in money. Submitting is free. Attending an IETF meeting is not
required in order to submit, and the mailing lists are open.

**Realistic timeline:** the draft online in days. An RFC through the Independent
Stream, if that is the route, takes one to two years.

## 2. A preprint with a DOI

This serves immediate citability and establishes priority, which matters for
work whose contribution is conceptual.

- **arXiv** (`cs.CR` or `cs.DC`). Free. A first submission in a category usually
  requires an endorsement from someone who has published there, which is the
  only real friction.
- **Zenodo.** No gatekeeper at all, a DOI immediately, and it integrates with a
  GitHub repository. It is the shortest path to a stable identifier, and it does
  not prevent submitting to arXiv afterwards.

**Recommendation:** Zenodo first, because it is immediate and it unblocks citing
the work. arXiv next, if an endorser appears.

## 3. An academic venue

The paper in `paper/` is written for this. The defensible contribution is not
"another task protocol": it is **consent before delivery as a protocol
primitive**, with the per-relationship address and oracle-free retirement as the
mechanisms that make it operational, plus the empirical findings from an
implementation running on two domains.

In order of fit:

| Venue | Fit | Note |
|---|---|---|
| PETS / PoPETs | high | Privacy is the axis; it accepts systems work with analysis. Quarterly cycles, which is rare and welcome. |
| USENIX Security | high | Needs a stronger evaluation than the current one. |
| NDSS | high | Likewise. |
| ACM CoNEXT | medium | Networking focus; the federation angle fits. |
| A workshop (FOCI, ConPro) | a good way in | Less demanding, faster feedback, good for a first round. |

**What the paper still needs before it can be submitted:**

- [ ] **Quantitative evaluation.** There is now a benchmark harness measuring the
      absence of a timing oracle, what an attacker achieves, and what consent
      costs a legitimate issuer. What is still missing is the comparison against
      email with a real filter and against ActivityPub in the same scenario, and
      federated latency over a real network rather than localhost.
- [ ] **A formal threat model.** It exists as prose in
      [the threat model](../docs/threat-model.md); it needs to become the format
      the community expects, with a defined adversary and stated properties.
- [ ] **Rigorous related work.** [The landscape](../docs/prior-art.md) has the
      sources; what is missing is explicit positioning against ActivityPub,
      Matrix, JMAP, Solid and the capability systems.
- [ ] **An English review by a native speaker.** The drafts were written here;
      external review is cheap and avoids rejection on form.

## The order I would follow

1. **Zenodo now.** A DOI for the specification as a whole. Cost: one afternoon.
2. **The `core` and `consent` Internet-Drafts on the datatracker.** Cost: one
   afternoon, once the account exists. This makes the work citable and visible
   to people who work on protocols.
3. **Notify the lists.** `calext@ietf.org` and `dispatch@ietf.org`, pointing at
   the drafts and asking for criticism. This is where you find out whether there
   is a community.
4. **A paper for a workshop**, with the quantitative evaluation done. Fast
   feedback, and the criticism improves the PETS submission.
5. **PETS**, with whatever came back from the workshop incorporated.

## What is not a prerequisite

Worth saying, because this is where projects like this get stuck:

- **You do not need a mature implementation.** There is one, it runs on two
  domains, and that is already more than most drafts have on the day they are
  submitted.
- **You do not need adoption.** A draft is not a de facto standard; it is a
  proposal.
- **You do not need a legal entity**, nor sponsorship, nor an in-person meeting.
- **It does not need to be finished.** A draft is a draft by definition, and the
  `-00` exists in order to be criticised.

What you do need is the document in English in the right format, and three of
those already exist.
