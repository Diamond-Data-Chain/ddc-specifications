# Node Operations

**Version:** 1.0.0
**Status:** Stable
**Last Updated:** 2026-08-05

---

## Abstract

This document defines the operational standards for running DDC nodes in production. It covers hardware requirements, software installation, configuration, key management, monitoring, maintenance, and upgrade procedures. Node operators MUST comply with all normative requirements in this document to maintain validator eligibility and network standing.

---

## Node Roles

| Role | Description | Requirements |
|------|-------------|--------------|
| **Full Node** | Stores the complete chain; serves API requests | See §2 |
| **Validator Node** | Full node that participates in consensus | Full Node requirements + bonded stake |
| **Archive Node** | Full node that retains all historical state (no pruning) | Full Node requirements + additional storage |
| **Light Node** | Stores only block headers; verifies data via proofs | Minimal |

---

## Hardware Requirements

### Full Node (Minimum)

| Component | Requirement |
|-----------|-------------|
| CPU | 4 cores, x86-64, 2.5 GHz |
| RAM | 8 GiB |
| Storage | 500 GiB NVMe SSD |
| Network | 100 Mbps symmetric, static IP |
| OS | Ubuntu 22.04 LTS or Debian 12 |

### Full Node (Recommended)

| Component | Requirement |
|-----------|-------------|
| CPU | 8 cores, x86-64, 3.0 GHz |
| RAM | 16 GiB |
| Storage | 1 TiB NVMe SSD |
| Network | 1 Gbps symmetric, static IP |
| OS | Ubuntu 22.04 LTS |

### Validator Node (Required)

Validator nodes MUST meet the **Recommended** full node specification. Additionally:

| Component | Requirement |
|-----------|-------------|
| CPU | 8+ cores, x86-64, 3.0+ GHz |
| RAM | 32 GiB |
| Storage | 2 TiB NVMe SSD (RAID-1 recommended) |
| Network | 1 Gbps symmetric, static IP, redundant uplink |
| UPS | Uninterruptible power supply |

### Archive Node

| Component | Requirement |
|-----------|-------------|
| CPU | 8 cores, x86-64 |
| RAM | 32 GiB |
| Storage | 4 TiB NVMe SSD (or NVMe RAID array) |
| Network | 1 Gbps |

---

## Software Installation

### Binary Installation

