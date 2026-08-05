# Implementation plan

How to get from eleven RFCs to three things running: a **server** (the reference host), a **client** and a **web wallet** holding every setting of the identity. The [ROADMAP](../ROADMAP.md) has the phases and the exit criteria; this document has the technical execution.

## 1. The decision that organises everything: one core, three shells

The classic mistake would be writing three implementations of the protocol. Instead: **one core library, compiled for every target**.

```
                    ┌───────────────────────────┐
                    │  ework-core (Rust)        │
                    │  identity, keys, MLS,     │
                    │  envelopes, sync, rules   │
                    └────────────┬──────────────┘
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
   native binary            wasm (browser)           FFI (uniffi)
        │                        │                        │
   server + CLI             web wallet            mobile and desktop app
```

The reasons: the cryptography (MLS via OpenMLS) has to be identical in all three, and duplicated cryptography is divergent cryptography; the same code validates envelopes on the server and on the client; and a third party who wants to write their own client gets the library ready-made, which is the difference between "the protocol is open" and "the protocol is usable".

Rust is the choice because of OpenMLS (the mature, audited implementation of RFC 9420) and because it compiles well to wasm and FFI. Go would be quicker to write on the server, but it would force duplicating the cryptography in the browser, which is exactly what must not happen.

## 2. The three deliverables

### 2.1 Server (the reference host)

One binary, one database, one domain. It has to run in a homelab without ceremony and carry a family.

| Module | What it does | RFC |
|---|---|---|
| Discovery | serves `/.well-known/ework`, `org.json`, handle proofs | 0001, 0003 |
| Session and API | authentication by device key, batched calls, state per type | 0004 |
| Blobs | resumable upload, ranged download, hash addressing, garbage collection | 0004 |
| Federation | signed inbox, a queue with retries and giving up, receipts | 0005 |
| Consent edge | validates credential, scope, rate and purpose before accepting | 0007, 0011 |
| Address registry | contact addresses per box, their state, retirement with no oracle | 0003 |
| MLS transport | orders commits per group, replicates ciphertext, is never a member | 0006 |
| Push | Web Push and UnifiedPush, with no content in the push | 0004, 0009 |

The build order: discovery and session, then tasks and sync, then blobs, then federation with a second host, and only then the consent edge (which only makes sense once there is somebody on the other side).

### 2.2 Client

The client is where the intelligence lives, because the server is blind. Two incarnations, the same core:

- **A CLI**, first, because it is what lets you test the protocol without arguing about pixels: create an identity, list, accept, complete, synchronise two devices, run two hosts talking to each other. It is also the conformance tool.
- **An app** (mobile first, proposed in Flutter or in Kotlin and Swift over the FFI): where the promise meets real people, with the three views, local alarms and the QR consent flow.

Responsibilities that necessarily stay in the client: decrypting, computing actionability from the graph, firing local alarms, running self-management rules, applying the anti-abuse presentation rules (RFC 0011 §3) and keeping the display preferences.

### 2.3 Web wallet

The wallet is the **configuration** surface of the identity, and it is web because configuration is something you do sitting down, on a big screen, once in a while. It runs entirely in the browser with the core in wasm: keys never leave the machine, and the wallet's server serves static files and nothing else.

What it has to expose, and this is where "all the possible settings" becomes a concrete list:

**Identity and keys**
- Create an identity, generate and check the recovery kit, restore from it
- Devices: list, verify by QR or short numbers, revoke
- Root key: rotate, view the continuity chain, challenge a rotation
- Hosts: add, remove, migrate a box, see what each host knows

**Relationships and addresses** (the heart of the wallet)
- A list of relationships with the address each one knows and since when
- Per relationship: granted scope, purposes, rate limit, maximum urgency, status policy
- Actions: revoke politely, retire silently, rotate while keeping the relationship
- Leak attribution: highlight when something arrives outside the purpose or from someone who should not have the address
- Requests in quarantine, with what the issuer asked for and an example of what it would send

**Human identifiers and discovery**
- Bind a phone and an email, with the attestation and its validity in plain sight (civil identifiers are out of scope, ADR-0010)
- Switch discovery on and off per attribute, per requester class and per directory (everything is born off)
- A record of who tried to reach you, through which attribute and with what outcome
- Revoke a binding, without depending on the verifier

**Privacy**
- The default status returned, and adjustment per relationship and per task
- Assisted mode per collection, with the consequence written before you switch it on
- The urgency hint in the clear: switch it on or not, with the trade-off explained
- What each host can see (the list from RFC 0006 §6, rendered, not hidden away in a PDF)

