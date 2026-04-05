# Running Two Block Producers on a Single Koinos Node

This guide walks through the entire process of setting up a Koinos node with two independent block producers on the same machine — from installing Docker and syncing the chain, to burning KOIN for VHP, registering producer keys, and configuring the second producer. Both producers share the node's core infrastructure (chain, mempool, p2p, AMQP) but produce blocks with different addresses and keys.

---

## Table of Contents

1. [Background: How Koinos Block Production Works](#1-background-how-koinos-block-production-works)
2. [Install Docker and Docker Compose](#2-install-docker-and-docker-compose)
3. [Set Up and Sync a Koinos Node](#3-set-up-and-sync-a-koinos-node)
4. [Install kcli](#4-install-kcli)
5. [Prepare Two Producer Addresses with VHP](#5-prepare-two-producer-addresses-with-vhp)
6. [Configure the First Block Producer](#6-configure-the-first-block-producer)
7. [Add the Second Block Producer](#7-add-the-second-block-producer)
8. [Start and Verify](#8-start-and-verify)
9. [Architecture Overview](#9-architecture-overview)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Background: How Koinos Block Production Works

Koinos uses **Proof of Burn (PoB)** as its consensus mechanism. Instead of staking tokens or spending electricity, block producers **burn KOIN** (the native token) to receive **VHP** (Virtual Hash Power). VHP is a non-transferable token that represents your mining power.

- The more VHP an address holds, the more frequently it wins the right to produce a block.
- When a block is produced, a small amount of VHP is consumed and the producer earns new KOIN as a reward.
- Over time, VHP slowly depletes as blocks are produced, so producers periodically burn more KOIN to maintain their VHP.

Each block producer needs three things:

1. A **producer address** — a Koinos address holding VHP
2. A **signing key** — a private key used to sign produced blocks
3. A **producer key registration** — an on-chain record linking the signing public key to the producer address

The signing key and the address owner key can be different. This allows you to keep the key that controls your funds in cold storage while using a separate "hot" key for block signing.

---

## 2. Install Docker and Docker Compose

Koinos runs as a set of Docker containers orchestrated by Docker Compose. You need **Docker Engine** and **Compose V2**.

### Ubuntu / Debian

```bash
# Remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc 2>/dev/null

# Install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Add Docker's official GPG key and repository
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine and Compose plugin
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Verify
docker --version
docker compose version
```

### Other platforms

See the official [Docker installation docs](https://docs.docker.com/engine/install/).

### Post-install (optional)

To run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
# Log out and back in for the group change to take effect
```

---

## 3. Set Up and Sync a Koinos Node

### Clone the Koinos repository

```bash
git clone https://github.com/koinos/koinos.git ~/koinos
cd ~/koinos
```

### Configure the node

Copy the example configuration files:

```bash
cp -r config-example config
cp env.example .env
```

This creates:

```
~/koinos/
├── docker-compose.yml       # Service definitions
├── .env                     # Environment variables (BASEDIR, image tags, ports)
└── config/
    ├── config.yml           # Microservice configuration
    ├── genesis_data.json    # Genesis block (never modify)
    ├── koinos_descriptors.pb # Protocol Buffers definitions for JSON-RPC
    └── rabbitmq.conf        # RabbitMQ settings
```

The default `.env` sets `BASEDIR=~/.koinos` — this is where all chain data, keys, and logs are stored on the host.

### Review `.env`

```bash
BASEDIR=~/.koinos

# Uncomment and adjust if you need to expose ports externally:
# P2P_INTERFACE=0.0.0.0
# P2P_PORT=8888
# JSONRPC_INTERFACE=0.0.0.0
# JSONRPC_PORT=8080

# Enable profiles for the services you need:
# COMPOSE_PROFILES=block_producer,jsonrpc

# Image tags (update these as new versions are released):
BLOCK_PRODUCER_TAG=v1.3.0
CHAIN_TAG=v1.5.0
MEMPOOL_TAG=v1.5.0
P2P_TAG=v1.3.0
# ... etc
```

### Start the core node (without block production)

First, sync the chain. This can take hours or days depending on your hardware and network:

```bash
docker compose up -d
```

This starts the core services: `amqp`, `chain`, `mempool`, `block_store`, and `p2p`.

### Monitor sync progress

```bash
docker compose logs --tail 5 --follow
```

You can also enable the JSON-RPC API to check sync status:

```bash
docker compose --profile jsonrpc up -d
```

Then query the current head block:

```bash
curl -s http://localhost:8080/ \
  -d '{"jsonrpc":"2.0","method":"chain.get_head_info","params":{},"id":1}' | \
  python3 -m json.tool
```

Compare the `head_topology.height` with a public explorer like [Koinos Blocks](https://koinosblocks.com) to see how far behind you are.

### Wait for full sync

**Do not start block production until the node is fully synced.** A node that is behind will produce blocks on old state, which will be rejected by the network.

---

## 4. Install kcli

[kcli](https://github.com/pgarciagon/kcli) is a command-line tool for interacting with the Koinos blockchain. You'll use it to check balances, burn KOIN, and register producer keys.

### Install via npm

```bash
npm install -g @pgarciagon/kcli
```

### Configure kcli to use your local node

By default, kcli connects to `https://api.koinos.io`. To use your local JSON-RPC:

```bash
kcli config --rpc http://localhost:8080
```

Or pass it per command:

```bash
kcli -r http://localhost:8080 chain-info
```

### Verify it works

```bash
kcli chain-info
```

This should return the current head block height and chain ID.

---

## 5. Prepare Two Producer Addresses with VHP

You need two Koinos addresses, each with VHP. There are several ways to obtain them:

### Option A: Generate new wallets

```bash
# Generate two new wallets
kcli generate-wallet
kcli generate-wallet
```

Each command outputs a private key (WIF format) and its corresponding address. **Save these securely — if you lose the private key, you lose the funds.**

### Option B: Use existing wallets

If you already have Koinos addresses (e.g., from [Kondor wallet](https://chrome.google.com/webstore/detail/kondor/ghipkefkpgkladckmlmdnadmcchefhjl)), you can use their private keys.

### Option C: Use a Fogata mining pool address

[Fogata](https://fogata.io) pools are smart contracts that aggregate VHP from multiple participants. If you operate a Fogata pool, the pool contract address is your producer address. The pool manages its own VHP — you don't burn KOIN yourself.

### Fund the addresses with KOIN

Each address needs KOIN to burn for VHP. You can:

- Transfer KOIN from an exchange (e.g., MEXC, KoinDX)
- Transfer from another wallet
- Receive from another user

### Burn KOIN to get VHP

For each producer address, burn KOIN to convert it to VHP:

```bash
# Import the wallet
kcli import-wallet <PRIVATE_KEY_WIF>

# Check current balance
kcli balance

# Burn KOIN to VHP (specify amount or percentage)
kcli burn --amount 10000       # Burn exactly 10,000 KOIN
kcli burn --percentage 80      # Burn 80% of available KOIN
```

Repeat for the second address:

```bash
kcli delete-wallet
kcli import-wallet <SECOND_PRIVATE_KEY_WIF>
kcli burn --amount 10000
```

### Verify VHP balances

```bash
kcli balance <FIRST_ADDRESS>
kcli balance <SECOND_ADDRESS>
```

You should see non-zero VHP for both addresses. Example output:

```
Balances for 1Aaa...:
   KOIN: 2000.00000000
   VHP:  10000.00000000
   Mana: 2000.00000000
```

> **How much VHP do you need?** There's no minimum, but with very little VHP you'll rarely win blocks. As a rough guide, with the total network VHP around 5,000,000, having 100,000 VHP (~2% of the network) means you'd produce roughly 2% of all blocks.

---

## 6. Configure the First Block Producer

### Generate the signing key

If you want to use a separate key for block signing (recommended for security):

```bash
kcli generate-wallet
```

Save the private key — this will be your signing key. If you prefer to use the same key that owns the address, you can skip this step.

### Register the signing key on-chain

The signing public key must be registered on-chain so the network knows which key is authorized to sign blocks for your producer address:

```bash
# Import the wallet that OWNS the producer address (to authorize the registration)
kcli import-wallet <OWNER_PRIVATE_KEY_WIF>

# Register the signing key
# If using the same key as owner, just pass the address:
kcli register-producer-key <FIRST_PRODUCER_ADDRESS>

# If using a separate signing key, pass the public key explicitly:
kcli register-producer-key <FIRST_PRODUCER_ADDRESS> <SIGNING_PUBLIC_KEY_BASE64URL>
```

### Verify registration

```bash
kcli get-producer-key <FIRST_PRODUCER_ADDRESS>
```

Should output:

```
Registered Producer Public Key:
   Producer Address: 1Aaa...
   Public Key: Axxx...
   Key Length: 33 bytes
```

### Place the signing private key

```bash
echo "<SIGNING_PRIVATE_KEY_WIF>" > ~/.koinos/block_producer/private.key
chmod 600 ~/.koinos/block_producer/private.key
```

### Configure `config.yml`

Edit `~/koinos/config/config.yml` and set the producer address:

```yaml
block_producer:
  algorithm: pob
  producer: <FIRST_PRODUCER_ADDRESS>
```

### Start the first producer

```bash
docker compose --profile block_producer up -d
```

### Verify it's working

```bash
docker compose logs --tail 20 block_producer
```

You should see:

```
Koinos Block Producer v1.3.0
Public address: 1Xxx...          ← derived from private.key
Public key: Axxx...              ← must match on-chain registration
Producer address: 1Aaa...       ← your first address with VHP
...
Producing with 10000.00000000 VHP
...
Produced block - Height: ..., ID: 0x1220...
```

If you see `proof failed vrf validation`, the private key doesn't match the registered public key. See [Troubleshooting](#10-troubleshooting).

---

## 7. Add the Second Block Producer

Now that the first producer is working, add the second one.

### 7.1 Register the second producer's signing key

Same process as the first producer:

```bash
# Import the wallet that owns the second producer address
kcli delete-wallet
kcli import-wallet <SECOND_OWNER_PRIVATE_KEY_WIF>

# Register the signing key
kcli register-producer-key <SECOND_PRODUCER_ADDRESS> <SECOND_SIGNING_PUBLIC_KEY_BASE64URL>

# Verify
kcli get-producer-key <SECOND_PRODUCER_ADDRESS>
```

> **Fogata Pools:** If the second address is a Fogata pool, skip this step. The pool contract manages its own producer key. Check the registered key with `kcli get-producer-key <POOL_ADDRESS>` and make sure you have the corresponding private key.

### 7.2 Create the data directory

```bash
mkdir -p ~/.koinos_2/block_producer
```

### 7.3 Place the signing private key

The private key here **must** correspond to the public key registered on-chain for the second producer address:

```bash
echo "<SECOND_SIGNING_PRIVATE_KEY_WIF>" > ~/.koinos_2/block_producer/private.key
chmod 600 ~/.koinos_2/block_producer/private.key
```

### 7.4 Add the service to `docker-compose.yml`

The default `docker-compose.yml` from the [Koinos repository](https://github.com/koinos/koinos) only includes one `block_producer` service. Add a `block_producer_2` service right after it:

```yaml
   block_producer_2:
      image: koinos/koinos-block-producer:${BLOCK_PRODUCER_TAG:-latest}
      restart: always
      profiles: ["block_producer", "all"]
      depends_on:
         - amqp
         - mempool
         - chain
      configs:
         - source: koinos-config
           target: /koinos/config.yml
      volumes:
         - "${BASEDIR}_2:/koinos"
      command: --basedir=/koinos --producer=<SECOND_PRODUCER_ADDRESS>
```

Replace `<SECOND_PRODUCER_ADDRESS>` with the actual Koinos address.

### Why the `--producer` flag is required

Both Docker services run the same `koinos-block-producer` binary. Internally, the binary identifies itself as `block_producer` and reads only the `block_producer:` section from `config.yml` — it does **not** look for a `block_producer_2:` section, regardless of the Docker service name. Without the `--producer` flag, both instances would use the same producer address, causing VRF validation failures.

The `--producer` CLI flag overrides the address from the config file.

### Comparison of both services

| | `block_producer` | `block_producer_2` |
|---|---|---|
| **Data volume** | `${BASEDIR}` (~/.koinos) | `${BASEDIR}_2` (~/.koinos_2) |
| **Producer address** | From `config.yml` | From `--producer` flag |
| **Private key** | `~/.koinos/block_producer/private.key` | `~/.koinos_2/block_producer/private.key` |

---

## 8. Start and Verify

### Start both producers

Both producers share the `block_producer` profile, so they start together:

```bash
docker compose --profile block_producer up -d
```

Or set `COMPOSE_PROFILES` in your `.env`:

```bash
# .env
COMPOSE_PROFILES=block_producer,jsonrpc
```

```bash
docker compose up -d
```

### Check the logs

```bash
docker compose logs --tail 20 block_producer
docker compose logs --tail 20 block_producer_2
```

For each producer, verify:

1. **Producer address** — shows the correct, distinct address
2. **Public key** — matches the on-chain registration
3. **"Producing with X VHP"** — reflects that address's VHP, not the other producer's
4. **"Produced block"** messages appear (may take minutes, depending on VHP amount)

### Check VHP balances

```bash
kcli balance <FIRST_PRODUCER_ADDRESS>
kcli balance <SECOND_PRODUCER_ADDRESS>
```

### Monitor block production

```bash
# Follow logs for both producers
docker compose logs --follow block_producer block_producer_2
```

---

## 9. Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                    Docker Network                     │
│                                                      │
│  ┌──────────┐   ┌───────┐   ┌────────────────────┐  │
│  │ RabbitMQ │◄──│ chain │──►│    block_store     │  │
│  │  (AMQP)  │   └───┬───┘   └────────────────────┘  │
│  └────┬─────┘       │                                │
│       │         ┌───┴────┐                           │
│       │         │mempool │                           │
│       │         └───┬────┘                           │
│       │             │                                │
│  ┌────┴─────────────┴──────────────┐                 │
│  │                                 │                 │
│  ▼                                 ▼                 │
│  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │  block_producer  │  │    block_producer_2      │  │
│  │  ~/.koinos/      │  │    ~/.koinos_2/          │  │
│  │  Address A       │  │    Address B             │  │
│  │  Key: key_a      │  │    Key: key_b            │  │
│  └──────────────────┘  └──────────────────────────┘  │
│                                                      │
│  ┌─────┐  ┌────────┐  ┌──────┐  ┌────────────────┐  │
│  │ p2p │  │jsonrpc │  │ grpc │  │     rest       │  │
│  └─────┘  └────────┘  └──────┘  └────────────────┘  │
└──────────────────────────────────────────────────────┘
```

The Koinos node is a collection of microservices communicating through RabbitMQ (AMQP). The core services — chain, mempool, block_store, and p2p — form the backbone. Block producers are optional services that connect to this backbone.

Adding a second block producer is lightweight: it connects to the same AMQP bus and shares the same chain state, mempool, and p2p network. Each producer independently competes for blocks using its own VHP. There is no conflict — they propose blocks signed with different keys for different addresses.

Optional API services (jsonrpc, grpc, rest, transaction_store, account_history, contract_meta_store) are independent of block production and can be enabled or disabled via Docker Compose profiles.

---

## 10. Troubleshooting

### `proof failed vrf validation`

**Cause:** The private key on disk does not match the public key registered on-chain for that producer address.

**Diagnose:**

```bash
# What key is registered on-chain?
kcli get-producer-key <PRODUCER_ADDRESS>

# What key is the producer using?
docker compose logs block_producer_2 | grep "Public key"

# Or check the file directly
cat ~/.koinos_2/block_producer/public.key
```

The "Public key" from the logs and the on-chain key must be identical.

**Fix:** Either:
- Replace `private.key` with the one that matches the on-chain registration, or
- Register the new public key on-chain: `kcli register-producer-key <ADDRESS> <PUBLIC_KEY>`

### Both producers show the same address

**Cause:** The `--producer` flag is missing from the `command` in `docker-compose.yml` for `block_producer_2`.

**Fix:** Add `--producer=<SECOND_ADDRESS>` to the command:

```yaml
command: --basedir=/koinos --producer=<SECOND_PRODUCER_ADDRESS>
```

Then recreate the container:

```bash
docker compose --profile block_producer up -d block_producer_2
```

### "Producing with X VHP" shows wrong amount

**Cause:** Same as above — the producer is reading the wrong address. Verify the `--producer` flag and check the "Producer address" line in the logs.

### Producer never produces blocks

**Possible causes:**

- **VHP too low:** With very little VHP relative to the network total, blocks are won infrequently. Check `kcli balance <ADDRESS>` and compare your VHP to the network total shown in `Estimated total VHP producing` in the logs.
- **Node not synced:** The producer won't produce until the chain is synced. Check `docker compose logs chain` for sync progress.
- **Gossip not ready:** By default, the producer waits for p2p gossip to be active before producing. Check `docker compose logs p2p` for peer connections.

### Containers can't connect to AMQP (`i/o timeout`)

**Cause:** On systems running Docker with the nf_tables iptables backend (common on Linux kernels 6.x+), inter-container networking can break due to bridge netfilter incorrectly filtering intra-bridge traffic.

**Diagnose:**

```bash
# Check iptables backend
iptables --version
# If it says "(nf_tables)", you may be affected

# Test inter-container connectivity
docker exec koinos-p2p-1 ping -c1 -W2 amqp
```

**Fix:**

```bash
sudo sysctl -w net.bridge.bridge-nf-call-iptables=0
sudo sysctl -w net.bridge.bridge-nf-call-ip6tables=0
```

To make it persistent:

```bash
sudo cat > /etc/sysctl.d/99-docker-bridge.conf << 'EOF'
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
EOF
```

Then restart Docker and the stack:

```bash
docker compose --profile block_producer down
sudo systemctl restart docker
docker compose --profile block_producer up -d
```

---

## Quick Reference

| | First Producer | Second Producer |
|---|---|---|
| Docker service | `block_producer` | `block_producer_2` |
| Data directory | `~/.koinos` | `~/.koinos_2` |
| Private key | `~/.koinos/block_producer/private.key` | `~/.koinos_2/block_producer/private.key` |
| Address source | `config.yml` → `block_producer.producer` | `--producer` CLI flag |
| Profile | `block_producer` | `block_producer` (shared) |

### Minimal checklist

1. Both addresses have VHP (`kcli balance <ADDRESS>`)
2. Signing keys registered on-chain (`kcli get-producer-key <ADDRESS>`)
3. Correct `private.key` in each data directory
4. `block_producer_2` added to `docker-compose.yml` with `--producer` flag
5. `docker compose --profile block_producer up -d`
6. Both producers show distinct addresses and "Produced block" in logs
