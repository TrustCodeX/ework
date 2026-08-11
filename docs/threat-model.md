# Threat model

Initial draft (2026-08-04), revised on 2026-08-08 after the adversarial review recorded in [08-revisao-de-seguranca.md](security-review.md). It must be reviewed on every change to RFCs 0005, 0006 and 0007.

## 1. Assets to protect

| Asset | Examples | Sensitivity |
|---|---|---|
| Task content | debts, health (appointments, medication), the household routine | Extremely high: it is a portrait of the person's life |
| The relationship graph | who sends tasks to whom (banks, clinics, lawyers) | High: metadata that gives away context |
| Keys | root key, device keys, group keys | Critical |
| Availability of critical reminders | medication, a court deadline | High: failure causes physical or legal harm |
| Consents | who is allowed to reach the user | Medium: manipulation opens the door to fraud |

## 2. Adversaries and scenarios

### A1. A curious or compromised host
The provider (or an intruder inside it) tries to read content or map out the user's life.
Mitigations: E2EE by default (RFC 0006); content and attachments are ciphertext; status sent to issuers is encrypted in the relationship group. Residual: routing metadata (see section 3).

### A2. A malicious issuer or spammer
A company (or a fraudster impersonating one) tries to flood users with offers or fake charges.
Mitigations: consent-first (RFC 0007): a stranger reaches only a consent request in silent quarantine; issuer verification by domain plus key; a revocable send capability taking effect at the edge (RFC 0005); rate limiting by consent scope; critical urgency requiring an explicit scope.
Residual: typosquatting of a lookalike domain (`acme-energla.com.br`). Mandatory UX countermeasures: always show the verified domain, mark new issuers, and pluggable reputation directories (outside the core).

### A3. The fake bill scam
The Brazilian variant of A2, today a billion-real industry.
Structural mitigation: in e-work a charge only arrives if (a) the issuer is verified by domain, (b) prior consent exists, and (c) the object comes signed by the issuer (self-authenticating, RFC 0001). The classic scam (a tampered bill by email) dies by construction, as with DDA, but without locking the user into a bank's app. The UI MUST show the payee of the payment payload and warn if it differs from the issuer.

### A4. A network attacker
Interception or tampering in transit.
Mitigations: TLS mandatory on every hop; transport signature between hosts (RFC 9421); the author's signature on the object; E2EE of the content. A packet intercepted by someone outside the group does not open, which is the literal requirement from the vision.

### A5. A lost or stolen device
Mitigations: device keys revocable by the root identity (RFC 0003); revocation removes the device from the MLS groups (a new epoch); post-compromise security limits the accessible past; local biometrics or PIN are the client's responsibility.
Residual: whatever the device had already decrypted locally before revocation.

### A6. A member removed from a project
A former supplier tries to keep reading the project.
Mitigation: an MLS Remove creates a new epoch; they open nothing from that point on. What they saw while a member, they saw (knowledge cannot be revoked). RFC 0013 §7 now requires the removal to be executed as a Remove in the group, forbids presenting the departure as done before the commit, and mandates treating the tree as authoritative when it diverges from the `Project` list: taking the name off the list was the natural action in the interface, and the only one that took away nobody's key.

### A7. Total loss of devices
The user loses everything and does not have the recovery kit.
The honest position: E2EE content without a kit and without a key backup is unrecoverable, by design. UX mitigations: a recovery kit mandatory at account creation; a key backup encrypted with the recovery key and hosted on the host (RFC 0006); optional custody (trading trust for convenience, which has to be an informed choice).

### A8. A coerced host (court order, government)
Mitigations: the host can only hand over what it sees (section 3). In assisted mode the surface grows: the decision to downgrade a collection belongs to the user and the UI MUST make the consequence explicit.

### A9. An address leaked, sold or scraped
A legitimate issuer suffers a breach, sells its database, or a collector scrapes addresses from somewhere.
Mitigations: an address per relationship (RFC 0003 §6) makes the leak attributable (only one issuer knew that address) and the cut surgical; the credential bound to the address (RFC 0007 §4) means a scraped address delivers nothing; `acceptsNewRequests: false` closes off even the consent request.
Residual: the leak of the data itself (name, tax ID, amount owed) already happened at the issuer and no protocol undoes that.

