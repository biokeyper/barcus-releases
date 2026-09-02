# Running an availability attester

An availability attester is the network's mystery shopper: it randomly samples storage
miners' datasets and files **availability reports** against miners that have gone silent.
A report is not a verdict — it *asks the chain a question it can settle*: the accused miner
is immediately challenged, and the existing proof path decides who was right. Both outcomes
cost someone, which is what keeps reports honest.

## Prerequisites

- A running, synced node with the **binary RPC enabled** ([observer guide](observer.md)
  with the 4th argument, e.g. `0 6 7400 8500`).
- A funded wallet: the daemon bonds **500 PODO** at registration, and each report it files
  stakes real funds on top. Devnet funding: the faucet (explorer Faucet page, or
  `FaucetClaim` — capped on-chain at 1,000,000 PODO per recipient per 7 days).

## Start

```sh
BARCUS_ROLE_SEED=0x<64 hex>              # or BARCUS_KEYSTORE=<file>
./barcus-node-linux-x86_64 attester 127.0.0.1:8500 30000
```

Arguments: `attester <rpc_host:port> [interval_ms]` — the sampling interval defaults to
30,000 ms.

### Environment

| Variable | What |
|---|---|
| `BARCUS_ROLE_SEED` / `BARCUS_KEYSTORE` | The funded signing wallet — this address is the attester identity. |
| `BARCUS_ATTESTER_FILE_REPORTS` | `0` = observe-only mode: sample and log, file nothing. Reporting is **on** by default — but this daemon spends real money, and an operator who sees it misfiring needs a switch faster than a rebuild. |

## What the daemon does for you

Registers on-chain (`AttesterRegister`), then loops: pick miner/dataset pairs → check when
each miner last proved custody → if a miner has been silent past the threshold, file an
availability report (staking funds), which triggers an immediate PoR challenge against the
accused.

- The miner **answers** the challenge ⇒ your report was wrong ⇒ **your stake is forfeited**.
- The miner **fails** it ⇒ the report stands ⇒ the miner is sanctioned and your stake
  returns with the reward.

Run in observe-only mode first and watch the logs until you trust its judgement on your
network view — a flaky connection *on your side* makes honest miners look silent.

## Economics, honestly

- **Earning**: upheld reports pay; attesters also take a share of storage-rent splits.
- **False reports cost you**: every wrong report forfeits its stake. There is no free
  accusation.
- **Leaving**: `AttesterDeregister` refunds the bond and removes the record — refused while
  any report you filed is still open (you cannot exit mid-adjudication), refused while you
  hold a live appraiser licence, and refused permanently with slash history.
- **Governance veto**: the validator committee can eject an attester by supermajority. The
  open-report and licence rules above still apply — a veto is not a door out of an
  adjudication — then your own `AttesterDeregister` pays the bond out; the address stays
  barred from re-registering the role.
- **One address, one role** — an attester may not also be a miner (nobody audits their own
  warehouse), a validator, or a PRE node.

## Verify

- Your address appears in `podo_getAttesters` (or the explorer), status `active`.
- The daemon logs each sampling round: pairs checked, challenges seen, reports filed.
