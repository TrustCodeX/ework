# Landscape: what already exists (prior art)

Research carried out on 2026-08-04, on three fronts: formal standards (IETF and OASIS), decentralised protocols, and the real product ecosystem, including the Brazilian analogue that maps directly onto the bill scenario (the DDA direct debit scheme). Sources are linked on each item. The conclusion, up front: **every piece of e-work already exists, tested, somewhere; what exists nowhere is the combination, and in particular the federation of tasks with consent as a rule of the protocol.**

## 1. Formal standards

### iCalendar VTODO (RFC 5545) plus iTIP (RFC 5546) plus iMIP (RFC 6047)
Task and negotiation semantics have already been standardised: VTODO has STATUS, PERCENT-COMPLETE, DUE, PRIORITY; iTIP defines REQUEST (assign), REPLY with PARTSTAT (ACCEPTED, DECLINED, DELEGATED and so on), COUNTER and DECLINECOUNTER (counter proposal) and CANCEL, and it applies to tasks as well. Almost no client implements VTODO assignment, and the transport (email) has neither strong authentication nor consent: the calendar invitation became a spam vector. An important development: **draft-ietf-calext-ical-tasks** (December 2025, approved, in the RFC Editor queue) adds TASK-MODE, the STATUS values `PENDING` and `FAILED`, and ESTIMATED-DURATION, designed for task assignment. It is proof that the IETF itself sees the gap.
We reuse: the negotiation vocabulary (accept, decline, counter) and the new states. [RFC 5546](https://www.rfc-editor.org/rfc/rfc5546.html), [ical-tasks](https://datatracker.ietf.org/doc/draft-ietf-calext-ical-tasks/)

### CalDAV (RFC 4791)
Alive in the self-hosted world (Nextcloud, Tasks.org, Thunderbird, Radicale), abandoned by the giants (Apple Reminders left CalDAV in iOS 13; Google Tasks never joined). It proves that "a task as a syncable object in collections" works, and it is the obvious legacy bridge. But: verbose XML, a single account per server, no federation between organisations, and the server has to read everything (zero E2EE). [RFC 4791](https://datatracker.ietf.org/doc/html/rfc4791)

### RFC 9253 (iCalendar relationships)
The dependency vocabulary the furniture scenario needs already exists: RELTYPE `DEPENDS-ON`, `FINISHTOSTART` (and the other three), the `GAP` parameter, the `LINK` property. Published in 2022, with low adoption. It is vocabulary only: there is no protocol for propagating state across the graph between organisations. [RFC 9253](https://datatracker.ietf.org/doc/html/rfc9253)

### JMAP (RFC 8620), JSCalendar (RFC 8984) and JMAP Tasks
The state of the art in client-server synchronisation: JSON, batched calls, exact state strings per type, `/changes` for deltas, integrated push, blobs. JSCalendar defines `Task` in JSON with progress (including `failed`), percentComplete, estimatedDuration and per-participant progress. `draft-ietf-jmap-tasks` expired in 2023 but the working group keeps a milestone for 2027 (parked, not abandoned); JMAP for Calendars is in the RFC Editor queue in 2026; Fastmail with Cyrus, and Stalwart, implement the JMAP family. We reuse: JSCalendar's Task object as the basis of the data model, and JMAP's sync design as the basis of RFC 0004. The limit: JMAP is client-server, not federation, and it assumes a server reading plaintext. [RFC 8620](https://datatracker.ietf.org/doc/html/rfc8620), [RFC 8984](https://www.rfc-editor.org/rfc/rfc8984), [jmap-tasks](https://datatracker.ietf.org/doc/draft-ietf-jmap-tasks/), [Stalwart](https://stalw.art/blog/jmap-collaboration/)

### Structured Email (the IETF sml working group)
It tries to embed a machine-readable version (schema.org) in invoice and booking emails, with a "trust" draft covering who may send actionable content. No RFC published as of August 2026; implementations are incipient (Nextcloud Mail, Roundcube). It is the bill use case, but one-way: no lifecycle, no status coming back, no sync. It serves as a fallback transport and as validation of the problem. [sml](https://datatracker.ietf.org/doc/draft-ietf-sml-structured-email/)

### OASIS WS-HumanTask and BPEL4People
The SOAP standard (2007 to 2010) for human tasks between systems: a rich state machine, generic roles (initiator, potential owners, actual owner and others), deadlines with escalation. Killed by WS-* complexity and enterprise coupling, with no consumer story. We take the lifecycle and the roles; we ship them as a minimal core plus extensions, the opposite of their strategy. [spec](https://docs.oasis-open.org/bpel4people/ws-humantask-1.1-spec-cs-01.pdf)

### SyncML and OMA DS
Dead. The negative lesson is preserved: vague anchors produced slow syncs and duplicates; a chatty protocol; "data agnostic" turned into broken interoperability. JMAP's state strings are the historical correction, and RFC 0004 inherits that.

### Web Push (RFC 8030, 8291, 8292) and MLS (RFC 9420)
Ready-made blocks: push with a payload encrypted end to end all the way to the user agent, universal in browsers; MLS provides asynchronous group E2EE with forward secrecy and post-compromise security, adopted by RCS (GSMA, 2025) with a Google and Apple rollout in 2026, used by Wire, Webex and Discord. MLS's separation between Delivery Service and Authentication Service fits federation of blind hosts. MLS gives confidentiality, not semantics: ordering and state merging remain our problem. [RFC 8030](https://datatracker.ietf.org/doc/html/rfc8030), [MLS in RCS](https://www.ietf.org/blog/rcs-adopts-mls/)

## 2. Decentralised protocols

### ActivityPub and ForgeFed
Federation by inbox and outbox with a signed POST; the community is migrating from the HTTP Signatures draft to RFC 9421, and adopting a signature on the object itself (FEP-8b32), which is the right form: self-authenticating objects survive relaying and migration. Weaknesses: no E2EE, identity tied to the instance, moderation by blocklist (the spam wave of February 2024 proved it). ForgeFed federates issues between git forges and is the closest real example of "federated tasks": it took around seven years for the basics (Forgejo federated "stars" in 2025). The lesson: keep the scope lean or nothing moves. [forgefed.org](https://forgefed.org/)

### Matrix
The most tested multi-device package in the world for "ordinary people": cross-signing, device verification, key backup with a recovery key, invisible crypto in Matrix 2.0. MLS has not been adopted there yet (Megolm remains the standard; "arewemlsyet" says not yet). The identity `@user:server` tied to the homeserver is the project's number one public regret (the cryptographic identity MSCs have not landed). The replicated room DAG is far too heavy for tasks. Policy servers (2025) as a pre-delivery filter are a good anti-abuse pattern. [Matrix 2.0](https://matrix.org/blog/2024/10/29/matrix-2.0-is-here/), [MSC4080](https://github.com/matrix-org/matrix-spec-proposals/pull/4080)

### XMPP
A double lesson: extensibility without a mandatory core kills interoperability (every client with a different subset of XEPs); and presence subscription (mutual approval before seeing anything) is a consent-first primitive that worked. [analysis](https://blog.samwhited.com/2019/02/whats-wrong-with-xmpp/)

### Nostr
Identity equals key; NIP-05 maps `name@domain` to the key through `/.well-known/`, with verification in a layer separate from identity. Maximum portability (interchangeable relays), but a raw key with neither rotation nor recovery is no good for ordinary people. Anti-spam by cost (proof of work, paid relays) and a web of trust. Tasks over Nostr: only experiments, with no traction. [NIP-05](https://nips.nostr.com/5)

### AT Protocol
Best-in-class identity: a stable DID plus a mutable DNS handle; the DID document points at keys, recovery keys and the current server; account migration moves the signed repository. In 2026 the PLC directory becomes an independent association (neutral governance matters). No E2EE by design (the data is public), so it cannot serve as a direct base. [account migration](https://atproto.com/guides/account-migration), [PLC directory](https://docs.bsky.app/blog/plc-directory-org)

### Solid and W3C DIDs
Solid never took off (RDF complexity, no killer app, passive pods): a protocol has to be born out of the concrete use case, not out of the data architecture. DIDs: steady institutional adoption (EUDI Wallet, Entra, did:plc); the pragmatic path for us is to be DID-compatible (did:key for people, did:web for organisations) without requiring the entire ecosystem. [DID 1.1](https://www.w3.org/TR/did-1.1/)

### Anti-abuse in open federation
SPF, DKIM and DMARC were a retrofit twenty years late, and they authenticate the sender without creating consent; the default of "accept everything from everyone" turned the defence into the opaque reputation of an oligopoly. Mastodon (2024) and Matrix (2025) repeated the pattern and suffered the same waves of spam. Pieces of consent-first do exist (Matrix knocking, XMPP subscription, Bluesky's DM allow-list), but **no federated protocol has "content only enters after consent" as a protocol default**. That is precisely e-work's space.

## 3. The product ecosystem

### Task synchronisers
TaskChampion (Taskwarrior 3) synchronises through an operation log encrypted on the client: a linear chain of versions, rebase on conflict, snapshots; the server only ever sees bytes. A simple, auditable model, with E2EE for free across one person's devices; it is the basis of our E2EE sync. Todoist, TickTick, Google Tasks and Microsoft To Do: four proprietary APIs, no common standard, sharing always within the platform; none of them lets an external company create a task in the user's account without an ad-hoc integration. Vikunja (self-hosted): a good API plus CalDAV, with federation non-existent and off the roadmap. The gap is confirmed from every side. [TaskChampion sync](https://gothenburgbitfactory.org/taskchampion/sync-protocol.html), [Todoist API](https://developer.todoist.com/api/v1/)

### Structured actions in email
Gmail markup (schema.org), AMP for Email and Microsoft Actionable Messages (Adaptive Cards) prove the demand for "a verified issuer plus an action in the message", and they prove the anti-pattern: registration and approval controlled by two platforms, with no standardised status coming back. Issuer verification has to be open (domain plus key), the way SPF and DKIM were. [Gmail markup](https://developers.google.com/workspace/gmail/markup/registering-with-google), [Actionable Messages](https://learn.microsoft.com/en-us/outlook/actionable-messages/adaptive-card)

## 4. The Brazilian precedent: DDA, Open Finance, Pix

### DDA (authorised direct debit)
The direct analogue of scenario S1, running since 2009: the payer signs up at their bank ("elected payer") and every registered bill against their tax ID starts being presented electronically in the bank's app. Because presentation comes from the registered base, the fake-bill scam dies by construction. Its limitations are our pitch: bank-by-bank enrolment with no single view, presentation locked into the bank's channel (no open API for the user's task app), no semantics for a refusal returned to the issuer, and whoever does not open the app misses the bill. e-work is, among other things, "DDA as an open protocol, with the consent belonging to the user instead of the bank". [FEBRABAN](https://portal.febraban.org.br/pagina/3051/1088/pt-br/dda)

### Open Finance Brasil
The most mature consent model in the ecosystem: consent as an API resource with granular scope, a purpose, a term, revocation at any moment from either end, and the golden rule: **revoking must be as simple as granting**. 62 million active consents prove the model scales. The Consent object of RFC 0007 is traced from here. [journey](https://openfinancebrasil.atlassian.net/wiki/spaces/OF/pages/1128890377/Jornada+Otimizada)

### Pix
What a payment task carries: the EMV BR Code (static, or dynamic with a JWS payload), and the fields of a charge with a due date (CobV): txid, debtor, amount, due date, late fee, interest, discount. Pix Automático (live since 2025-06-16) defines the enrolment journeys for recurrence, including the composite QR code ("immediate charge plus recurrence"), which is the UX template for our recurring offer. [central bank guide](https://liftchallenge.bcb.gov.br/content/estabilidadefinanceira/pix/automatico/guia_pix_automatico.pdf)

## 5. Urgency and notification delivery

The PagerDuty Events API: `event_action` trigger, acknowledge and resolve, plus an idempotent `dedup_key`; escalation policies with ordered levels, a timeout without acknowledgement and limited repetition; the acknowledgement pauses the escalation without resolving. That is the state machine of RFC 0009, simplified for private individuals (Opsgenie dies in 2027; PagerDuty is the reference). ntfy and UnifiedPush: open, self-hostable push delivery with five priorities and action buttons; the protocol must not assume FCM or APNs. [Events API v2](https://developer.pagerduty.com/docs/events-api-v2/trigger-events/), [UnifiedPush](https://unifiedpush.org/)

## 6. Consolidated lessons (what the RFCs inherit)

1. **Do not invent a data model:** a profile of JSCalendar Task plus RFC 9253 plus calext-ical-tasks covers around 90% of the scenarios' semantics. (RFC 0002)
2. **Sync is a solved problem: copy JMAP** (state strings, changes, batching, push) and, for E2EE, TaskChampion's encrypted operation log. (RFC 0004)
3. **The real gap is federation with consent.** Innovate here and only here: a consent handshake before any full delivery, a revocable credential at the edge. (RFC 0005, 0007)
4. **Identity: a stable DID plus a DNS handle**, recovery keys in the identity document, bidirectional migration. Do not repeat Matrix's `@user:server`, nor Nostr's raw key. (RFC 0003)
5. **Multi-device E2EE: copy Matrix's package** (cross-signing, verification, key backup) over MLS groups; a blind server routing ciphertext. (RFC 0006)
6. **Self-authenticating objects:** a signature on the object as well as on the transport, for relaying, auditing, migration and "this charge really did come from bank X". (RFC 0001, 0005)
7. **Open Finance style consent over demand proven by DDA:** an object with scope, validity and symmetry between granting and revoking. (RFC 0007)
8. **PagerDuty style urgency:** acknowledgement distinct from resolution, a dedup key, escalation with a timeout and the consent of whoever is escalated to. (RFC 0009)
9. **A small mandatory core plus registered extensions plus a use case that hurts on day one** (the bill). XMPP, WS-HumanTask and Solid show the three ways of dying by ignoring this. (RFC 0000, 0001)
10. **Bridges:** CalDAV with VTODO, and email, as gateways assuming the real lowest common denominator; agnostic push; open issuer verification, with no gatekeeper. (RFC 0010)
