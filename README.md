# Running Two Block Producers on a Single Koinos Node

This guide explains how to configure a Koinos node to run two independent block producers sharing the same infrastructure (chain, mempool, p2p, AMQP). Each producer uses its own address, private key, and data directory.

## Prerequisites

- A working Koinos node running via Docker Compose (see the [main Koinos README](https://github.com/koinos/koinos))
- Two Koinos addresses, each with VHP (burned KOIN)
- The private key for each producer address registered on-chain via `register-producer-key`

## Overview

The Koinos Docker Compose stack already includes a `block_producer_2` service definition. By default it shares the `block_producer` profile, so both producers start and stop together. The key challenge is that both services receive the same shared `config.yml`, and the block producer binary always reads the `block_producer` config section — regardless of the Docker service name. This means you need to override the producer address via the command line for the second producer.

Each producer also needs:

- Its own **data directory** with a matching `private.key`
- The corresponding **producer key registered on-chain**

## Step 1: Prepare the Data Directory

The first producer uses `$BASEDIR` (typically `~/.koinos`). The second producer uses `$BASEDIR_2` (typically `~/.koinos_2`).

```bash
mkdir -p ~/.koinos_2/block_producer
```

## Step 2: Place the Private Key

Copy the private key (WIF format) for your second producer address into the new data directory:

```bash
echo "YOUR_PRIVATE_KEY_WIF" > ~/.koinos_2/block_producer/private.key
```

The public key file will be generated automatically on first run.

### Verifying the Key Matches On-Chain Registration

The private key you place here **must** correspond to the public key registered on-chain for that producer address. You can check the registered key with:

```bash
kcli get-producer-key YOUR_PRODUCER_ADDRESS
```

If the keys don't match, the producer will fail with `proof failed vrf validation` errors. To register a new key, use:

```bash
kcli register-producer-key PRODUCER_ADDRESS PUBLIC_KEY_BASE64URL
```

## Step 3: Configure `docker-compose.yml`

The default `docker-compose.yml` already includes a `block_producer_2` service. The critical change is adding `--producer=ADDRESS` to the command, because the binary ignores the `block_producer_2` section in `config.yml` and reads `block_producer` instead:

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
  command: --basedir=/koinos --producer=YOUR_SECOND_PRODUCER_ADDRESS
```

Replace `YOUR_SECOND_PRODUCER_ADDRESS` with the actual Koinos address (e.g., `1KfD7n93LnnihyygopWUVTkbtWVe5aXXGW`).

### Why `--producer` Is Needed

Both `block_producer` and `block_producer_2` Docker services run the same `koinos-block-producer` binary. Internally, the binary identifies itself as `block_producer` and reads only the `block_producer:` section from `config.yml`. Adding a `block_producer_2:` section to `config.yml` has no effect — the binary simply doesn't look for it.

The `--producer` CLI flag overrides the address from the config file, letting you run two instances with different producer addresses from the same shared config.

## Step 4: Optionally Update `config.yml`

While the `block_producer_2` section in `config.yml` is not read by the binary, you may still want to keep it as documentation:

```yaml
block_producer:
  algorithm: pob
  producer: FIRST_PRODUCER_ADDRESS

# Note: block_producer_2 section is for reference only.
# The actual address is set via --producer in docker-compose.yml.
# block_producer_2:
#   algorithm: pob
#   producer: SECOND_PRODUCER_ADDRESS
```

## Step 5: Start Both Producers

Both producers share the `block_producer` profile:

```bash
docker compose --profile block_producer up -d
```

Or start everything:

```bash
docker compose --profile all up -d
```

## Step 6: Verify

Check that both producers are running and producing:

```bash
docker compose logs --tail 20 block_producer
docker compose logs --tail 20 block_producer_2
```

A healthy producer log looks like:

```
Public key: AufFOI9avSLIsr5zwQqVXJbmb_BLpDHYaOz2zw7mQ2G7
Producer address: 1KfD7n93LnnihyygopWUVTkbtWVe5aXXGW
...
Producing with 872178.21543544 VHP
...
Produced block - Height: 34828642, ID: 0x1220785285a0...
```

If you see `proof failed vrf validation` errors, the private key in `~/.koinos_2/block_producer/private.key` does not match the producer key registered on-chain. See Step 2.

You can also check VHP balances with:

```bash
kcli balance FIRST_PRODUCER_ADDRESS
kcli balance SECOND_PRODUCER_ADDRESS
```

## Architecture Summary

```
┌─────────────────────────────────────────────────┐
│                  Docker Network                  │
│                                                  │
│  ┌──────────┐   ┌───────┐   ┌────────────────┐  │
│  │  RabbitMQ │◄──│ chain │──►│   block_store  │  │
│  │  (AMQP)  │   └───┬───┘   └────────────────┘  │
│  └────┬─────┘       │                            │
│       │         ┌───┴───┐                        │
│       │         │mempool│                        │
│       │         └───┬───┘                        │
│       │             │                            │
│  ┌────┴─────────────┴────────────┐               │
│  │                               │               │
│  ▼                               ▼               │
│  ┌────────────────┐  ┌────────────────────────┐  │
│  │ block_producer │  │   block_producer_2     │  │
│  │ ~/.koinos/     │  │   ~/.koinos_2/         │  │
│  │ Address: AAA   │  │   Address: BBB         │  │
│  │ Key: key_a     │  │   Key: key_b           │  │
│  └────────────────┘  └────────────────────────┘  │
│                                                  │
│  ┌─────┐  ┌────────┐  ┌──────┐  ┌───────────┐   │
│  │ p2p │  │jsonrpc │  │ grpc │  │   rest    │   │
│  └─────┘  └────────┘  └──────┘  └───────────┘   │
└─────────────────────────────────────────────────┘
```

Both producers connect to the same AMQP bus, chain, and mempool. They independently attempt to produce blocks using their respective VHP. There is no conflict — each proposes blocks signed with its own key for its own address.

## Troubleshooting

### `proof failed vrf validation`

The private key on disk does not match the public key registered on-chain. Verify with:

```bash
kcli get-producer-key YOUR_ADDRESS
```

Compare the output with the `public.key` file in the producer's data directory.

### Producer address shows the wrong address

The `--producer` flag is missing from the `command` in `docker-compose.yml`. Without it, both producers read the same `block_producer.producer` value from `config.yml`.

### Containers can't connect to AMQP (`i/o timeout`)

On systems running Docker 29.x with kernel 6.x and the nf_tables iptables backend, inter-container networking can break. The fix is:

```bash
sysctl -w net.bridge.bridge-nf-call-iptables=0
sysctl -w net.bridge.bridge-nf-call-ip6tables=0
```

To make it persistent:

```bash
cat > /etc/sysctl.d/99-docker-bridge.conf << 'EOF'
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
EOF
```

Then restart Docker and bring the stack back up.
