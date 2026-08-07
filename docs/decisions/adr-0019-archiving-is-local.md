# ADR-0019: Archiving a project is yours, not the project's

**Status:** accepted (2026-08-06)

## Context

The handoff asks for the project list to have "In progress, Archived, All" tabs, and for archiving to have a real effect: the project's tasks leave Today, My tasks, Calendar and Board, the counters drop, nothing is deleted, and reactivating brings everything back.

Archiving exists in no RFC, and the question that decides the design is not an interface one: **is archiving mine or the project's?**

An e-work project has members on different hosts, each with their own inbox. The two readings are incompatible:

1. **Mine.** It disappeared from my projections. The others keep seeing it and keep receiving. It is a display preference, it does not travel, and it needs no field in the shared model.
2. **The project's.** Someone archives and nobody receives notices any more. It becomes project state, it travels in the envelope, and it requires a rule about who may archive, what happens to a task in flight at the moment of archiving, and what a member does if they disagree.

The second reading drags federated project governance along with it, which is a subject the protocol does not yet have. And it drags an expectation problem: if archiving belongs to the project and the interface does not make that crystal clear, someone will archive thinking they are tidying their own screen and will silence the project for everyone else.

## Decision

**Archiving is a local preference of each member. It is not a project field, it does not travel, and it does not change what the others see.**

1. **Per-member state, stored at the host of whoever archived**, alongside the membership record and not the project. It is the same nature as "dark theme": it concerns how that person sees, and it goes away when she changes her mind.

2. **The effect is on projections, and it is complete.** Tasks of an archived project leave Today, My tasks, Calendar and Board, and the counters. They do not disappear: they remain readable, exportable and reachable through the "Archived" tab. Archiving is not deleting, and the dialog says so.

3. **The task keeps arriving.** The issuer is not told, consent does not change, and the history keeps being written. Whoever reactivates finds the project in its real state, not in a photograph of the day she archived it.

4. **The interface must not suggest a collective effect.** The dialog text says, in those words, that it disappears from **your** view and that the other members keep seeing it. Without that sentence the ambiguity comes straight back in through the front door.

## Alternatives considered

**Project state, decided by the owner.** Rejected for now. It requires answering who may archive, what happens to work in flight, and how a member disagrees. None of those questions has an answer today, and none of them needs answering for a person to be able to get an ended project out of her way. It remains the right design for "closing a project", which is a different thing from archiving and deserves its own ADR when there is a federated project in real use.

**Project state, propagated by a signed entry.** This is the correct way to do the alternative above, if it is done: an effect on third parties is a signed entry under EW-RFC 0015, never a flag changed in silence. Rejected along with it, for the same reason, and noted so it is not lost.

**Not having archiving, and using a filter.** Rejected: a filter is a choice remade on every visit, and the problem is precisely that an ended project keeps taking up attention every day. The person wants to decide once.

**Keeping the preference only in the browser.** Rejected: a person has more than one device, and a project archived on the phone would come back on the computer. A preference that does not follow the identity does not solve the problem that motivated the request.

## Consequences

- EW-RFC 0013 gains a paragraph saying archiving is local, and EW-RFC 0002 §10 gains the exception in the projections. Issue #36 stops being blocked.
- The host gains `Project/archive` and `Project/unarchive`, which write to the membership record, and `Project/list` starts returning the state of whoever is asking.
- **None of this travels between hosts**, so the decision is reversible without migration: if archiving one day becomes project state, the local field still makes sense as an overriding preference.
- Filtering the projections now depends on knowing which project each task belongs to, which already exists in `ework:projectUid`.
