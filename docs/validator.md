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

## 2. Request a seat

On devnet-2, open an issue in this repository with your address. Admission is currently a
governance vote by the existing committee; it is moving to an **automated candidacy
pipeline** (register keys → declare candidacy on-chain → automatic promotion from the
candidate set at epoch boundaries), at which point this step becomes a transaction, not a
request.

## 3. Promotion happens on its own

Once seated, nothing restarts — yours or anyone else's:

- Your node keeps syncing; the network observes it **demonstrating presence** (authoring
  or acknowledging in the DAG).
- At the next epoch boundary after presence is demonstrated, the seat is promoted into the
  active committee and starts counting toward quorum.
- A seat whose node is not yet up costs the network nothing: it is not counted until ready.

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
| Voluntary exit | Ask an operator to propose removal (a `CandidacyWithdraw` transaction arrives with the candidacy pipeline). |

Since validators post no bond, there is nothing to confiscate on exit — a removed seat
simply stops counting at the next boundary.
