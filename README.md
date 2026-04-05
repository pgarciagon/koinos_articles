# Running Two Block Producers on a Single Koinos Node

This guide walks through configuring a standard [Koinos node](https://github.com/koinos/koinos) to run two independent block producers on the same machine. Both producers share the node's core infrastructure (chain, mempool, p2p, AMQP) but produce blocks with different addresses and keys.

## Prerequisites

- A synced Koinos node running via Docker Compose ([setup guide](https://github.com/koinos/koinos))
- Two Koinos addresses, each with VHP (KOIN burned via the Proof of Burn contract)
- The first producer already configured and working
- [kcli](https://github.com/pgarciagon/kcli) installed (optional, for verification)

## How Koinos Block Production Works

In Koinos Proof of Burn (PoB), block producers burn KOIN to receive VHP (Virtual Hash Power). The more VHP an address holds, the more frequently it wins the right to produce blocks. Each producer needs:

1. A **producer address** with VHP
2. A **private key** used to sign blocks
3. A **producer key registered on-chain** that links the signing key to the producer address

## Starting Point

This guide assumes you have a working Koinos node with the standard file structure:

```
~/koinos/                    # Project directory
├── docker-compose.yml
├── .env
└── config/
    ├── config.yml
    ├── genesis_data.json
    ├── koinos_descriptors.pb
    └── rabbitmq.conf

~/.koinos/                   # Data directory (BASEDIR)
├── block_producer/
│   ├── private.key          # First producer's private key
│   └── public.key           # Auto-generated
├── chain/
├── block_store/
├── mempool/
├── p2p/
└── ...
```

With a `config.yml` that includes a single block producer:

```yaml
global:
  amqp: amqp://guest:guest@amqp:5672/
  log-level: info
  log-color: true
  log-datetime: true
  log-dir: logs
  instance-id: Koinos
  fork-algorithm: pob
  blacklist:
    - block_store.add_block
    - chain.propose_block

block_producer:
  algorithm: pob
  producer: 1AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA    # Your first producer address
```

And an `.env` like:

```bash
BASEDIR=~/.koinos
BLOCK_PRODUCER_TAG=v1.3.0
# ... other tags ...
# COMPOSE_PROFILES=block_producer
```

## Step 1: Register the Producer Key On-Chain

Before running a second producer, the signing key must be registered on-chain for the second address. If this is a new address that has never produced blocks, you need to:

1. Generate or obtain a private key for block signing
2. Register the corresponding public key on-chain

Using kcli:

```bash
# Check if a producer key is already registered
kcli get-producer-key <SECOND_PRODUCER_ADDRESS>

# If not registered, import the wallet that owns the second address
kcli import-wallet <OWNER_PRIVATE_KEY_WIF>

# Register the block signing public key
kcli register-producer-key <SECOND_PRODUCER_ADDRESS> <SIGNING_PUBLIC_KEY_BASE64URL>
```

> **Note:** The owner key (the one that controls the address and its funds) and the signing key (used to sign blocks) can be different. The `register-producer-key` command links a signing public key to a producer address. The signing private key is what goes into the `private.key` file.

> **Fogata Pools:** If your second address is a Fogata mining pool, the producer key is managed by the pool contract. You need to use the signing key that the pool has already registered — you cannot change it via kcli. Check the registered key with `kcli get-producer-key <POOL_ADDRESS>` and use the corresponding private key.

## Step 2: Create the Second Producer's Data Directory

Each producer needs its own data directory to store its private key and logs:

```bash
mkdir -p ~/.koinos_2/block_producer
```

Place the signing private key (WIF format) that corresponds to the public key registered on-chain:

```bash
echo "<SIGNING_PRIVATE_KEY_WIF>" > ~/.koinos_2/block_producer/private.key
chmod 600 ~/.koinos_2/block_producer/private.key
```

### Verify the Key Matches

This is the most common source of errors. The private key in `~/.koinos_2/block_producer/private.key` **must** correspond to the public key registered on-chain for that producer address.

```bash
# Check what's registered on-chain
kcli get-producer-key <SECOND_PRODUCER_ADDRESS>

# After the producer starts, compare with the public.key file it generates
cat ~/.koinos_2/block_producer/public.key
```

If these don't match, the producer will fail with `proof failed vrf validation` errors on every block attempt.

## Step 3: Add the Second Producer to `docker-compose.yml`

The default `docker-compose.yml` from the Koinos repository only includes one `block_producer` service. Add a `block_producer_2` service after it:

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

Replace `<SECOND_PRODUCER_ADDRESS>` with the actual Koinos address (e.g., `1Bbb...`).

### Key Differences from the First Producer

| | `block_producer` | `block_producer_2` |
|---|---|---|
| **Data volume** | `${BASEDIR}` (~/.koinos) | `${BASEDIR}_2` (~/.koinos_2) |
| **Producer address** | From `config.yml` | From `--producer` flag |
| **Private key** | `~/.koinos/block_producer/private.key` | `~/.koinos_2/block_producer/private.key` |

### Why the `--producer` Flag Is Required

Both Docker services run the same `koinos-block-producer` binary. Internally, the binary always reads the `block_producer:` section from `config.yml` — it does not look for a `block_producer_2:` section, regardless of the Docker service name. Without the `--producer` flag, both instances would try to produce for the same address, causing VRF validation failures.

The `--producer` CLI flag overrides the address from the config file, letting you run two instances with different producer addresses from the same shared `config.yml`.

## Step 4: Start Both Producers

Both producers share the `block_producer` profile, so they start and stop together:

```bash
docker compose --profile block_producer up -d
```

Or if you use `COMPOSE_PROFILES` in your `.env`:

```bash
# .env
COMPOSE_PROFILES=block_producer
```

```bash
docker compose up -d
```

## Step 5: Verify

Check the startup logs for both producers:

```bash
docker compose logs --tail 20 block_producer
docker compose logs --tail 20 block_producer_2
```

A healthy startup looks like:

```
Koinos Block Producer v1.3.0
Public address: 1Xxx...          ← derived from private.key
Public key: Axxx...              ← must match on-chain registration
Producer address: 1Bbb...       ← the address with VHP
Connecting AMQP client...
Estimated total VHP producing: 5500000.00000000 VHP
Producing with 800000.00000000 VHP
Produced block - Height: 34828642, ID: 0x1220...
```

Verify that:

1. **Producer address** shows the correct address for each producer
2. **Public key** matches what's registered on-chain
3. **"Producing with X VHP"** reflects the VHP balance of that address, not the other producer's
4. **"Produced block"** messages appear (may take a few minutes depending on VHP)

You can also check VHP balances:

```bash
kcli balance <FIRST_PRODUCER_ADDRESS>
kcli balance <SECOND_PRODUCER_ADDRESS>
```

## Architecture

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

Both producers connect to the same AMQP message bus and share chain state, mempool, and p2p networking. They independently compete for block production using their respective VHP. There is no conflict — each proposes blocks signed with its own key for its own address.

## Troubleshooting

### `proof failed vrf validation`

The private key on disk does not match the public key registered on-chain for that producer address.

**Diagnose:**

```bash
# What's registered on-chain?
kcli get-producer-key <PRODUCER_ADDRESS>

# What key is the producer using? (check logs)
docker compose logs block_producer_2 | grep "Public key"

# Or check the file directly
cat ~/.koinos_2/block_producer/public.key
```

**Fix:** Either replace `private.key` with the correct one, or register the new public key on-chain with `kcli register-producer-key`.

### Both producers show the same address

The `--producer` flag is missing from the `command` in `docker-compose.yml`. Without it, both producers read `block_producer.producer` from the shared `config.yml`.

**Fix:** Add `--producer=<ADDRESS>` to the `block_producer_2` service command.

### "Producing with X VHP" shows the wrong VHP amount

Same root cause as above — the producer is reading the wrong address from config. Verify the `--producer` flag is set correctly and check the "Producer address" line in the logs.

### Containers can't connect to AMQP (`i/o timeout`)

On systems running Docker with the nf_tables iptables backend (common on newer Linux kernels), inter-container networking can break due to bridge netfilter incorrectly filtering intra-bridge traffic.

**Fix:**

```bash
sysctl -w net.bridge.bridge-nf-call-iptables=0
sysctl -w net.bridge.bridge-nf-call-ip6tables=0
```

To make it persistent across reboots:

```bash
cat > /etc/sysctl.d/99-docker-bridge.conf << 'EOF'
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
EOF
```

Then restart Docker and bring the stack back up:

```bash
docker compose --profile block_producer down
systemctl restart docker
docker compose --profile block_producer up -d
```

## Summary

| What | First Producer | Second Producer |
|---|---|---|
| Docker service | `block_producer` | `block_producer_2` |
| Data directory | `~/.koinos` | `~/.koinos_2` |
| Private key | `~/.koinos/block_producer/private.key` | `~/.koinos_2/block_producer/private.key` |
| Address source | `config.yml` → `block_producer.producer` | `--producer` CLI flag |
| Profile | `block_producer` | `block_producer` (shared) |

The process boils down to: create a data directory, place the correct private key, add the service to `docker-compose.yml` with `--producer`, and start.
