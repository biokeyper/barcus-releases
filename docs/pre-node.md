# Running a PRE node

A PRE (proxy re-encryption) node holds **key fragments** for leased datasets and
re-encrypts them to paying lessees — without ever seeing the plaintext. Leases select a
k-of-n threshold of PRE nodes, so no single node can decrypt anything alone. The node
serves a small HTTP API that lessees dial directly.

## Prerequisites

- A running, synced node with the **binary RPC enabled** ([observer guide](observer.md)
  with the 4th argument, e.g. `0 6 7400 8500`).
- A funded wallet: the daemon bonds **500 PODO** at registration. On devnet, fund via the
  faucet (explorer Faucet page, or `FaucetClaim` — capped on-chain at 1,000,000 PODO per
  recipient per 7 days).
- A **publicly reachable HTTP endpoint** — this is the part most operators miss. Your
  API URL is published *on the chain* for lessees to dial; an unreachable one fails
  silently (registration succeeds, requests simply never arrive).

## Start

```sh
BARCUS_ROLE_SEED=0x<64 hex>              # or BARCUS_KEYSTORE=<file>
BARCUS_PRE_ANNOUNCE=https://pre.example.org \
./barcus-node-linux-x86_64 pre 127.0.0.1:8500 8570
```

Arguments: `pre <rpc_host:port> [http_port]` — `http_port` (default 8570) is where the
re-encryption API listens.

### Environment

| Variable | What |
|---|---|
| `BARCUS_ROLE_SEED` / `BARCUS_KEYSTORE` | The funded signing wallet — this address is the PRE identity. |
| `BARCUS_PRE_ANNOUNCE` | The address lessees dial, **published on-chain**. Either a full URL (`https://pre.example.org` — for a node behind a tunnel or reverse proxy) or a bare host/IP, which is expanded with the http port. If unset, the daemon announces loopback — fine only when everything runs on one host. |

## What the daemon does for you

- Derives a real **ML-KEM-768** keypair from your seed and registers on-chain
  (`PreRegister`: bond, public key, announce URL). If the chain already has your address
  under an older key or URL, it rotates the record automatically — the bond is untouched.
- Serves the re-encryption API: when a lease names you, the lessee presents its request,
  and your node re-encrypts its key fragment to the lessee's key.

## Economics, honestly

- **Earning**: leases carry a PRE share of the lease payment, split across the selected
  nodes.
- **Duty**: while a live lease names you, the lessee's paid recoverability depends on your
  availability. The chain holds you to it (below).
- **Leaving**: `PreDeregister` refunds the bond in full and removes the record — but is
  refused while any lease naming you is neither acknowledged nor expired, and refused
  permanently if the address has slash history. Plan exits around your lease book.
- **Slashing**: a council-adjudicated slash forfeits the bond and bars the usual exits.
- **Governance veto**: the validator committee can eject a PRE node by supermajority.
  Live leases still run to acknowledgement or expiry first; then your own `PreDeregister`
  pays the bond out. The address stays barred from re-registering the role.
- **One address, one role** — a PRE node may not also be a miner (the warehouse must not
  hold the keys), a validator, or an attester.

## Verify

- Your node appears in `podo_getPreNodes` (or the explorer) with your announce URL and a
  1184-byte ML-KEM-768 public key, status `active`.
- From a machine that is *not* yours: `curl <your announce URL>/health` (or simply confirm
  the port answers). If only you can reach it, lessees can't.
