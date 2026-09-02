# Barcus — node releases

Official binary releases of `barcus-node`, the node software for **Barcus**: a Rust-native,
post-quantum layer-1 blockchain built on DAG-BFT consensus with **Proof of Data Ownership
(PoDO)** — validator weight comes from verified data custody and committee membership, not
from stake.

> **This repository contains binaries and documentation only.** The Barcus source code is
> not published here and is not licensed for redistribution. See [NOTICE](NOTICE.md).

## Release channels

Each branch is one network's channel — pick yours:

| Branch | Network | Status |
|---|---|---|
| **[`devnet`](../../tree/devnet)** | **devnet-2** (development network) | **live — start here** |
| [`testnet`](../../tree/testnet) | testnet | not launched yet |
| `main` | mainnet | not launched yet — this branch opens with it |

The [`devnet` branch](../../tree/devnet) carries the full documentation: download +
verification, quickstarts, and per-role setup guides (observer, validator, storage miner,
PRE node, availability attester).

Release tags are prefixed by channel (`devnet2-…`); each release's notes state the network
id it belongs to. **Always verify checksums** — see the channel branch's README.