**Notification and urgency**
- Channels per device, quiet hours, who may pierce the quiet
- Escalation contacts: invite, accept, revoke, see whose contact you are
- The default escalation policy, and one per critical task

**Automation and agents**
- Auto-accept rules per issuer and per purpose, off by default
- Rules for automatic scheduling, deferring and archiving
- Where the rules run: on the devices, or on the host if in assisted mode
- Agents: create a delegation with scope (circles, actions, maximum amount, the band that requires a human, validity), see what each one did and revoke in one tap
- The default execution policy: what always requires human approval, regardless of what the issuer asks for

**Data and rights**
- Full export, import, migration between hosts
- Space used per box, blob retention policy
- Guided account deletion, with an export offered, revocations issued and a grace window (RFC 0016 §5)
- The data subject's rights exercisable in the interface: access, portability, rectification, objection, erasure (RFC 0016 §3)

### 2.4 The three views

List, calendar and board are projections defined in [RFC 0002 §10](../spec/data-model.md), not an invention of each client. A client may offer one, two, all three or a fourth that nobody thought of, and it stays interoperable, because the projection says how to derive the view from the data, and personal preferences never travel in the protocol. The single exception is the columns of a shared project, which are an agreement between members and therefore do travel.

## 3. What makes third parties write clients

This is the product, just as much as the server is:

1. **`ework-core` published** (crates.io, npm with wasm, FFI packages) with a stable, documented API.
2. **An executable conformance suite:** a binary that points at an implementation and says what passes and what fails, per profile (client, host, issuer gateway).
3. **Versioned test vectors:** signed envelopes, identity documents, rotation statements, example MLS groups, error cases. Without vectors, each implementation gets the cryptography right in its own way and nobody interoperates.
4. **A public sandbox server** with toy issuers (an "Acme Energia" issuing a fake monthly invoice), so a developer can test without standing up infrastructure.
5. **An implementation guide** covering the predictable mistakes: do not leak an address oracle, do not fetch remote resources, do not auto-accept a first contact.

## 4. Execution sequence

Each step ends with something demonstrable. No building for six months in the dark.

| Step | Delivery | Demonstrates |
|---|---|---|
| **E1** | `ework-core` with identity, keys and signed envelopes; a CLI that creates an identity and signs | the identity works |
| **E2** | A server with discovery, session, tasks and sync; a CLI with two devices | multi-device sync |
| **E3** | Binary blobs and attachments | the bill's PDF travels |
| **E4** | Federation between two hosts, with a queue and receipts | two domains talk |
| **E5** | Consent, contact addresses and the edge | an issuer comes in, and can be cut off |
| **E6** | An issuer gateway and a simulated issuer | the whole S1 scenario, end to end |
| **E6.5** | A reference directory with blind delivery, and an enumeration test as part of the suite | remote first contact, with no queryable catalogue |
| **E7** | MLS: personal, relationship and project groups | a genuinely blind server, with an adversarial test |
| **E7.5** | Compartments, sealed sections and default classification | the fitter does not see the payment, and it can be proven |
| **E8** | Web wallet over wasm | complete configuration in the user's hands |
| **E9** | Mobile app with the three views and local alarms | the S4 scenario, and real people using it |
| **E10** | Conformance suite, vectors and sandbox | third parties can implement |

E1 to E6 are phase 1 of the roadmap, E7 is phase 2, E8 and E9 are phase 3, and E10 opens phase 4.

## 5. Open decisions

1. **Is Rust confirmed for the core?** The argument is OpenMLS and compilation to wasm and FFI. The cost is the learning curve if Michel is writing alone, and the honest alternative would be Go on the server with the cryptographic core still in Rust, accepting one more boundary.
2. **App: Flutter or native?** Flutter delivers both platforms from one base, and the FFI to the core is a solved problem. Native gives better critical alarms and high priority notifications, which is precisely the S4 scenario.
3. **Are the web wallet and the web client the same app?** The proposal: they start separate (the wallet is configuration, the client is daily use), and they may converge later. Separating reduces the scope of the first web deliverable.
4. **Where to host the public sandbox** and under which domain, which depends on the protocol's naming decision.

## 6. What not to build now

Issuer reputation, a global identity directory, a complete CalDAV bridge, chained subdelegation, human presence attestation in an enclave, payloads beyond the four in RFC 0008, and anything to do with organisational on-call. All of it has a place reserved in the specification and no place in the first year of code.

Simple delegation to an agent (RFC 0014) is the exception and comes early, at step E7.5: Michel runs agents himself, it is the fastest way to exercise actor classes and execution policy with a real load, and scenario S6 depends on it.
