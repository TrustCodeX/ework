# Vision

## The world we imagine

A world where everyone lives busy, owns every kind of device and still nobody manages to keep their tasks and appointments organised the way they should be: with dependencies, importance, self-management, multiple contexts and different levels of criticality.

The central diagnosis of e-work: **most of an ordinary person's tasks are not born with them, they are born at third parties**. Companies, clinics, schools, building managements, notaries and governments generate obligations and commitments all the time, and deliver them through channels built for free text: email, WhatsApp, paper, text messages. The work of turning those messages into organised tasks (with a deadline, a reminder, a priority and a way to close them) is 100% manual, and it is exactly the work ordinary people do not know how, or do not manage, to do.

The thesis: if a common protocol exists with task semantics (deadline, acceptance, status, dependency, urgency, consent), the cost of getting organised drops to almost zero for the user, and the issuer gains a visibility it does not have today, in exactly the measure the user allows.

Email did this for messages. iCalendar and iTIP did it for corporate meetings. Nobody has done it for the real-life tasks of ordinary people. The space is empty, and the prior art research ([docs/01](prior-art.md)) confirms it is still empty in 2026.

## Personas

- **Cléia, 58, self-employed.** Pays seven bills a month, uses WhatsApp and her bank's app. She misses due dates because the bill arrives by email and disappears in the inbox. She has never configured a task app in her life and is not going to. For her, the task has to arrive ready, with a button to pay it.
- **Michel, developer and self-hoster.** Wants to run his own host in his homelab, with everything end-to-end encrypted, and to aggregate his personal, family and work boxes into a single client. He is the advanced user who validates the promise of federation.
- **Clínica Vida, 12 employees.** Confirms hundreds of appointments a month by phone and WhatsApp. It wants to send the appointment straight into the patient's box, with preparation instructions, and to receive a confirmation or a request to reschedule with no human involved.
- **Acme Energia, 2 million customers.** Issues a monthly invoice. It knows part of its late payment is pure forgetfulness. Brazil's direct debit scheme only covers those who signed up at their bank and locks presentation into the bank's app; Acme wants an open channel, with a delivery receipt and acceptance status.
- **Marcenaria Alfa plus three suppliers.** A fitted kitchen involves measuring, design, manufacturing, delivery and assembly, each stage at a different company, with real dependencies between them. Today the coordination is a WhatsApp group; the end customer has no view of the whole at all.

## Reference scenarios

These six scenarios are the success criterion for the specification. Every design decision has to be tested against them.

### S1. The monthly bill

1. Acme Energia (an issuer verified by the domain `acme-energia.com.br`) asks for consent exactly once: at signup, Cléia scans a QR code or accepts a request that landed, silently, in the requests tab of her app.
2. The consent granted is a scoped object: Acme may send tasks of the payment type, at most N per month, with a maximum urgency of "normal", and receives back only milestones (accepted, completed), not details.
3. Every month, when it issues the invoice, Acme's system sends a task offer: "Pay the energy bill, due 2026-08-10", with the typed payment payload (the payable line, the barcode, the copy-and-paste instant payment string, the amount, the late fee and interest) and the bill's PDF as a binary attachment.
4. Cléia's app accepts automatically (a rule she turned on for trusted issuers) or asks for one tap of confirmation. The task lands in her box with reminders derived from the due date.
5. The recurrence is recognised by the issuer's deduplication key: September's invoice updates the series rather than creating a duplicate.
6. Cléia pays through her bank's app (e-work carries the charge data and the deep link; settlement belongs to the bank or the instant payment scheme, outside the protocol) and marks it complete, or attaches the receipt.
7. Acme receives the "completed" status because Cléia's privacy policy releases milestones. Had she chosen "receipt only", Acme would know just that the offer was delivered.
8. If Cléia revokes consent, her server starts refusing Acme's offers at the edge, immediately.

### S2. The medical appointment

1. Clínica Vida sends a task offer with a typed scheduling payload: two time options, the address, the professional and preparation instructions (an eight-hour fast).
2. The patient accepts one time, or makes a counter proposal (COUNTER semantics inherited from iTIP), or declines.
3. Once a time is accepted, the task gains staggered reminders (the day before and two hours ahead) and the preparation instruction becomes a dependent subtask with a deadline of its own.
4. Rescheduling is an update to the same task by the issuer, which the patient accepts or counters. The negotiation history stays on the task.

### S3. The multi-supplier project (the fitted furniture "epic")

1. Marcenaria Alfa creates a project, "Cléia's kitchen", and invites the customer and the three suppliers. Every participant uses their own e-work provider; nobody needs an account in anybody's system.
2. The project is encrypted: each member has their own key, and the packet only opens for those inside. If an outsider intercepts it, it does not open. When the fitter joins at the end, an existing member re-shares the state to their key, an explicit act, and from then on they follow what concerns them.
3. Tasks have typed dependencies: "delivery" only becomes actionable when "manufacturing" finishes; "assembly" starts two days after "delivery" (a gap). Each task has an owner at a different company.
4. **Not everyone sees everything, and that is cryptographic, not visual.** The fitter sees nothing about payments; supplier Beta does not see the price charged by Gama. The project has compartments with distinct keys: the amount travels sealed inside the same delivery task everyone coordinates around, and assembly depends on a neutral milestone ("payment released") rather than on the financial task that produced it. Whoever is outside the compartment does not have the key, so the restriction holds even if the person uses a modified client.
5. When manufacturing runs late, the replanning propagates: clients recompute the actionable dates from the graph and everyone sees the same state, without a WhatsApp group.
6. Cléia, who owns the project, sees the whole. Each supplier sees the compartment they belong to, and nothing more.

