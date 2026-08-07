# ADR-0018: No second factor at the host; the second proof is another device of yours

**Status:** accepted (2026-08-06)

## Context

The design handoff asks, on the sign-in screen, for "extra protection (second factor) on by default" and a "one-hour lockout warning after three errors". Neither exists in the host, and no RFC defines them.

The temptation is to implement what every product implements: a TOTP secret kept on the server, and an attempt counter that locks the account. That would be copying a design that solves a different problem.

Second factors on top of passwords exist because a password is "something you know", and something you know leaks through phishing, reuse and database breaches. e-work has no password anywhere. Signing in means proving possession of the recovery kit, which already is "something you have". A second layer, if it exists, has to protect against something else: the kit itself leaking.

And there is a deeper problem. EW-RFC 0003 and ADR-0003 say the server **is not the owner of the identity**: the identity is the root key, and the host merely routes and stores. A second factor kept at the host, required to sign in, inverts that in practice. The host becomes able to stop the owner from using her own identity, which is precisely the power the protocol was designed not to grant it. The same holds, with more force, for "lock the account after three errors": locking an identity is the host deciding about something that is not its own.

There is also the side effect nobody anticipates: if the second factor is mandatory and the person loses it, we create a **second way to lose the account forever**, in a protocol whose RFC 0003 §8 already says there is no second copy. Two ways to lose is worse than one.

## Decision

**There is no second factor at the host. The second proof, when required, is a signature from another already-authorised device of the same identity. Rate limiting exists, is per network origin, and never per identity.**

1. **No second-factor secret on the server.** No TOTP, no email code, no SMS. The host neither stores nor verifies any factor beyond the signature.

2. **A device approves a device.** Adding a new device already requires, today, a signature from the root key (the kit) or from an already-authorised device, and the signed identity document is the proof. That is already two factors in the sense that matters: the kit alone, or a device of yours that still works. It is the same mechanism as EW-RFC 0003 §5, inventing nothing.

3. **Blocking is per origin, not per account.** The limiter in EW-RFC 0011 §4.3 already refuses on a sliding window and returns `over-rate` with `retryAfter`. It protects the host against volume, which is the real problem, and it gives nobody the power to lock someone else's inbox by sending three wrong attempts to their address.

4. **The interface does not promise what does not exist.** The door shows no second-factor switch, and it says what actually protects: the key is on the device, and without the kit there is no way in.

## Alternatives considered

**TOTP kept at the host, mandatory.** Rejected for two independent reasons: it gives the host power to deny the use of the identity, contradicting ADR-0003; and it creates a second way to lose the account permanently.

**TOTP optional, off by default.** Rejected for being worse than both ends. Whoever turns it on gains the illusion of extra protection that the host can remove at any time (it is the host that keeps the secret), and whoever leaves it off gains nothing. A security control that the most likely adversary, the host operator, can disable on its own is not a control.

**Email code.** Rejected: e-work exists in part because email is not a trustworthy channel for this, and the reference host's domain does not even accept mail. Besides, it would turn the email provider into a third party with power over the identity.

**A second factor in the identity, inside the signed document.** This is the only form that does not contradict ADR-0003, because it would travel with the identity and the host would have no power over it. Rejected **for now**, not on principle: it would require extending EW-RFC 0003, and the benefit over "a device approves a device" is small while the number of devices per identity is what it is. It remains a possible extension, with the path noted.

**Locking the identity after N errors.** Rejected: it is a denial-of-service tool against third parties. Anyone who knows someone's address could lock that person's inbox whenever they liked.

## Consequences

- Issues #27 (I already have an account) and #28 (I lost my device) stop waiting on a second factor. #28 becomes what it always was in the protocol: sign in with the kit, or approve from a device that still works, and revoke the lost one in Settings.
- **A leaked kit remains total compromise**, and this is said in plain words on the About screen, in the "what we cannot promise" section. The decision does not improve that case; it refuses to pretend it does.
- Whoever wants additional protection has a path that already exists and is stronger than TOTP: do not keep the kit on any device, and use a dedicated device to approve the others.
- The Settings screen gains device revocation as the incident-response mechanism, which is where it belongs.