### A10. A legitimate issuer that turns into marketing
The company with consent to bill starts sending promotions, or begins using high urgency to force attention.
Mitigations: a purpose declared per offer within the scope (RFC 0007 §2); the operational test of what is not a task (RFC 0011 §6); rate and urgency limits at the edge; silent retirement as an answer that requires no negotiation. The economics invert: insisting costs the whole relationship, with no warning and no appeal.
Residual: the protocol does not detect marketing semantically, and does not pretend to.

### A11. Rotation hijacking
An attacker tries to redirect someone's address by publishing a forged rotation.
Mitigations: replacement requires a signature from the old key or from the root, never from the suspect key itself (RFC 0003 §7.1); a challenge window on root rotation. An issuer that accepts a rotation without validating the proof of continuity is non-conforming and becomes a billing fraud vector.

### A12. Address list validation
A collector probes addresses to discover which ones exist, which is the product the spam industry buys.
**The host as a network proxy.** Any field where the client says "go and fetch this" or "notify over there" lends the client the host's network position, which usually reaches things the client cannot: an internal service, the cloud metadata endpoint, the LAN gateway. That is classic SSRF, and in EWP it shows up in the gateway webhook (RFC 0010 §3) and in any fetch of a client-supplied URL. Mitigation: refuse loopback, private ranges, link-local, CGNAT and reserved names before making any request. Found in the reference implementation while reviewing the gateway.

Mitigation: absence of an oracle (RFC 0003 §7.4, RFC 0011 §4.2). An identical response in code, message and timing for an address that never existed, one that was retired, one frozen for deletion and one revoked. Implementations MUST test equality of timing, not only of code, and MUST cover every public endpoint: the reference implementation leaked exactly here, with delivery refusing while the handle proof went on publishing the document. One honest endpoint and one careless one add up to a whole oracle.

### A13. Directory enumeration
An attacker sweeps the phone number space to build a list of who has an account, which is the raw material of targeted fraud.
Structural mitigation: no lookup returns an address (RFC 0012 §3). Enumerating produces only consent requests thrown into the void, with a quota, a verified requester identity and a record visible to the target. Equality of response includes latency, and implementations MUST test that. The handle proof no longer returns the `did:key` to whoever presents only the name (RFC 0003 §3.1): it answers with a confirmation to whoever already knows the target, because returning the key made it a public oracle of account existence, and every hit handed over, through the identity document, all of the person's handles and hosts.
Residual: the attacker learns, at most, that they spent their quota. If the target has `allow: "anyone"` switched on, they receive a request in quarantine, which is noise, not exposure.

### A14. A compromised directory
Whoever operates the directory is attacked, or is malicious.
Mitigations: bindings stored as an HMAC pointing to a routing token encrypted for the user's host, so a dump reveals neither the attribute nor the destination; splitting across several directories; the directory leaves the path after first contact, and therefore does not accumulate the graph of who talks to whom.
Residual, stated without decoration: while operating, with the HMAC key in hand, the directory can test guesses over a small space and observe attempts. This is the part of the system where the operator's choice matters, and that is why it demands written governance and auditing.

### A15. Hijacking through a recycled number
The person's phone number is deactivated, returns to the carrier's pool and a third party receives it.
Mitigations: attribute attestations expire and require reverification; a phone binding MUST NOT, on its own, authorise account recovery, which continues to depend on the recovery kit (RFC 0003 §8). It is the mistake that phone-anchored protocols make by construction.

### A16. Coercion through identity binding
A company or a government body demands that the person switch on discovery, or bind an identifier, as a condition of service.
Mitigations: conditioning service on excessive scope is a compliance violation (RFC 0011 §5.6); no attribute is required in order to create or use an identity (RF-56); and the strongest structural mitigation is the absence of the most valuable target, since civil identifiers are out of scope (RFC 0012 §6, ADR-0010): you cannot coerce someone into switching on a binding the protocol does not implement. A public body participates like any other issuer. If a jurisdiction creates a legal obligation to receive by civil registry, that is an explicit national profile, with the political cost in plain sight, never an exception hidden in the core.

### A17. Leaking through the wrong compartment at authoring time
Someone creates the task with the amount in the clear in the broad compartment, and the leak happens at the instant of writing.
Mitigations: normative default classification per payload type (RFC 0008 §6, RFC 0013 §5), with closing automatic and opening an explicit and recorded act; the conformance suite tests that a payment payload in a multi-member project is born sealed.
Residual: nothing is retroactive. Reclassifying protects the future, and the interface is obliged to say so before the person believes they have fixed the leak.

