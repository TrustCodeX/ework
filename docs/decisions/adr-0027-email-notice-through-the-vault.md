# ADR-0027: The host asks, the vault sends, and neither learns what the other knows

**Status:** accepted | 2026-08-12 | Michel Wilhelm

## Context

A task inbox that only tells you something when the screen is open does not do what the product promises. The six scenarios in the vision all depend on the person finding out that something arrived while they are doing something else, and today the only notice that exists is the screen refreshing itself, which requires the screen to be open.

EW-RFC 0009 §2 defines urgency and the notification behaviour each level calls for, and deliberately defines no channel: it maps to ntfy priorities, Web Push and Android and iOS notification channels, and stops there. Choosing a channel is not a protocol decision.

The problem is that the only off-screen address this product knows sits on the wrong side of a boundary. ADR-0009 deliberately keeps email out of the protocol: it is a verified attribute for discovery, with blind delivery, and the host does **not** index people by email. And ADR-0024 put the identity vault outside the protocol, in a separate service, with the sentence written into the host's code: "the host announces it in discovery and nothing more: the host does not talk to it".

So: the one who knows something arrived is the host, the one who knows the email is the vault, and they do not talk. Any email notice crosses that line, and the question is where, carrying what, and who learns what along the way.

## Decision

**The host asks, the vault sends, and the boundary moves as little as possible: the host never learns an email address, and the vault never learns content.**

1. **The preference lives on the host**, as a column on the account. It is "this inbox wants to be told off-screen", and nothing else: no address, no channel.
2. **The host asks by handle.** When something arrives **from outside**, it sends the vault `{handle, host}` and stops there. Not who sent it, not the subject, not the task identifier.
3. **The handle, not the `did`.** The vault already knows the handle, because it is what the person reads in their own list of boxes. The `did` is global and stable, and sending it would make the vault store an identifier it does not need to hold.
4. **The notice fires only at the from-outside entry point.** That is the federation inbox, not the internal state-change notification: that one also fires for what the person themselves just did, and emailing someone about their own action is the fastest way to turn a notice into noise and get it switched off.
5. **The vault decides whether it may send**, and the conditions are its own: address confirmed, and at most one notice per window for the same box.
6. **The notice route requires a secret shared** between the two services, and **answers the same thing in every case**: box that does not exist, address not confirmed, throttled notice and sent notice all come out identical. Without that, whoever held the secret could find out, handle by handle, which of a host's boxes have a vault there.
7. **The notice says something arrived, not what arrived.** That is not discretion, it is all the service has: content is encrypted for the participants and the vault has never seen it.
8. **With no secret configured, none of this exists.** A host without `EWORK_VAULT_SECRET` asks for no notice, a vault without `VAULT_SECRET` keeps the route closed, and the interface does not offer the option rather than offering it switched off.

## Alternatives considered

**The preference in the vault, with the host always notifying.** Less code: the host sends on every delivery and the vault filters. Rejected because the cost falls on people who asked for nothing: the vault would come to know arrival times for every box of everyone who has an account there, including those who never turned notices on. Keeping the preference on the host is what limits the vault's knowledge to those who asked.

**The host storing the email.** It would solve this with one service fewer, and it contradicts ADR-0009 on the sentence that matters most in it: the host does not index people by email. A host with an email column per account is a host that, once leaked, hands over the list of who is who, which is exactly what blind discovery exists to prevent.

**The vault asking the host.** It would reverse the direction and spare the vault an inbound secret. Impossible: reading someone's inbox requires that person's key, and what the vault holds is the **encrypted** phrase. It has no way to authenticate as anyone, and it is good that it does not.

**Only the client notifying.** The client already knows when something arrives and could send the request. Except it only knows with the screen open, which is precisely the case where the notice is not needed.

**Putting the subject in the email.** It is what nearly every product does, and here it is impossible without undoing the rest: neither the host nor the vault reads content. A summary in the email would require someone to have read it.

## Consequences

**The vault learns the timing, for those who asked.** That is the price, and it is real: for whoever turns the notice on, the vault comes to know when something arrives in that box, even though it knows neither what nor from whom. Whoever does not want that cost does not turn the notice on, and loses nothing beyond the notice.

**One more secret to operate.** Both services need the same value, and if it diverges the notice silently stops going out, with one line in the host's log. That is the acceptable failure: the envelope has already been delivered when the request happens, and the alternative (refusing delivery because a courtesy email failed) would trade what matters for what helps.

**The notice covers federation, not everything.** It fires from the federation inbox, so it covers what arrives from outside. A change the person makes on another device of their own produces no email, and that is intentional.

**One per window, and the window belongs to the operator.** A box that receives fifty envelopes in an afternoon produces one email. That is what keeps the notice switched on after the first week.

**This is NOT a protocol channel.** No RFC changes, no host needs to implement this to federate, and a host without a vault remains complete. It is the same layer as ADR-0024, and the boundary that moved is described above rather than erased.
