# Running a storage miner

A storage miner bonds capital, hosts dataset content in IPFS, and earns by answering
**Proof-of-Retrievability (PoR) challenges** — cryptographic spot-checks that it still
holds the actual bytes. In Barcus, the storage miner *is* the IPFS node operator: one
role, two daemons (the miner daemon + a local Kubo).

> **Read this first — external miners on devnet-2:** the devnet's IPFS swarm is *private*.
> Its `swarm.key` is not published and must be obtained from the operator (open an issue).
> Until you have it, your Kubo cannot fetch or serve dataset content and the miner daemon
> cannot do useful work. Every step below is still accurate; the swarm key is the one
> artifact this repository cannot hand you.

## Prerequisites

- A running, synced node with the **binary RPC enabled** — the [observer guide](observer.md)
  with the 4th argument, e.g. `0 6 7400 8500`. The miner daemon talks to `127.0.0.1:8500`.
- [Kubo](https://github.com/ipfs/kubo) (`ipfs`) installed, initialized, and running with the
  network's swarm key in place.
- A funded wallet. Mining takes a real bond: the chain's minimum is **1,000 PODO** for one
  capacity unit (1 TiB), scaling with declared capacity — and it **doubles for every past
  slash** on the address. On devnet, fund via the explorer's Faucet page or a `FaucetClaim`
  transaction (on-chain cap: 1,000,000 PODO per recipient per 7 days).

## Start

```sh
BARCUS_MINER_SEED=0x<64 hex>            # or BARCUS_KEYSTORE=<file>
BARCUS_IPFS_API=http://127.0.0.1:5001 \
BARCUS_MINER_CAPACITY_GB=20 \
BARCUS_MINER_JURISDICTION=UG \
./barcus-node-linux-x86_64 miner 127.0.0.1:8500
```

Arguments: `miner <rpc_host:port> [dataset_mb] [poll_ms]` — `dataset_mb` (default 256)
sizes the self-registered starter dataset; `poll_ms` (default 200) is the challenge poll
interval.

### Environment

| Variable | What |
|---|---|
| `BARCUS_MINER_SEED` / `BARCUS_KEYSTORE` | The funded signing wallet. This address becomes the miner identity. |
| `BARCUS_IPFS_API` | Your Kubo API endpoint. Without it the daemon runs but cannot pin or serve content. |
| `BARCUS_MINER_CAPACITY_GB` | Your declared ceiling in GiB (default 20). The chain records `min(this, your store's measured share)` — you cannot advertise space you do not have. |
| `BARCUS_MINER_JURISDICTION` | ISO 3166-1 alpha-2 country code of where your disks are (default `UG`). Self-declared; datasets with a jurisdiction requirement are only assigned to matching offers, and country diversity affects replica placement. |
| `BARCUS_MINER_OFFER_REVISE_SECS` | How often the daemon refreshes its standing offer (default 60). |

## What the daemon does for you

On startup it registers the miner (`StorageMinerRegister`, debiting the bond), publishes a
**standing offer** (capacity, price floor, jurisdiction, accepted data classes), and then
loops forever: watch for PoR challenges on datasets assigned to you → read the challenged
bytes from your store → answer with a Merkle proof before the deadline.

The **offer is your consent**: the protocol assigns datasets strictly within what you
declared. Withdraw the offer (`accepting: false`) to stop *new* assignments — this never
sheds liability for data already assigned to you.

## Economics, honestly

- **Earning**: PoR rewards are drawn from each dataset's collateral; your offer's
  `min_collateral_per_gib` is your price floor.
- **Missing challenges costs you**: repeated misses slash the bond; enough of them
  blacklist the address, which locks the remaining bond permanently.
- **Bonds escalate**: a re-registration after a slash pays `min_bond × 2^slash_count`.
- **Leaving**: `StorageMinerDeregister` with nothing assigned and a clean record refunds
  the bond in full and removes the record. With data still in custody the exit waits until
  your replicas are reassigned — a miner may never strand data someone paid to store.
- **Governance veto**: the validator committee can eject a miner by supermajority (a
  policy decision, no evidence required). Ejection is not confiscation — your bond still
  pays out through your own deregister once custody drains — but the address is
  permanently barred from re-registering the role.

## Verify

- Your miner appears in the explorer's storage page / `podo_getMiners`, status `active`.
- The daemon logs each answered challenge; the chain's record of you shows
  `missedChallenges` staying at 0.
