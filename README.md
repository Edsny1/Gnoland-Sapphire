# Gnoland sapphire Validator Node Setup

Full guide to run a validator node on **gno.land sapphire** testnet.

> **Quick start:** Use the automated setup script — `bash scripts/setup.sh`

---

## Auto Install

```bash
wget -O setup.sh https://raw.githubusercontent.com/Edsny1/Gnoland-Sapphire/Edsny/scripts/setup.sh
chmod +x setup.sh
bash setup.sh
```

Or with curl:

```bash
curl -o setup.sh https://raw.githubusercontent.com/Edsny1/Gnoland-Sapphire/Edsny/scripts/setup.sh
chmod +x setup.sh
bash setup.sh
```

> This assumes `scripts/setup.sh` sits on the `Edsny` branch of
> `Edsny1/Gnoland-Sapphire`, mirroring the topaz repo layout. If you push to
> `main` or a different path instead, update the URL accordingly.

---

## Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| OS | Ubuntu 22.04+ | Ubuntu 24.04 |
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 200 GB SSD | 500 GB NVMe |
| Go | 1.22+ | latest |

---

## Manual Setup

### 1. Install dependencies

```bash
sudo apt update && sudo apt install -y git make wget curl zstd pv
```

Install Go (skip if already installed with version >= 1.22):

```bash
GO_VERSION="1.22.4"
wget -q "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"
rm "go${GO_VERSION}.linux-amd64.tar.gz"

echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### 2. Build binaries

```bash
git clone https://github.com/gnolang/gno.git
cd gno && git checkout chain/sapphire
make -C gno.land install.gnoland install.gnokey
```

Or build a Docker image:

```bash
docker build --target gnoland -t gnoland:sapphire .
```

Or pull prebuilt image:

```bash
docker pull ghcr.io/gnolang/gno/gnoland:chain-sapphire
```

Verify:

```bash
gnoland version
gnokey --help
```

---

### 3. Download and verify genesis

```bash
cd ~/gno

wget -O genesis.json \
  https://github.com/gnolang/gno/releases/download/chain/sapphire/genesis.json

# Verify SHA256 — must match exactly
shasum -a 256 genesis.json
# expected: d511e0e5b767d4e53f5c1afeeea1bc61d2c7b2118146c820f1f3e4296f67498e
```

---

### 4. Initialize config and keys

```bash
cd ~/gno
gnoland config init
gnoland secrets init
```

---

### 5. Configure node

Apply required chain-wide settings:

```bash
# Persistent peers (required — note: sapphire uses p2p.persistent_peers,
# NOT p2p.seeds like topaz did)
gnoland config set p2p.persistent_peers \
  "g10xll77gz6yzg43v9mdalj8360ng6sunt2vvvhf@seed-1.sapphire.testnets.gno.land:26656,g1gw2d7qsmrg06p204ty2qs8ygzd32t2c7p46te0@seed-2.sapphire.testnets.gno.land:26656"

# Chain-wide consensus settings (must match exactly)
gnoland config set application.prune_strategy syncable
gnoland config set consensus.timeout_commit 3s
gnoland config set consensus.peer_gossip_sleep_duration 10ms
gnoland config set p2p.flush_throttle_timeout 10ms

# Performance
gnoland config set mempool.size 10000
gnoland config set p2p.max_num_outbound_peers 40
```

Set your node-specific values:

```bash
gnoland config set moniker "YOUR-NODE-NAME"
gnoland config set p2p.external_address "YOUR-SERVER-IP:26656"
gnoland config set p2p.pex true
```

#### Using custom ports (17xxx)

If port 26xxx is already in use on your server:

```bash
gnoland config set p2p.laddr "tcp://0.0.0.0:17656"
gnoland config set rpc.laddr "tcp://127.0.0.1:17657"
gnoland config set telemetry.prometheus_listen_addr ":17660"
```

> Update `p2p.external_address` to match your P2P port.

---

### 6. Load snapshot (fast sync)

Skip hours of syncing with a community snapshot for sapphire. Note this is a
**different compression format than the old topaz snapshot** — this one is
`.tar.lz4`, not `.tar.zst`, so the extraction command uses `lz4` instead of
`zstd`.

```bash
# Stop node if running
sudo systemctl stop gnoland 2>/dev/null || pkill -f "gnoland start" 2>/dev/null || true

