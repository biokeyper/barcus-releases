# Running an observer node

An observer syncs the full chain, verifies every block, and serves local RPC. It holds no
consensus role, needs no funds and no permission — it is the foundation every other role
builds on, so set one up first even if your goal is mining or attesting.

## Prerequisites

- Linux x86-64 (glibc). ~2 GB RAM to start; disk grows with the chain.
- The verified binary from this repository's [releases](../../../releases) — see the README's
  *Downloading and verifying* section.
- A **bootstrap multiaddr** from a network operator (open an issue here to request one).

## Start

```sh
BARCUS_TRANSPORT=libp2p \
BARCUS_BOOTSTRAP=/dns4/<bootstrap-host>/tcp/7400/p2p/<PeerId> \
BARCUS_LISTEN=0.0.0.0:7400 \
BARCUS_DATA_DIR=$HOME/barcus/data \
BARCUS_HTTP_PORT=8545 \
./barcus-node-linux-x86_64 0 6 7400 8500
```

### The positional arguments: `<index> <total> <gossip_port> [rpc_port]`

| Arg | Meaning |
|---|---|
| `index` | This node's slot in the genesis layout. For any node that is not a genesis validator, use `0`. |
| `total` | **Consensus-critical.** The network's *genesis* validator count — currently **6** for devnet-2. A seat added later by governance is *not* part of this number. A wrong value builds a different genesis: your node runs happily and never agrees with anyone. |
| `gossip_port` | The libp2p listen port (conventionally 7400). |
| `rpc_port` | *Optional.* Starts the **binary RPC** listener on this port (loopback by default). Role daemons — miner, PRE, attester — connect through it, so pass it (e.g. `8500`) if you plan to run one. Omit it for a pure observer. |

### Environment

| Variable | What |
|---|---|
| `BARCUS_TRANSPORT=libp2p` | Required — selects the production transport. |
| `BARCUS_BOOTSTRAP` | One reachable peer's multiaddr. One is the entire join; discovery finds the rest. |
| `BARCUS_LISTEN` | Bind address for gossip (default port from the args). |
| `BARCUS_DATA_DIR` | Where the chain database lives. |
| `BARCUS_HTTP_PORT` | JSON-RPC over HTTP (loopback by default). |
| `BARCUS_KEYSTORE` | *Optional for an observer* — a signing key file (see the README's *Keys* section). Set it now if this node will ever be more than an observer. |
| `BARCUS_RPC_HOST` | *Optional.* Bind host for the binary RPC listener (default loopback). It is unauthenticated — if you bind it off-loopback, put a firewall in front. |

## Verify it worked

1. **First boot prints `net=<8 hex>`** — it must match the network id in the latest release
   notes. A mismatch means your genesis inputs (usually `<total>`) are wrong.
2. **It syncs**: `curl -s -X POST http://127.0.0.1:8545 -H 'content-type: application/json' -d '{"jsonrpc":"2.0","id":1,"method":"podo_getHead","params":{}}'` — the height should climb,
   and once caught up it should match the public gateway (`https://rpc.biokeyper.com`, same call).
3. **Chain identity**: `podo_getChainId` must return the id from the release notes.

## Keep it running (systemd)

```ini
# ~/.config/systemd/user/barcus-observer.service
[Unit]
Description=Barcus observer node
After=network-online.target
Wants=network-online.target

[Service]
Environment=BARCUS_TRANSPORT=libp2p
Environment=BARCUS_BOOTSTRAP=/dns4/<bootstrap-host>/tcp/7400/p2p/<PeerId>
Environment=BARCUS_LISTEN=0.0.0.0:7400
Environment=BARCUS_DATA_DIR=%h/barcus/data
Environment=BARCUS_HTTP_PORT=8545
ExecStart=%h/barcus/barcus-node-linux-x86_64 0 6 7400 8500
Restart=on-failure

[Install]
WantedBy=default.target
```

```sh
systemctl --user daemon-reload
systemctl --user enable --now barcus-observer
loginctl enable-linger $USER   # keep it alive after logout / across reboots
```

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Stays at height 0, wrong `net=` | Wrong `<total>` or genesis inputs — re-read the args table. |
| Connects, peer count stays 0 | Your firewall. Allow the gossip port (7400) locally; "connection refused on a port the node listens on" is almost always ufw/iptables on your own machine. |
| Height climbs then stalls | Check the release notes for a regenesis announcement — after a regenesis, delete `BARCUS_DATA_DIR` and resync from the new genesis. |
