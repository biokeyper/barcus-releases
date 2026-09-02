# Barcus — node releases

Official binary releases of `barcus-node`, the node software for **Barcus**: a Rust-native,
post-quantum layer-1 blockchain built on DAG-BFT consensus (Narwhal/Bullshark-style) with
**Proof of Data Ownership (PoDO)** — validator weight comes from verified data custody and
committee membership, not from stake.

> **This repository contains binaries and documentation only.** The Barcus source code is
> not published here and is not licensed for redistribution. See [NOTICE](NOTICE.md).

---

## What is Barcus?

| | |
|---|---|
| Consensus | DAG-BFT (certified batches, 75% quorum, epoch-pinned committees) |
| Sybil resistance | PoDO — membership + data custody, **not** proof-of-stake |
| Signatures | Post-quantum **ML-DSA-65** (FIPS 204); ML-KEM-768 for encryption |
| Native asset | PODO (18 decimals; base unit "Tete") |
| Networking | libp2p — gossipsub consensus, Kademlia discovery, request-response sync |
| Extras | Native multi-asset ledger, RWA registry, storage-miner market (IPFS), optional EVM |

The current public network is **devnet-2** — a development network. Balances and state carry
no value and the chain may be reset (a "regenesis") with notice in the release entry.

## Downloading and verifying

Every release ships:

- `barcus-node-linux-x86_64` — the node binary (Linux, x86-64, glibc; stripped)
- `SHA256SUMS` — checksums for every asset

```sh
curl -LO https://github.com/biokeyper/barcus-releases/releases/latest/download/barcus-node-linux-x86_64
curl -LO https://github.com/biokeyper/barcus-releases/releases/latest/download/SHA256SUMS
sha256sum -c SHA256SUMS
chmod +x barcus-node-linux-x86_64
./barcus-node-linux-x86_64 version
```

**Always verify the checksum.** A node's signatures are only as trustworthy as the binary
that produced them.

## Quickstart — read-only access (no node)

The devnet's public RPC gateway:

```sh
curl -s -X POST https://rpc.biokeyper.com \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"podo_getHead","params":{}}'
```

Useful methods: `podo_getHead`, `podo_getBlockByHeight`, `podo_getBalance`,
`podo_getChainId`, `podo_getValidators`, `podo_getProposals` (supports `{limit, offset}`).

## Quickstart — run an observer node

An observer syncs the full chain and serves local RPC, with no consensus role. You need a
**bootstrap multiaddr** from a network operator (see *Getting connected* below).

```sh
BARCUS_TRANSPORT=libp2p \
BARCUS_BOOTSTRAP=/dns4/<bootstrap-host>/tcp/7400/p2p/<PeerId> \
BARCUS_LISTEN=0.0.0.0:7400 \
BARCUS_DATA_DIR=$HOME/barcus/data \
BARCUS_HTTP_PORT=8545 \
./barcus-node-linux-x86_64 0 6 7400
```

- The trailing arguments are `<index> <total> <port>`. **`<total>` is consensus-critical**:
  it must equal the network's *genesis* validator count (currently **6** for devnet-2 — a
  seat added later by governance is *not* part of this number). A wrong value builds a
  different genesis: your node runs happily and never agrees with anyone.
- First boot prints a line containing `net=<8 hex>` — it must match the current network id
  published in the release notes. A mismatch means your genesis inputs are wrong.
- Sync is automatic: one reachable bootstrap address is the entire join.

## Keys and the keystore

Every role signs with an ML-DSA-65 key derived from a 32-byte seed. Generate one:

```sh
./barcus-node-linux-x86_64 keygen
# seed:    0x<64 hex chars>     ← the secret. The address is derived from it.
# address: 0x<40 hex chars>
```

A **keystore is just that seed in a file** — plain text, 64 hex chars (with or without
`0x`), nothing else:

```sh
./barcus-node-linux-x86_64 keygen | head -1 | cut -d' ' -f2 > node.key
chmod 600 node.key
BARCUS_KEYSTORE=$PWD/node.key ./barcus-node-linux-x86_64 …
```

Guard the file: whoever holds the seed **is** the address. There is no recovery.

## Roles — what this one binary can run

The same executable runs every network role as a subcommand. What each needs:

| Role | Start | Needs | Entry |
|---|---|---|---|
| **Observer** | `barcus-node 0 6 7400` + a bootstrap | nothing | permissionless |
| **Validator** | same, with `BARCUS_KEYSTORE` | a committee seat | governance vote (see below) |
| **Storage miner** | `barcus-node miner <rpc_host:port>` | funded wallet (bond), a local Kubo (`ipfs`) daemon, and the network's private-swarm key | permissionless tx (`StorageMinerRegister`) — but see the note |
| **PRE node** | `barcus-node pre <rpc_host:port>` | funded wallet (bond), a reachable announce URL | permissionless tx (`PreRegister`) |
| **Availability attester** | `barcus-node attester <rpc_host:port>` | funded wallet (bond) | permissionless tx (`AttesterRegister`) |

Role daemons take their signing wallet from `BARCUS_ROLE_SEED=0x<64 hex>` or
`BARCUS_KEYSTORE=<file>`. Fund a devnet wallet via the faucet (the explorer's Faucet page,
or a `FaucetClaim` transaction). **One address, one role** — the chain refuses an address
that tries to hold two (a miner may not attest its own warehouse, a validator may not buy
weight with a miner, and so on).

> **External storage miners, honestly:** the devnet's IPFS swarm is *private*; its
> `swarm.key` is not published and must be obtained from the operator (open an issue). Until
> you have it, the miner daemon cannot fetch or serve dataset content. Every other role above
> works from this repository's artifacts alone.

## Becoming a validator

1. Generate a keystore (above) and run your node with `BARCUS_KEYSTORE=<file>`; let it sync
   to the tip.
2. Request a seat: on devnet-2, open an issue in this repository with your address.
   Validator admission is moving to an **automated candidacy pipeline** — register keys →
   declare candidacy on-chain → automatic promotion from the candidate set at epoch
   boundaries — at which point this step becomes a transaction, not a request.
3. Once seated, your node joins the committee at the next epoch boundary after it
   demonstrates presence — no restarts, yours or anyone else's.

## Transaction signing (SDK/tooling authors)

Since the 2026-09-01 regenesis, **transaction signatures are chain-bound**: the signing
domain is `"barcus-tx-v2:" ‖ <chain id>`, where the chain id is the genesis header hash.
Fetch it once via `podo_getChainId` and sign under it. A signature produced for any other
network — or any earlier genesis of this one — does not verify. Post-quantum envelope:
`algTag(1) ‖ ML-DSA-65 pubkey(1952) ‖ signature(3309)` over `SHA3-512(domain ‖ borsh(tx))`.

## Ports

| Port | What |
|---|---|
| 7400 (TCP) | libp2p gossip + sync (the only port another node needs to reach) |
| 8545 (TCP) | JSON-RPC over HTTP — binds loopback by default; it is unauthenticated, so put it behind a proxy or firewall if exposed |

If you run a firewall (ufw etc.), remember to allow the p2p port **on your own machine** —
"connection refused on a port the node is listening on" is almost always the local firewall.

## Getting connected

Devnet-2 bootstrap addresses are handed out by the network operator rather than published:
open an issue in this repository and one will be provided. The network id and any regenesis
announcements appear in each release's notes.

## Support

- **Issues** (join problems, verification failures, bootstrap requests): open an issue here.
- A node that connects but stays at height 0 and shows the wrong `net=`: your `<total>` or
  genesis inputs are wrong — see the observer quickstart.