# Clear old data (keys and config are NOT touched)
rm -rf ~/gno/gnoland-data/db ~/gno/gnoland-data/wal
mkdir -p ~/gno/gnoland-data

# Download and extract snapshot
curl -L https://server-9.hazennetworksolutions.com/gnoland-db-snapshot.tar.lz4 \
  | lz4 -d \
  | tar -xf - -C ~/gno/gnoland-data
```

> Requires the `liblz4-tool` package for the `lz4` CLI:
> `sudo apt install -y liblz4-tool`

The stable URL above always points at the latest snapshot, so its checksum
changes over time — it's not pinned in this guide. This is a third-party
snapshot host (not run by Anthropic or gno.land); use your own judgment about
trusting it, same as any community-provided chain data dump. Sample metadata
for a given snapshot generation looks like this:

```json
{
  "chainId": "sapphire-1",
  "file": "gnoland_sapphire_2026-08-11_80804.tar.lz4",
  "url": "https://server-9.hazennetworksolutions.com/gnoland-sapphire/gnoland_sapphire_2026-08-11_80804.tar.lz4",
  "stableUrl": "https://server-9.hazennetworksolutions.com/gnoland-db-snapshot.tar.lz4",
  "blockHeight": 80804,
  "sizeBytes": 146174656,
  "sha256": "12404e412451ae83dd09aad576be87018800eefc596425c44006811f6701d51c",
  "generatedAt": "2026-08-11T09:17:04Z",
  "compression": "lz4",
  "contents": ["db", "wal"]
}
```

If you want to pin a specific snapshot generation instead of always tracking
the latest, download the dated `url` (e.g.
`gnoland_sapphire_2026-08-11_80804.tar.lz4`) and verify against that
generation's own `sha256` before extracting:

```bash
curl -L -o snapshot.tar.lz4 \
  https://server-9.hazennetworksolutions.com/gnoland-sapphire/gnoland_sapphire_2026-08-11_80804.tar.lz4
shasum -a 256 snapshot.tar.lz4
# compare against that generation's sha256 in its metadata before extracting
```

---

### 7. Create systemd service

```bash
sudo tee /etc/systemd/system/gnoland.service > /dev/null <<EOF
[Unit]
Description=Gnoland sapphire Node
After=network-online.target
Wants=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME/gno
Environment=GNOROOT=$HOME/gno
Environment=HOME=$HOME
ExecStart=$(which gnoland) start \
  --chainid sapphire-1 \
  --genesis $HOME/gno/genesis.json \
  --skip-genesis-sig-verification
Restart=on-failure
RestartSec=5s
LimitNOFILE=65535
StandardOutput=journal
StandardError=journal
SyslogIdentifier=gnoland

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable gnoland
sudo systemctl start gnoland
```

> `--skip-genesis-sig-verification` is required: some genesis transactions
> carry placeholder/intentionally-invalidated signatures (e.g. the
> `names.Enable` call runs with a patched caller), so the node panics on
> startup without it.

Check logs:

```bash
sudo journalctl -u gnoland -f
```

---

### 8. Check sync status

```bash
# Replace 17657 with 26657 if you're using default ports
curl -s http://127.0.0.1:17657/status | python3 -c "
import sys, json
d = json.load(sys.stdin)
info = d['result']['sync_info']
print('Latest block :', info['latest_block_height'])
print('Catching up  :', info['catching_up'])
"
```

Wait until `catching_up: False` before proceeding.

---

### 9. Add operator wallet

Create a new wallet:

```bash
gnokey add YOUR-KEY-NAME
```

Or recover from existing mnemonic:

```bash
gnokey add YOUR-KEY-NAME --recover
```

Get your operator address:

```bash
gnokey list
```

---

### 10. Get testnet GNOT

Visit the faucet and request tokens for your `g1...` operator address:

**https://sapphire.testnets.gno.land/faucet**

Verify balance:

```bash
gnokey query \
  -remote "https://rpc.sapphire.testnets.gno.land" \
  auth/accounts/YOUR-G1-ADDRESS
