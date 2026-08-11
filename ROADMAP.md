# Roadmap

Phases with exit criteria. Dates are honest targets for a one-person project with a full life, not promises.

## Phase 0: Specification (now, 2026-Q3)

- [x] Prior art research (2026-08-04)
- [x] Founding decisions (ADRs 0001 to 0015)
- [x] The RFC suite 0000 to 0016 in Draft status (0011 to 0016 added on 2026-08-04)
- [x] Resolve the blocking questions: session authentication (RFC 0004 §1) and scheduled release of escalation (RFC 0009 §6), both closed on 2026-08-04
- [ ] End-to-end consistency review (the six scenarios run "on paper" with no magic)
- [ ] Decide the final name (RFC 0000 OQ-1)
- [x] Property verification of the state machines with light model checking, in `scripts/check_maquinas.py`, inside `make check` (2026-08-10). There are **two** machines, not three: the confirmation one is a view of the execution one, and treating them as separate things is what let an edge exist in only one of them. It found four defects: the two diagrams diverged, a task in `awaiting-confirmation` was stuck with no way out for the owner, `countered` looked final, and the implementation did not check compatibility between action and state, so `confirm` completed a task nobody had executed

**Leaves the phase when:** an external technical reader can point at holes just by reading, and the known holes are recorded as questions rather than hidden.

The detailed technical execution (shared core, server, client, web wallet and the E1 to E10 sequence) is in the [implementation plan](docs/implementation-plan.md).

## Phase 1: Proof of life (MVP delivered on 2026-08-04)

- [x] Rust confirmed for the core. Workspace in impl/: `ework-core`, `ework-host`, `ework-cli`.
- [x] Reference host: one binary, `/.well-known/ework`, session by challenge-response, batched RPC with per-type state, raw binary blobs, federation inbox, consent edge and a web interface. Assisted mode, with the MLS points marked in the code.
- [x] CLI client: create an identity, register, list, accept, comment, complete, verify history, add an agent.
- [x] Real federation between two hosts, with consent and the edge working.
- [x] A simulated issuer sending the monthly invoice with a payment payload and receiving filtered status back.
- [x] Beyond what was planned: an autonomous agent barred by key class (S6), silent retirement with no oracle, and detection of history tampering through the hash chain.
- [x] A Python library (`impl/ework-py`) for issuing jobs from other projects, serving also as a second independent implementation. It found two real defects in the Rust core on its first run, which is exactly what it is for.
- [x] A retry queue with backoff on the normative schedule of RFC 0005 §3, with a background worker, giving up by age and distinguishing transient from permanent errors. Tested by knocking the destination host over in the middle of a delivery.
- [x] Running on a real domain: `ana@eworkprotocol.org` with the host at `app.eworkprotocol.org` (delegation), stack in Arcane, image in Harbor.
- [x] Rate limiting on registration, session, handle probing, inbox and RPC, with an explicit refusal and `retryAfter`. Verified in production behind the tunnel, reading the real address from `cf-connecting-ip`.

**Left the phase when:** scenario S1 ran end to end between two hosts, with consent, an offer carrying a binary attachment, acceptance, completion and status coming back. `impl/demo.sh` reproduces all of it.

### After the phase cut (2026-08-05)

Implementation kept landing without reopening the phase, because none of it changes the exit criteria. It is recorded here so the roadmap does not lie about the state of things:

- **A second host on `dainner.app`**, and with it the first federation between two real domains: until then it had only ever run between two processes on the same localhost, where it always works.
- **Complete compartments** (EW-RFC 0013): circles, sealed sections and the public milestone that lets a dependency cross circles without leaking the existence of the private task. Scenario S3 from the visibility side, still without MLS.
- **Real attachments**: upload by hash in the client, transfer between hosts with a transport signature and a grant list, hash checked on arrival.
- **A dependency graph** (EW-RFC 0002 §4) refusing cycles and blocking actions while something is pending.
- **Address rotation with selective migration** (EW-RFC 0003 §7.3): the clean sweep that replaces every address and takes along only the ones you choose.
- **Account deletion** (EW-RFC 0016 §5) with a window for changing your mind, and an address that does not come back onto the market.
- **Escalation of a critical task** (EW-RFC 0009) with scheduled release by the host, the PagerDuty scenario from the vision.
- **Bridges** (EW-RFC 0010): an iCalendar feed for any calendar app, and an issuer REST gateway with a webhook.

Three security defects were found along the way, all of them by connecting the ends rather than by reading the code: attachment download with no authentication, an account being deleted still publishing its identity document, and the status filter that obeyed the issuer instead of the person's policy. All three became normative rules in RFCs 0005, 0003 and 0007.

## Phase 2: E2EE (target: 2027-Q1)

- MLS groups via OpenMLS: personal group (multi-device), relationship group, project group.
- Cryptographic agility exercised: a hybrid post-quantum ciphersuite (ML-KEM with X25519) as a target as soon as it stabilises in the MLS ecosystem (EW-RFC 0006 §8).
- Encrypted operation log in sync; key backup with a recovery kit; device verification in the CLI.
- Scenario S3 (a project with 3 identities plus a member joining with history re-share) demonstrated.

**Leaves the phase when:** the host demonstrably cannot read anything from the E2EE boxes (documented adversarial test) and members joining and leaving work the way the mental model of ADR-0004 says.

## Phase 3: Real people (target: 2027-Q2)

- A client with an interface (proposal: Flutter, mobile first) usable by a non-technical person: onboarding with a recovery kit, quarantine of requests, accepting offers, urgency and acknowledgement.
- Escalation (S4) with a consented contact.
- Bridges: client-side CalDAV gateway plus `.ics` import; issuer REST gateway v0.
- Real daily use by the author and family (dogfooding): real bills through their own issuer gateway.

**Leaves the phase when:** a non-technical family member uses it for 30 days without support.

## Phase 4: Opening up (target: 2027-H2)

- Full translation of the suite into English (the canonical version, decided on 2026-08-04).
- Publication: public repository, specification site, implementation guide.
- Seek a second independent implementation (the criterion for any RFC to go from Proposed to Stable).
- Conversations with adjacent communities: calext and sml at the IETF, self-hosted projects (Nextcloud, Vikunja, Tasks.org), and potential pilot issuers.
- Define governance and licences (RFC 0000 OQ-2 and OQ-3).
- Specialist legal review of RFC 0016 (the language of cryptographic destruction, the roles, the regulatory profiles): this blocks publication.

## Mapped risks

| Risk | Mitigation |
|---|---|
| Scope swallowing the project (this is an entire protocol) | Phases with one target scenario each; a minimal core; extensions can wait |
| MLS being hard to get right | OpenMLS (an audited library), never home-grown cryptography; phase 2 dedicated to nothing else |
| Nobody issuing (coldness on the company side) | A trivial issuer gateway plus an email fallback; the "open direct debit" narrative; dogfooding with an issuer of our own |
| Staying at one user forever | Phase 4 exists precisely for that; bridges from phase 3 onwards; specs written for third parties from day one |