### A18. Inference from the graph and from sealing metadata
A member deduces the private content from the structure: the existence of a sealed section, its size, a compartment's name, the moment a dependency unblocked.
Mitigations: a public milestone as a proxy, revealing only what needs revealing (RFC 0013 §6); a recommendation that compartment names not describe content; an honest list of what remains visible (RFC 0013 §8).
Residual: whoever watches cadence and size infers activity. Padding and temporal batching remain under study, with no promise for v0.1.

### A19. A legitimate member who passes on what they see
Whoever is inside the compartment copies and forwards.
No cryptographic solution, and it is the limit of every sharing system. That is why the correct granularity is "whoever needs to" and not "whoever is trusted": the mechanism reduces the surface, not human nature.

### A20. A compromised agent
Whoever controls the agent starts acting on the person's behalf.
Mitigations: the delegation's scope (circles, actions, payload types, maximum amount, rate, validity), the `requiresHumanAbove` band forcing a human above a value, immediate revocation with a new epoch, and the impossibility of the agent producing human approval because it holds no key of that class (RFC 0014 §2, §4). No implicit subdelegation, so the chain of responsibility does not dissolve.
Residual: harm within the delegated scope does happen. A tight scope is the only real defence, and the interface has to help the person keep it tight.

### A21. The empty human stamp
The flow asks for confirmation so many times that the person approves without reading, and the protection degrades without any technical control noticing.
Mitigations: there is no cryptography against inattention. Only design: prominence proportional to risk, a ban on "approve all", and frugality with confirmation prompts so that each one means something (RFC 0014 §4). It is the likeliest threat in this section, and the easiest to underestimate.

### A22. An actor lying about its class
An identity declares as human a key that is in fact inside a robot.
Mitigation: receivers MUST resolve the key in the identity document and use exclusively the class signed by the root, and a divergence from the class declared in the entry renders the entry without effect (RFC 0014 §4, RFC 0015 §1). Before that the distinction was declarative: the agent wrote `actorClass: "human"` in its own entry, signed it with its own key, and the only written rule said to display the field. It remains true that the root can lie when declaring a key's class, and then the lie is attributable. Human presence attestation in an enclave remains a possible extension for high risk (RFC 0014, open question 1).

### A23. A comment to the wrong audience
The person writes "I will pay when my salary lands" thinking it is a private note, and sends it to the issuer.
A purely interface mitigation, because the protocol has no way of guessing intent: display the destination with the circle's name before sending (RFC 0015 §2). It is the likeliest leak in the whole system, and it has no cryptographic defence.

### A24. A forged or pruned history
Two variants with the same purpose, winning an argument later: inserting into the record a "so-and-so changed the deadline" that never happened, or deleting one's own inconvenient action.
Mitigations: activity is derived locally from the signed operations and never travels as an assertion (RFC 0015 §7), so there is no channel through which the lie could enter; an entry with an action is immutable and cannot be removed or edited by anyone, including a project owner or a host administrator (RFC 0015 §10); and the hash chaining of `after` makes any pruning, editing or withholding detectable by mechanical verification of the chain (RFC 0015 §6). An audit trail the audited party can rewrite is not an audit trail.
Residual: whoever controls a client can choose not to display what they received, but cannot make the other participants forget, because each of them holds the signed copy.

### A25. Comments as a parallel abuse channel
The issuer uses comments to send marketing, working around the task's limits.
Mitigations: a comment creates no delivery right, counts against the same rate limit, has to fall within the granted purpose, and a mention does not raise urgency (RFC 0011 §3.4b, RFC 0015 §8). The user's answer remains the most effective one: retire that relationship's address.