Official DDC node binaries are published on the [DDC GitHub Releases](https://github.com/Diamond-Data-Chain) page. Operators MUST verify the binary signature before installation:

```bash
# Download binary and signature
curl -LO https://releases.ddc.network/v1.0.0/ddc-node-linux-amd64
curl -LO https://releases.ddc.network/v1.0.0/ddc-node-linux-amd64.sig

# Verify signature (using DDC release signing key)
gpg --verify ddc-node-linux-amd64.sig ddc-node-linux-amd64

# Install
chmod +x ddc-node-linux-amd64
sudo mv ddc-node-linux-amd64 /usr/local/bin/ddc-node
```

### Docker

```bash
docker pull ddcnetwork/ddc-node:1.0.0
```

Verify the image digest against the value published on the official release page before running.

### From Source

```bash
git clone https://github.com/Diamond-Data-Chain/ddc-node
cd ddc-node
git checkout v1.0.0
cargo build --release
```

Requires Rust 1.78 or later.

---

## Configuration

The node is configured via a TOML file (default: `/etc/ddc/config.toml`) or environment variables (prefixed `DDC_`).

### Minimal Configuration (Full Node)

```toml
[node]
name = "my-ddc-node"
role = "full"

[network]
listen_addresses = ["/ip4/0.0.0.0/tcp/30333"]
bootnodes = [
  "/dns4/boot1.ddc.network/tcp/30333/p2p/<peer-id>",
  "/dns4/boot2.ddc.network/tcp/30333/p2p/<peer-id>",
]

[rpc]
http_port = 9933
ws_port   = 9944
cors      = ["https://myapp.example.com"]

[database]
path = "/var/lib/ddc/db"
pruning = "archive"    # "archive" | "1000" (number of blocks to retain)
```

### Validator Configuration

```toml
[node]
name    = "my-validator"
role    = "validator"

[validator]
# Path to the session key file (see §5 Key Management)
session_key_path = "/etc/ddc/session_key.json"
```

### Environment Variable Overrides

All config keys can be overridden with environment variables using `DDC_` prefix and `__` as separator:

```bash
DDC_NODE__NAME="my-node"
DDC_NETWORK__LISTEN_ADDRESSES="/ip4/0.0.0.0/tcp/30333"
```

---

## Key Management

### Node Identity Key

Each node has an identity Ed25519 keypair that determines its `peer_id`. This key:

- Is generated automatically on first startup if not present.
- MUST be stored at `{database_path}/identity.key` (or a path specified in config).
- MUST NOT be shared or reused across nodes.
- Loss of this key changes the node's `peer_id`; this is non-critical for full nodes but requires re-bonding for validators.

### Session Keys

Validator nodes use **session keys** for signing consensus messages. Session keys:

- Are hot keys that reside on the validator's host.
- MUST be rotated at least every **90 days**.
- Are registered on-chain via the `session.setKeys` extrinsic before activation.

Generating new session keys:

```bash
ddc-node key generate-session-keys --output /etc/ddc/session_key.json
```

The output contains a JSON object with Ed25519 and BLS12-381 public keys to submit on-chain.

### Stash/Controller Keys

Validator stake management uses a **stash/controller** model:

- **Stash Account** – Holds the bonded funds. SHOULD be kept in cold storage (hardware wallet).
- **Controller Account** – Manages staking operations (bond, unbond, nominate). Kept online with minimal funds.

Operators MUST NOT store stash account private keys on the validator host.

---

## Systemd Service

```ini
[Unit]
Description=DDC Node
After=network-online.target
Wants=network-online.target

[Service]
User=ddc
ExecStart=/usr/local/bin/ddc-node --config /etc/ddc/config.toml
Restart=always
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable ddc-node
sudo systemctl start ddc-node
```

---

## Monitoring

### Required Metrics

Nodes MUST expose Prometheus metrics on port **9615** (configurable). The following metrics MUST be present:

| Metric | Type | Description |
|--------|------|-------------|
| `ddc_block_height` | Gauge | Current finalized block height |
| `ddc_peer_count` | Gauge | Number of connected peers |
| `ddc_is_syncing` | Gauge | 1 if syncing, 0 if synced |
| `ddc_tx_pool_size` | Gauge | Pending transactions in mempool |
| `ddc_finality_lag` | Gauge | Seconds since last finalized block |
| `ddc_validator_active` | Gauge | 1 if node is an active validator, 0 otherwise |
| `ddc_consensus_rounds_total` | Counter | Total consensus rounds participated in |
| `ddc_slashing_events_total` | Counter | Slashing events detected |

### Alerting Recommendations

| Condition | Severity | Action |
|-----------|----------|--------|
| `ddc_peer_count < 3` | Warning | Investigate connectivity |
| `ddc_peer_count == 0` | Critical | Immediate investigation |
| `ddc_finality_lag > 30s` | Warning | Check consensus health |
| `ddc_finality_lag > 120s` | Critical | Escalate immediately |
| `ddc_is_syncing == 1` for > 1 hour | Warning | Check sync progress |
| `ddc_slashing_events_total` increases | Critical | Investigate equivocation |

---

## Upgrade Procedure

### Scheduled Upgrade

1. Monitor governance proposals for scheduled upgrade blocks.
2. Download and verify the new node binary before the activation block.
3. Schedule a maintenance window to restart the node.
4. Stop the node service, replace the binary, and restart:
   ```bash
   sudo systemctl stop ddc-node
   sudo mv /usr/local/bin/ddc-node /usr/local/bin/ddc-node.bak
   sudo mv ddc-node-linux-amd64-NEW /usr/local/bin/ddc-node
   sudo systemctl start ddc-node
   ```
5. Confirm the node has restarted and is syncing/participating in consensus.

Validators MUST complete the upgrade before the governance-specified activation block. Failure to upgrade may result in being excluded from the validator set.

### Emergency Upgrade

In the event of a critical bug, the DDC Foundation may issue an emergency release. Operators MUST apply emergency releases within **4 hours** of announcement.

---

## Data Backup

Full node operators SHOULD periodically back up the chain database. Validators MUST maintain backups of session keys (securely, offline).

```bash
# Stop node before backup to ensure consistency
sudo systemctl stop ddc-node
sudo rsync -az /var/lib/ddc/db/ /backup/ddc-db/
sudo systemctl start ddc-node
```

Operators MAY use LVM snapshots or filesystem snapshots for zero-downtime backups.

---

## Security Hardening

Operators MUST apply the following security measures:

1. **Firewall:** Allow inbound only on:
   - TCP 30333 (P2P) — public
   - TCP 9933, 9944 (RPC) — restrict to trusted IPs or loopback only
   - TCP 9615 (Prometheus) — restrict to monitoring host

2. **OS Hardening:**
   - Run the node as a non-root system user (`ddc`).
   - Apply automatic security updates (`unattended-upgrades`).
   - Disable unused services.

3. **SSH:**
   - Disable password authentication; use SSH keys only.
   - Consider restricting SSH to a management VPN.

4. **Secrets:**
   - Do not store stash account keys on the node host.
   - Session key files MUST have mode `0600` and be owned by the `ddc` user.

---

## References

- [Architecture Overview](architecture.md)
- [Consensus Protocol](consensus.md)
- [Governance](governance.md)
- [Prometheus Monitoring](https://prometheus.io/)
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