```

---

### 11. Register as validator candidate

Get your consensus public key:

```bash
gnoland secrets get validator_key
# Note the gpub1... value
```

Register on the valoper realm (must be signed by the operator key — the
realm rejects the call if the signer doesn't control the operator address):

```bash
gnokey maketx call \
  --pkgpath gno.land/r/gnops/valopers \
  --func Register \
  --args "YOUR-MONIKER" \
  --args "YOUR-DESCRIPTION" \
  --args "data-center" \
  --args "YOUR-G1-OPERATOR-ADDRESS" \
  --args "YOUR-GPUB1-CONSENSUS-PUBKEY" \
  --gas-fee 1000000ugnot \
  --gas-wanted 50000000 \
  --chainid sapphire-1 \
  --remote https://rpc.sapphire.testnets.gno.land \
  --broadcast \
  YOUR-KEY-NAME
```

> **Note:** Registration only makes you a **candidate**. A GovDAO member must
> create and pass a proposal to add you to the active validator set
> (via `r/sys/validators/v3`).

---

### 12. Update description (optional)

Description limit is **2048 characters**. To update after registration:

```bash
gnokey maketx call \
  --pkgpath "gno.land/r/gnops/valopers" \
  --func "UpdateDescription" \
  --args "YOUR-G1-OPERATOR-ADDRESS" \
  --args "YOUR-NEW-DESCRIPTION" \
  --gas-fee 1000000ugnot \
  --gas-wanted 50000000 \
  --chainid sapphire-1 \
  --remote https://rpc.sapphire.testnets.gno.land \
  --broadcast \
  YOUR-KEY-NAME
```

---

## Useful commands

```bash
# Service management
sudo systemctl start gnoland
sudo systemctl stop gnoland
sudo systemctl restart gnoland
sudo systemctl status gnoland

# Logs
sudo journalctl -u gnoland -f
sudo journalctl -u gnoland --since "1 hour ago"

# Sync status
curl -s http://127.0.0.1:17657/status | python3 -c \
  "import sys,json; d=json.load(sys.stdin)['result']['sync_info']; print('Height:', d['latest_block_height'], '| Catching up:', d['catching_up'])"

# Validator key info
gnoland secrets get validator_key

# Wallet list
gnokey list
```


## Explorer & resources

| Resource | URL |
|----------|-----|
| Explorer | https://gnoscan.io |
| Faucet | https://sapphire.testnets.gno.land/faucet |
| Valopers | https://sapphire.testnets.gno.land/r/gnops/valopers |
| Active validators | https://sapphire.testnets.gno.land/r/sys/validators/v3 |
| RPC | https://rpc.sapphire.testnets.gno.land |
| Snapshot | https://server-9.hazennetworksolutions.com/gnoland-db-snapshot.tar.lz4 |

---

## Firewall

```bash
# P2P — must be open to public
sudo ufw allow 17656/tcp comment "gnoland P2P"

# RPC — open if you serve public endpoints
sudo ufw allow 17657/tcp comment "gnoland RPC"

# Prometheus — restrict to monitoring server only
sudo ufw allow from YOUR-MONITORING-IP to any port 17660
```
## Delete Node
```bash
sudo systemctl stop gnoland
sudo systemctl disable gnoland
sudo rm -f /etc/systemd/system/gnoland.service
rm -rf ~/gno
rm -f $(command -v gnoland)
rm -f $(command -v gnokey)
rm -f ~/go/bin/gnoland ~/go/bin/gnokey
```
