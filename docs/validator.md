# Becoming a validator

A validator holds a committee seat, authors and certifies DAG batches, and votes in
governance. Barcus is **not** proof-of-stake: validators post no bond, and voting weight
inside the committee comes from PoDO (membership + verified data custody), not capital.

## 1. Run a synced node with a keystore

Follow the [observer guide](observer.md) first, with one addition — a signing key:

```sh
./barcus-node-linux-x86_64 keygen | head -1 | cut -d' ' -f2 > node.key
chmod 600 node.key
```

Add `Environment=BARCUS_KEYSTORE=%h/barcus/node.key` to the unit and let the node sync to
the tip. The address printed by `keygen` is your validator identity — guard the seed file;
whoever holds it *is* the address, and there is no recovery.

## 2. Declare candidacy — no vote, no human in the loop

Admission is a fully **automated candidacy pipeline** — no governance vote, no operator
seats you by hand:

1. **Declare** with a `CandidacyDeclare` transaction signed by your validator key. It is
   free (validators post no bond, ever) and permissionless — it just enters you into the
   candidate pool. Refused only if your address holds a conflicting role or was permanently
   barred (provable equivocation, or a governance ejection).
2. **Prove presence** with a `ValidatorPresence` transaction quoting a fresh chain head —
   this marks your synced node *ready*. (Once seated you keep proving presence through
   normal DAG participation; the heartbeat is what keeps you eligible.)

That is the whole admission. From here the network promotes you on its own.

## 3. Promotion happens on its own

At the next **session boundary**, the network derives the active committee from the ready
candidate pool — deterministically, so every node agrees — and seats you. Nothing restarts,
yours or anyone else's.

- A candidate that is not yet ready (or whose node has gone dark) simply is not seated;
  it costs the network nothing until it is ready.
- Committee growth is **gradual and safe**: each boundary only admits as many newcomers as
  the continuing validators can still meet quorum for, so admission never stalls the chain.
  A large influx of candidates is absorbed over successive sessions, not in one jump.
- When the ready pool exceeds the committee-size cap, membership rotates each session over
  everyone eligible — unpredictable in advance, so no committee is targetable ahead of time.

Check yourself in `podo_getCandidates` (are you `ready`? `seated`?) and `podo_getValidators`
(once seated: `active`, an advancing `engineRound`, `lastCertifiedRound` tracking it).

Check yourself in `podo_getValidators` (public gateway or your own node): your entry shows
`active`, an advancing `engineRound`, and `lastCertifiedRound` tracking it.

## Operating expectations

- **Stay reachable.** Your gossip port must be reachable from the network; a validator that
  drops off is excluded from liveness accounting until it re-proves presence (this is
  recoverable — going dark is treated as an outage, not an offence).
- **Bandwidth matters more than CPU.** Consensus duty is a steady stream of certificate
  publishes with multi-hundred-KB bursts; a wired connection with ≥ 2 MiB/s *uplink* is a
  realistic floor. A starved uplink looks exactly like misbehaviour to the network's
  scoring — cable beats WiFi.
- **One address, one role.** A validator address cannot also be a miner, PRE node or
  attester; the chain refuses the second registration.

## Removal — what can end a seat

| Cause | Effect |
|---|---|
| Provable equivocation (signing conflicting batches) | Unseated by consensus rules. |
| Governance vote (`RemoveValidator`) | The committee's supermajority removes the seat. |
| Voluntary exit | A `CandidacyWithdraw` transaction — you leave the pool at the next session boundary; earnings you have accrued still pay out. |

Since validators post no bond, there is nothing to confiscate on exit — a removed seat
simply stops counting at the next boundary.