### S4. The critical task

1. Michel's mother's controlled medication has to be taken at 8 a.m., every day. It is a recurring task with "critical" urgency and an escalation policy.
2. At 8 a.m. all her devices ring at maximum priority (piercing silent mode, with prior system permission). The notification demands an acknowledgement, which is different from completion.
3. With no acknowledgement within 15 minutes, the protocol escalates: it notifies Michel, who previously agreed to be an escalation contact. He calls his mother.
4. The acknowledgement pauses the escalation; completion closes the day's cycle. It is PagerDuty's state machine (trigger, acknowledge, resolve, with deduplication), domesticated for real life.

### S5. Self-management

1. The user's rules operate over the tasks: auto-accept offers from issuers with consent, schedule tasks that carry a duration estimate into the free spaces of the calendar, defer low priority tasks when the day fills up, archive completed ones.
2. In the default E2EE mode the rules run on the clients. In assisted mode (opt-in per collection) the server can execute them for you, even with all your devices switched off.
3. An assistant (human or AI) can receive a delegated credential with limited scope to operate a specific box. Delegation is a future extension, but the credential model is born ready for it.

### S6. Machine-to-machine work, with a human where one is needed

Not every client is a person, and that is a difference of nature, not an implementation detail.

1. Cléia's agent runs on her host with a narrow delegation: it may accept and schedule payment tasks up to 200 reais, only within the financial circle, for three months. It has a key of its own, declared as autonomous.
2. The 180-reais energy bill arrives. The agent accepts it, fits the payment into the calendar and completes it, without waking anyone. Cléia sees it later, and the record says clearly that it was her agent, not her.
3. A 1,200-reais charge arrives. It is above the autonomous band, so the task stops and waits for human approval. The agent cannot approve it even by lying: human approval is a signature from a key of the human class, and it has none. The restriction holds even if somebody swaps out the agent's software.
4. A stock system at one company talks to a system at another with no human at either end: machine-readable acceptance criteria, a structured and signed result, idempotency so that re-execution does not duplicate the effect. They are still tasks, with deadlines and dependencies, and the human owner sees everything whenever they care to look.
5. At furniture delivery, whoever executes is not whoever confirms: the carrier marks it delivered, and the task waits for Cléia's confirmation. She confirms, or disputes it and sends it back for execution. The word of whoever executed is no longer final.

## Design principles

1. **A protocol, not a product.** The value is in interoperability. Any client, any server, any issuer that implements the specification participates on equal footing.
2. **Consent before delivery.** No existing federated protocol has this as a protocol rule, and that is why they all turn into spam. In e-work, strangers reach at most a consent request in a silent quarantine.
3. **E2EE by default, honestly.** Content encrypted in a group; a blind server. The trade-offs (server-side search, hosted automation, escalation with dead devices) are resolved by an explicit assisted mode, never by silent weakening.
4. **Identity belongs to the person.** A root key owned by the user, a readable address as a verifiable alias, server migration with no loss of identity. Matrix's number one regret will not be ours.
5. **A minimal mandatory core, registered extensions.** XMPP died of optional extensions; WS-HumanTask died of complexity. The e-work core fits in a few pages and everything else is an extension with a registry.
6. **Reuse proven semantics.** JSCalendar for the data model, iTIP for negotiation, RFC 9253 for dependencies, MLS for group cryptography, PagerDuty for urgency, open finance for consent. Invent only where there is a real gap: consented federation of tasks.
7. **Bridges from day one.** A CalDAV gateway for existing clients, email as a presentation fallback, a simple webhook for issuers. Incremental adoption, no big bang.
8. **Binary where it matters, readable where it matters.** Attachments are raw binary blobs, addressed by hash, with no base64. Control envelopes are readable JSON (CBOR optional).
9. **A domain-agnostic core.** The protocol does not know what a bill, an appointment or a delivery is: it knows about tasks, offers, acceptance, deadlines, dependencies, attachments, actions, urgency and status. All domain knowledge lives in registrable typed payloads (EW-RFC 0008), which anyone can create without touching the core. Payment is only the first because it is the sharpest pain and the easiest to demonstrate; school enrolment, lab results, property inspection, a work order, document renewal and whatever else the world invents all come in through the same mechanism. If a core decision only makes sense for one domain, it is in the wrong place.

## Non-goals

- **Not an agile management tool for software teams.** Jira, Linear and the like remain better at that. e-work aims at the real life of ordinary people.
- **Not a calendar.** It integrates with calendars (and exports to them), it does not replace them.
- **Not a payment method.** It carries charge data and deep links; settlement belongs to the bank, the instant payment scheme, the card terminal.
- **No blockchain.** Decentralisation here means federated servers and user-held keys, not global consensus and not a token.
- **Not a social network.** No feed, no public discovery of people, no engagement metrics.

## Success measures for the specification phase

1. The six reference scenarios are executable end to end using only what the RFCs specify, with no "and here magic happens".
2. A developer can implement a minimal client (read the box, accept an offer, complete a task) in a few weeks reading only RFCs 0001 to 0004.
3. A reference host runs in a homelab: one binary, one database, one domain.
