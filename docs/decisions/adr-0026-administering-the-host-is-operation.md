# ADR-0026: Administering the host is operations, and the protocol has no superuser

**Status:** accepted | 2026-08-12 | Michel Wilhelm

## Context

Whoever operates a host needs someone in charge of it: to see what is stuck, to tighten limits, to answer for abuse. The reference host already had the concept and called it the owner. It lives in `settings['owner']`, holds an account's `did`, and is consulted by `api::is_owner`.

What it did not have was a way to change it. There were two paths, and both are bad:

1. **The first account registered becomes the owner.** In a proof of concept the first account is almost always a test account, and whoever actually operates the host is locked out of their own administration.
2. **The `--owner` flag in the compose file.** Fixing it means editing the file and redeploying, and the flag beats the database, so anyone trying to fix it through the database would see no effect at all.

The request that prompted this decision asked for "a command to set a specific user as a superuser". The word matters more than it seems: superuser, in the vocabulary of operating systems and of nearly every SaaS, means whoever can see and do everything. If e-work adopted that meaning it would contradict the entire product, and contradict it silently, because whoever operated the host would believe they hold powers they do not, or worse, would believe they can promise third parties an auditing power that the cryptography prevents.

It is worth distinguishing this from two things that already exist and are not it. The `owner` and `admin` roles of a **project** (EW-RFC 0013, ADR-0021) are authority inside a project, negotiated among participants, and have nothing to do with whoever operates the server. And the identity vault (ADR-0024) is an application service, with accounts of its own, which is also not this.

## Decision

**Administering the host is an operations role over one instance, not a protocol role. There is no e-work superuser, and there will not be one without an ADR that creates it.**

1. The role stays the `owner` the host already had, stored in `settings['owner']`. No parallel concept is created: a new `superuser` would be a column no code reads, sitting next to an `owner` that all the code already reads.
2. The command `ework-host owner <address>` sets who administers, and `ework-host owner` on its own shows who it is. It opens the same database the server opens, works with the host running, and takes effect immediately, with no restart.
3. The account must exist on the host. An address with no account is refused rather than stored.
4. `--remover` leaves the host with no owner, and the "first account becomes owner" rule stops applying from then on: removing the owner is a decision, and the next account registered does not undo it on its own.
5. When the `--owner` flag is declared, the command writes and **warns** that the operator's declaration beats the database. Answering "done" while the host keeps obeying the flag is the worst way to fail: silent and convincing.

And, above all, what administering does **not** grant:

- **It does not decrypt anyone else's content.** There is no key for that: content is encrypted end to end for the participants (ADR-0004, ADR-0011), and the host holds an opaque envelope. This is not a rule the host follows out of politeness, it is something it has no way of doing.
- **It does not skip consent.** Delivery still requires prior consent (ADR-0006), including for whoever administers.
- **It does not act on anyone's behalf.** Every action holds by the signature of whoever wrote it, and an action without authority is an entry without effect (ADR-0021). Administering the host grants no third party's signature, and therefore no way to write in a third party's place.
- **It is not a project role.** It opens no circle, does not become `owner` of someone else's project, and does not show up in policy composition.

What it does grant is what the host already granted `is_owner`: operation of the instance.

## Alternatives considered

**Create a real `superuser`, one who sees everything.** Rejected because it is impossible in the part that matters and dishonest in the part that remains. Content is encrypted for the participants, so "see everything" would reach metadata and nothing more; and the expectation the word creates, in a product that promises E2EE, is worse than the absence of the feature. An operator who believes they hold that power promises it to third parties, and finds out too late that they do not.

**An `accounts.superuser` boolean column, to admit several.** Rejected for now: no code would read it, and a flag nothing consults is worse than no flag, because it looks like protection. When there is a second person operating a host and an action that depends on it, the change is a new ADR and a data migration, and today's `owner` is its first record.

**Leave it as it was, with the `--owner` flag in the compose file.** Rejected because it does not solve the case that exists: a host already up, with the wrong owner or none, and an operator with no appetite for a redeploy to fix one line of configuration. And because the flag beat the database silently, which is what made the problem hard to see.

**Put the owner change in the web interface.** Rejected as circular: whoever needs the command is precisely whoever cannot administer through the web. The terminal is the path that does not depend on what is broken.

## Consequences

**The promise becomes small and true.** Whoever operates an e-work host administers the instance, not the people in it. It is less than a SaaS administrator holds, and it is the price of everything else: a host that could read would not be an e-work host.

**Whoever operates needs access to the server to declare themselves owner.** That is deliberate: a remote path to becoming owner would be a remote path to taking over the host.

**The command writes to a database the server is using.** It is a single write to one configuration key, with no migration and no lock, and the host reads that key on every check, so the effect is immediate and requires no restart.

**The flag still wins.** Whoever wants the database to rule takes `--owner` out of the compose file. The alternative, making the database beat the flag, would turn a database write into a way around the operator's declared configuration, which is the opposite of what the flag exists to guarantee.
