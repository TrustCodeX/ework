# ADR-0020: Discarding can be undone, blocking cannot, and both are silent

**Status:** accepted (2026-08-07)

## Context

Refusing a permission request is the most frequent action of whoever uses this inbox, and it was the least thought through. Measured on the reference host:

```
refuse (the button that says "Never")  ->  DELETE of the quarantine row, and nothing else
the same company knocks again          ->  back into quarantine, immediately
```

The knock path does not even consult the past decision: it deletes that issuer's previous request before inserting the new one, which is to say knocking again is explicitly supported. None of the host's 26 tables records who was refused.

Three problems follow, and the third is what motivated this decision.

**"Never" is not never.** The label promises a permanence that does not exist, and the consequence sentence, which is correct about the notice ("they are NOT told"), stays silent about what matters: they can come back tomorrow.

**There is no defence against insistence.** Whoever knocks every week keeps knocking, and the only available answer is to refuse every week.

**And there is no regret.** Refused by mistake, the envelope was deleted. There is no trash, there is no undo, and the other side has no signal at all that they should knock again, because refusing is silent by design. It is the worst possible combination: irreversible on your side and invisible on theirs.

EW-RFC 0007 does not deal with the subject. It specifies revocation, with the two paths of §7, and refusal of a **task** (the `decline` action), and says nothing about refusing a **permission request**. Unspecified area, so the decision comes before the code.

The real tension is with ADR-0008. Recording who was refused is accumulating state about people you decided not to have a relationship with, and any differentiated answer to a blocked party becomes an oracle: whoever knocks discovers whether they are blocked, and therefore discovers that the account exists and that it was seen.

## Decision

**There are two actions, with opposite consequences in reversibility and identical ones in silence.**

1. **Discarding** is reversible, and it is what the "Never" button becomes. The request leaves the inbox and goes to a trash with a deadline (SHOULD: 30 days), from which the person can restore it. Once the deadline passes, it is gone for good. It solves regret without inventing any notice for the other side.

2. **Blocking** is explicit, permanent and separate. Knocks from that issuer start being discarded on arrival, without entering quarantine. It is the answer to whoever insists, and never the default: blocking someone who merely got the address wrong is a cost the person should not pay by accident.

3. **The answer to the issuer is IDENTICAL in all four cases**: discarded, blocked, non-existent address, and account that never existed. It is the same `unknown-recipient` that EW-RFC 0003 §7.4 already requires, and it is what stops the channel from becoming an oracle. The mechanism already exists in the host, as `accepts_new=0` for a relationship address; what is missing is for it to hold for the main handle.

4. **The block list is visible and reversible**, under "Who may send to me". A list you cannot see is a list that decides for you, and the day someone needs to unblock will be a day when they do not remember blocking.

5. **Neither action notifies anyone.** Not the discarded party, not the blocked one, not the unblocked one. The system produces a read receipt nowhere, and refuses to produce one here too.

## Alternatives considered

**Keep it as it is: refusing deletes, and that is that.** Rejected because of the three problems in the context. The heaviest is regret: a wrong click in a list of silent requests is the kind of mistake that happens, and today there is no way back.

**Blocking only, no trash.** Rejected. It solves insistence and does not solve the mistake, which is the more common case. And it pushes people to block as a precaution, because discarding would remain irreversible: the design would teach the use of the more severe action.

**Trash only, no blocking.** Rejected. It solves the mistake and leaves the insistent party unanswered. The trash would become a place where the same company reappears every week, and the person would learn to ignore the whole inbox, which is exactly what silent quarantine exists to prevent.

**Notify whoever was blocked.** Rejected, and this is not a matter of preference: it turns the channel into an oracle. Whoever receives "you have been blocked" learns that the account exists, that someone read it, and that the address belongs to a real person. Without notice, whoever knocks cannot distinguish a block from a non-existent account, which is the whole property of ADR-0008.

**Block by address instead of by identity.** Rejected as the only path: whoever insists changes address, and blocking by address already exists (`accepts_new`). The two coexist, and the identity one is what answers insistence.

**Trash with no deadline.** Rejected. A trash that never empties is a permanent archive of whoever you refused, assembled without you asking for it, and it is precisely the accumulation that ADR-0008 avoids elsewhere. Thirty days covers regret, which is a matter of hours or days, not years.

## Consequences

- EW-RFC 0007 gains the section it was missing, and EW-RFC 0003 §7.4 comes to be cited by it: the equality of the answers is a normative requirement, not an implementation detail.
- The host gains a table of discarded requests with a deadline and one of blocked issuers, and the knock path comes to consult the second. Consulting is one line; the part that demands care is the answer staying identical.
- The interface swaps "Never" for "Discard", because the old label promised what it did not deliver, and gains "Block" as a separate action, with the consequence dialogue this product uses for everything that is hard to undo.
- **The trash is local state, and it does not travel.** A host that implements it differently, or does not implement it, stays interoperable: whoever knocks cannot tell, which is the requirement.
- It remains open what to do when a blocked issuer is the same one that already holds a granted relationship through another address. The provisional reading is that blocking the identity does not cut existing relationships, because cutting is a different action with a different consequence, and mixing the two would make one click do two things.
