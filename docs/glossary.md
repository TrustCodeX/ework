# Glossary

The project's vocabulary. The Portuguese column is kept because the drafts were written in Portuguese first: it is what makes the translation mechanical rather than archaeological.

| Term | PT | Definition |
|---|---|---|
| Taskbox | caixa (de tarefas) | The collection of tasks belonging to an identity hosted on a host, analogous to email's mailbox. One identity can have several boxes, on different hosts. |
| Host | host | A server that hosts boxes, takes part in federation, routes envelopes and wakes clients. In E2EE mode it does not read content. |
| Client | cliente | The user's application (mobile, desktop, web, CLI). It holds device keys, decrypts content and runs local rules. |
| Identity | identidade | The user's root key pair and the identity document they sign. Independent of any host. |
| Handle | endereço | A readable name `name@domain` pointing to an identity, with proof in both directions. |
| Issuer | emissor | An organisation (or person) that sends task offers to third parties. Organisations are anchored in a domain. |
| Task offer | oferta (de tarefa) | A task proposed to another identity, subject to acceptance, refusal or a counter proposal. |
| Consent | consentimento | A first-class object authorising an issuer to deliver offers to a user, with scope, validity, status policy and revocation. |
| Send capability | credencial de envio | A proof derived from the consent that the issuer presents on every send. |
| Envelope | envelope | The unit of transmission between parties: a routing header in the clear, a body (encrypted by default) and integrity proofs. The "packet". |
| Typed payload | carga tipada | Structured, versioned data inside a task: payment, scheduling, delivery, approval. |
| Attachment | anexo | A file or binary datum attached to the task, stored as a hash-addressed blob. |
| Action | ação | A structured operation offered by the task (pay, open, approve, copy), always confirmed by the user. |
| Blob | blob | A raw, immutable binary object, addressed by hash, transferred without base64. |
| Project | projeto | A shared collection of tasks with members, roles and a cryptographic group of its own. The "epic". |
| MLS group | grupo (MLS) | A cryptographic group (RFC 9420) whose members, and only they, open the packets. It exists for one person's devices, for user-issuer relationships and for projects. |
| Assisted mode | modo assistido | An opt-in state of a box or collection in which the host receives keys in order to provide services (search, hosted rules, server-side escalation). |
| Negotiation | negociação | The offer phase: pending, accepted, declined, countered, expired. Per participant. |
| Execution | execução | The work phase: to do, in progress, blocked, completed, failed, cancelled. |
| Actionable | acionável | A task whose dependencies are satisfied and whose start window has arrived. |
| Ack | ack | Acknowledgement of an urgent task. It pauses escalation; it does not complete anything. |
| Escalation | escalonamento | The chain of progressive notification when a critical task is not acknowledged in time. |
| Dedup key | chave de deduplicação | An idempotent identifier per issuer: resends update the existing task. |
| Contact key | chave de contato | A key belonging to one relationship, with an address of its own. It identifies the user to that issuer without revealing the root, and validates the credential presented in the relationship's group. One per issuer, never listed publicly. |
| Contact address, alias | endereço de contato | An opaque address derived from a contact key, known only to that relationship. |
| Primary handle | handle principal | The person's public, stable address, used with those they invite personally. |
| Key rotation statement | declaração de rotação | A signed object proving continuity between an old key and a new one. |
| Selective migration | migração seletiva | Choosing which correspondents receive the rotation statement. Whoever is left out loses the address. |
| Silent retirement | aposentadoria silenciosa | Ending a relationship without announcing it: the address starts answering as if it had never existed. |
| Leak attribution | atribuição de vazamento | Finding out who leaked or sold your data, because only that issuer knew that address. |
| Unlinkability | não-ligabilidade | The property that two issuers cannot discover they are talking to the same person. |
| Purpose | finalidade | What an offer is for, declared in the consent and on every send. |
| Oracle | oráculo | Any difference in response that reveals whether an address exists. Forbidden in EWP. |
| Verified attribute | atributo verificado | A real-world identifier (phone or email) bound to an identity by attestation. It is neither an identity nor an address. |
| Attribute attestation | atestado de vínculo | An object signed by a verifier binding an attribute to an identity, with validity and an assurance level. |
| Verifier | verificador | Whoever confirms that the person controls an attribute, typically through a one-time code. It can be the host itself. |
| Civil identifier | identificador civil | National ID numbers, social security numbers and equivalents. Out of scope for the protocol because they are permanent and immutable (ADR-0010). |
| Compartment | compartimento | A subset of a project's members with a cryptographic group of its own. Whoever is outside does not have the key. A specification term; in the interface it is called a circle. |
| Circle | círculo | The product name for a compartment ("just me and the carpenter"). It is what the user reads and says. |
| Actor class | classe de ator | A property of a key: human, assisted, autonomous or system. Only a human key confirms as a human. |
| Delegation | delegação | A scoped credential that lets an agent act on someone's behalf, with limits and immediate revocation. |
| Human approval | aprovação humana | A signature by a key of the human class, required before or after execution. |
| Execution confirmation | confirmação de execução | An attestation that the task was in fact done, possibly by someone who did not execute it. Different from an ack. |
| Evidence | evidência | Proof required at completion: an attachment or a structured result. |
| Entry | entrada | The unit of a task's history: a message plus an action, signed. An ordinary comment is an entry with a null action. |
| Action | ação | What the entry does to the task (accept, complete, dispute). Null when the entry is only a message. |
| Comment | comentário | The colloquial name for an entry, especially one without an action. |
| Private note | nota pessoal | A comment published in the group where only its author is a member. It never reaches the issuer. |
| Activity | atividade | A history entry derived locally from the signed operations. It does not travel, and therefore cannot be forged. |
| Causal ordering | ordenação causal | The rendering rule that guarantees the same conversation for everyone, with no server doing the ordering. |
| Retraction | retratação | A request to remove an entry that carries no action. Conforming peers comply; whoever already received it, received it. An entry with an action is never retracted. |
| Sealed section | seção selada | Fields of a shared task encrypted for a narrower compartment. |
| Classification | classificação | A sensitivity label on a field (`money`, `health`, `personal-data`), which the project's policy maps to a compartment. |
| Milestone | marco | A neutral task in a broad compartment representing the outcome of a private task, so that a dependency works without leaking. |
| Hint | dica | A presentation restriction with no key behind it. It must never be presented as a privacy control. |
| Data subject | titular | The person who owns the data: the user. |
| Controller | controlador | Whoever decides the purpose and means of processing. The issuer, for what it processes. |
| Processor | operador | Whoever processes on someone else's behalf. The host, as regards the content it does not read. |
| Crypto-shredding | destruição criptográfica | Deleting by destroying the keys: the remaining ciphertext becomes permanently unreadable. |
| Regulatory profile | perfil regulatório | A registrable bundle of one regime's requirements (br-lgpd, eu-gdpr, us-hipaa). |
| Grace window | janela de arrependimento | The period with the account frozen between the deletion request and definitive destruction. |
| Freshness proof | prova de frescor | A periodic statement signed by a device that the identity document is current. |
| Guardian | guardião | A contact holding a fraction of the recovery kit, for emergencies or inheritance. |
| Size class | classe de tamanho | The padding step for envelopes; the host sees the step, not the exact size. |
| Scheduled release | liberação agendada | A pre-encrypted envelope that the host releases at a given time, unless it is cancelled beforehand. |
| Assurance level | nível de garantia | How strong the verification of an attribute was (low, substantial, high), in the spirit of eIDAS. |
| Directory | diretório | A service that forwards consent requests addressed to attributes. A doorbell, never a catalogue. |
| Blind delivery | entrega cega | Forwarding without revealing the destination, answering the same whether a binding exists or not. |
| Enumeration | enumeração | Sweeping a space of identifiers to discover which ones exist. |
| Quarantine | quarentena | The silent area where consent requests from strangers land. |
| Receipt | recibo | Confirmation that an envelope was delivered or processed, filtered by the privacy policy. |
| Bridge | ponte | A compatibility gateway to the legacy world: CalDAV and VTODO, email, webhooks. |
| Recovery kit | kit de recuperação | An offline secret generated when the account is created, allowing identity and keys to be recovered. |
| Workspace | espaço | A context of the user's life (personal, family, work), typically a box of its own, possibly on a host of its own. |
| Federation | federação | Server-to-server communication with signed envelopes, retries and the consent edge. |
| EWP | EWP | e-work protocol, the protocol's abbreviation in this suite (a working name). |
| Void entry | entrada sem efeito | An entry that stays recorded in the history and changes no state, because it arrived without the authority the action requires or because it is incompatible with the current state. |
| Action authority | autoridade (de uma ação) | The role an identity must hold in the task for the action it signed to take effect. Distinct from authenticity, which is only who signed. |
| Signed head | cabeça assinada | The periodic publication of the pair (task, hash of the most recent known entry), used to detect a fork in the history. |
| Delegation proof | prova de delegação | An object signed by the key published in the addresses' domain, authorising a third-party host to serve that domain. |
| Ciphersuite floor | piso de ciphersuite | The weakest cryptographic suite an implementation MAY accept; below it, the group is refused. |
