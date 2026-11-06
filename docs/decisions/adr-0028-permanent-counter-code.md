# ADR-0028: The counter code may never expire, and what kills it is the list, not the clock

**Status:** accepted | 2026-08-26 | Michel Wilhelm

## Context

EW-RFC 0007 §3.3 defined the in-person offer: a `consent.offer` envelope signed by the issuer's host, bearer by nature, that becomes a QR code and lets a person admit an organisation on the spot, with no quarantine. The RFC itself calls it the main path in practice, because it is where trust already exists in the physical world.

It came with a ceiling: `expiresAt` mandatory, 30 days at most, with the reason written in one sentence, "a printed QR outlives the contract that justified it". The sentence is right. The conclusion was not.

The ceiling treats a deadline as if it were control, and it is not. A short deadline was the only withdrawal mechanism that existed as long as the code was thought of as paper, and its whole bill lands on whoever prints it. The case that prompted this ADR is literal: a QR printed in three dimensions, in relief, to sit on the counter. The same holds for a shop window plate, a door sticker, an engraved tag, a reception poster, a label on equipment installed in a customer's home. None of that gets reissued every 30 days.

In practice the ceiling forces a choice between two defeats: the organisation stays out of the protocol, or it leaves an expired plate where it is, teaching its customers that the counter code tends to fail. The second is worse, because it destroys trust in precisely the gesture the protocol wanted to make trivial.

And there was the other side: whoever issued a code had nowhere to see what they had issued. The code appeared on screen, went to the printer, and vanished from the system. There was no list, no state, and no way to take a code out of circulation before its deadline, not even in the case where that matters most, which is the plate still hanging in the shop that closed.

## Decision

**The offer declares how it ends, and permanence is allowed to whoever publishes how to revoke.**

1. **A deadline or permanence, never both.** The offer body carries `expiresAt` (30 day ceiling, as before) or `permanent: true`. With neither, the offer is refused: a bearer code with no declared end is a code nobody knows how to withdraw.
2. **A domain that issues permanent codes publishes the revocation list** at `GET /.well-known/ework/offers/revoked`, carrying the `id`s of the offers it killed. The list covers the whole domain, and is not a per-offer query.
3. **Failure closes for permanence and opens for deadlines.** A permanent offer whose list cannot be fetched MUST be refused, because permanence without published revocation is a code with no brakes. An offer with a deadline stays valid when no list exists, and that is what keeps compatible a domain that never heard of this decision.
4. **Revocation is final.** An `id` that enters the list does not leave it. A list that shrinks resurrects a dead code, and resurrects it first in readers with a stale cache, who are the ones who will not notice.
5. **Revoking the offer ends no relationship.** Whoever already accepted holds a Consent, which ends through EW-RFC 0007 §6.1, with the consequences described there. Two objects, two actions.
6. **Whoever issues sees what they issued.** The host keeps the offers it signed on an organisation's request, and the organisation sees the list with each one's state (valid, expired, revoked), the code to reprint, and the button to revoke.

## Alternatives considered

**Keep the 30 day ceiling.** It is the previous state and has the virtue of requiring nothing new. Rejected because it excludes from the protocol exactly the use the counter promises: the code that stays in the physical world, which is what saves the person from typing anything at all. A mechanism that only serves disposable paper is not a counter, it is a coupon.

**Permanence with no revocation.** Making `expiresAt` optional would be enough. Rejected because it produces an immortal bearer: the only way to kill it would be rotating the domain key, which takes down every other offer and the host's identity verification along with it. Issuing forever without being able to withdraw is the dangerous version of this same decision.

**A long deadline, in years.** Trading 30 days for 3650. Rejected because it is the same design with a bigger number: still no withdrawal, and it additionally promises the organisation a date on which the plate stops working with nobody left who remembers why.

**Per-offer status query at `/.well-known/ework/offers/<id>`.** It would publish less data and be simpler to implement. Rejected on privacy: the query tells the issuer which code was scanned, when, and from which network origin, turning every plate into a counter of who pointed a camera and walked away. That is the same oracle that closed the handle proof lookup desk (EW-RFC 0003 §3.1), and reopening it here would not be coherent.

**Publish the living instead of the dead.** A list of valid `id`s would solve growth, since entries disappear as they die. Rejected because it publishes more: the list of the living is the inventory of every plate that organisation has in the world, while the list of the dead tends to zero in normal operation and only grows when something went wrong. Between publishing what exists and publishing what was withdrawn, the second says less.

**Reversible revocation.** It would allow undoing a wrong tap, which is expensive here: the engraved plate does not come back. Rejected because the reversal reaches stale caches last, and it is impossible to explain to someone that the code works again in part of the network. The interface covers the mistake in the right place, by asking for confirmation with the consequence spelled out before revoking.

## Consequences

**Permanence depends on the domain still answering.** A host that is down takes its own permanent codes down while it is out, and takes nobody else's with it. That is the safe side of failure, and it is honest: permanence is a promise made by whoever issues, and whoever stops answering is not keeping it.

**One more request on the reading side.** The client already queried the domain to check the signing key; now it queries the list as well. The short cache the RFC recommends bounds the cost, and the number an organisation will hold us to is the gap between revoking and the code ceasing to work.

**The list grows and the protocol does not yet define pruning.** It is recorded as an open question in EW-RFC 0007, with the proposal under study (a cutoff by issuance date, published alongside) and the reason it is not settled: the cutoff would have to be per issuer, not per domain, so that one organisation clearing its plates does not take down its host neighbour's.

**The host starts keeping what it issued.** That is new record keeping on the issuer's side, about the organisation itself and not about whoever reads: the offer is bearer and the host does not know, and does not come to know, who scanned it. Whoever accepted shows up as a relationship, as they already did.

**Nothing changes for those who only use deadlines.** An offer with `expiresAt` is identical, verified as before, and valid against a domain that publishes no list at all.