### A26. Denial of service against a critical reminder
An adversary (or plain infrastructure failure) prevents the medication reminder.
Mitigations: channel redundancy in escalation (push plus the device's local sound plus a human contact); client-side timers do not depend on the host; RFC 0009 requires critical alarms to fire from local scheduling, not from push.

### A27. Rollback of the identity document
A host serves an old document in which a revoked device still appears, and an out-of-date peer accepts it.
Mitigations: a periodic freshness proof signed by a current device, and gossip of (id, seq) between peers inside the existing MLS groups (RFC 0003 §2); whoever learns a higher seq refuses earlier documents from then on.
Residual: whoever shares no group at all with the identity remains deceivable until the freshness proof expires. Key transparency is the recorded evolution for closing that remainder. Since the review, every key in the document carries `validFrom` and `validUntil` and revocation fills in the end instead of deleting the entry (RFC 0003 §2), which closes both sides of the problem: an old legitimate signature remains verifiable, and a revoked key stops being accepted by whoever only holds the current document.

### A28. Harvest now, decrypt later
An adversary records ciphertext today to decrypt it once there is a useful quantum computer.
Mitigation: declared cryptographic agility and a hybrid ciphersuite (ML-KEM with X25519) as the phase 2 target (RFC 0006 §8). The e-work corpus holds years of financial and health data, so this threat weighs more here than in ephemeral messaging, and that is why the KEM takes priority over the signatures.

### A29. Coerced deletion, or deletion with a stolen kit
Someone with the recovery kit (or coercing the account holder) requests deletion of the account in order to destroy evidence or cause harm.
Mitigations: a grace window with the account frozen, a notice on every device and cancellation from any verified device (RFC 0016 §5); the window CANNOT be shortened remotely.

### A30. A legitimate participant signing another's transition
Inside a group they belong to by right, a participant signs another's action: the issuer signs `accept` on its own offer and produces a signed record that the charge was accepted, the supplier signs `complete` to unblock its own delivery, someone signs `reassign`, `cancel` or `update` on somebody else's task.
Mitigation: authority per action (RFC 0015 §2.2, ADR-0021). The peer resolves the author in the `participants` and checks the role before applying the effect; an action without authority becomes an entry without effect, which stays written and does not count. Found in the review of 2026-08-08, and it is the finding that changed the specification the most: the flaw was not in any sentence, it was in the column the action table did not have.
Residual: whoever holds the role legitimately and acts in bad faith goes on acting. The mechanism reduces who can, never the intent.

### A31. The host in the middle of the relationship group
The host that routes the contact address presents its own key to the issuer as the user's, and sits in the middle of the relationship through which invoices, bills and exam results pass.
Mitigation: the contact key travels in the `consent.grant`, bound by signature to that consent, and the counterparty MUST refuse a group credential that does not match it (RFC 0007 §1, RFC 0006 §5). Before, the contact key had no validation chain at all, and the only thing attesting who was on the other side was the host that delivered it, which is exactly the party one wants to exclude.

### A32. Hijacking a domain's federated entry point
Whoever controls the DNS publishes the SRV pointing at their own host, serves the `.well-known` with a legitimate certificate for their own name and declares someone else's `addressDomain`. They forge nobody's signature, because what they want is to receive: they become the destination, read the `consent.request` objects in the clear and make senders give up forever with `unknown-recipient`.
Mitigation: `delegatedFrom` as a proof signed by the key the address domain publishes in DNS, validated before using `hostKey`, `clientApi` or `federationInbox`, plus pinning of the `hostKey` on first use (RFC 0001 §3.2). The old argument that the transport signature solved this held for the outbound path and never for the inbound one, and the very `hostKey` anchoring that argument came from the unsigned document the attacker served.

### A33. Correlation across issuers through the root identifier
Two issuers, or a broker who buys both databases, cross-reference records by the root's `did:key` and discover they are talking to the same person, despite the address per relationship.
Mitigation: no object destined for an issuer carries the root (ADR-0022). The `Consent`, the send capability and any entry addressed to the issuer use the identifier of that relationship's contact key. The address per relationship had existed since ADR-0008 and did not protect, because the object the relationship creates carried the global identifier along with it.
Residual: the issuer loses the ability to prove to a third party which global identity accepted. It is an accepted cost, and it is written in ADR-0022.

### A34. Downgrade by the inattentive member
A single member of a project switches on assisted mode in their own box, and their host starts reading in the clear the tasks of the circles they belong to, without anyone else in the circle knowing.
Mitigation: assisted mode MUST NOT reach a group of which the identity is not the sole holder, and a downgrade affecting someone else's project becomes a signed entry in that project's log (RFC 0006 §7, RFC 0013 §8). The anti-requirement of never degrading E2EE silently held for whoever switches it on, and was silent for everyone else.

## 3. Metadata: what leaks even with E2EE

The exhaustive list is maintained in RFC 0006. In summary: from and to addresses, envelope types, timestamps, approximate sizes, opaque group and epoch ids, and the urgency hint (if the user enables it). Future mitigations under study: size padding, temporal batching of delivery, sender obfuscation (sealed sender). None promised for v0.1: better an honest draft than a promise with a hole in it.

## 4. Security anti-requirements

- Never depend on a secret in the client (every client is reverse-engineerable).
- Never treat platform approval (app stores, Google, Microsoft) as a security layer.
- Never degrade E2EE silently: a downgrade is always an explicit act by the user, auditable locally.
