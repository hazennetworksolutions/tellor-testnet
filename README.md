<div align="center">

# ⛏️ Tellor Layer Testnet Full Node & Validator Setup Guide

**A complete guide to running a Tellor Layer testnet full node and registering as a validator**  
*System preparation, binary installation, Cosmovisor setup, and validator creation — step by step.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Tellor](https://img.shields.io/badge/Tellor-Palmito%20Testnet-2D2D2D?style=flat-square)](https://tellor.io)
[![Version](https://img.shields.io/badge/Node%20Version-v6.1.5-brightgreen?style=flat-square)](https://github.com/tellor-io/layer/releases)
[![Chain ID](https://img.shields.io/badge/Chain%20ID-layertest--5-blue?style=flat-square)](https://docs.tellor.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Network:** Tellor Layer Palmito Testnet (Chain ID: layertest-5)  
> **Version:** v6.1.5  
> **Last Updated:** May 2026

---

## Table of Contents

- [Hardware Requirements](#hardware-requirements)
- [Network Endpoints](#network-endpoints)
- [Step 1 — System Verification](#step-1--system-verification)
- [Step 2 — System Update and Dependencies](#step-2--system-update-and-dependencies)
- [Step 3 — Install Go](#step-3--install-go)
- [Step 4 — Download Binary](#step-4--download-binary)
- [Step 5 — Install Cosmovisor](#step-5--install-cosmovisor)
- [Step 6 — Create Systemd Service](#step-6--create-systemd-service)
- [Step 7 — Initialize the Node](#step-7--initialize-the-node)
- [Step 8 — Download Genesis and Addrbook](#step-8--download-genesis-and-addrbook)
- [Step 9 — Configure Ports and Pruning](#step-9--configure-ports-and-pruning)
- [Step 10 — Configure Seeds and Peers](#step-10--configure-seeds-and-peers)
- [Step 11 — Start the Node](#step-11--start-the-node)
- [Step 12 — Create a Wallet](#step-12--create-a-wallet)
- [Step 13 — Register as a Validator](#step-13--register-as-a-validator)
- [Monitoring the Node](#monitoring-the-node)
- [Useful Commands](#useful-commands)

---

## Hardware Requirements

| Component | Minimum | Recommended |
|---|---|---|
| Operating System | Ubuntu 22.04+ | Ubuntu 24.04 |
| CPU | 4 cores | 8+ cores |
| RAM | 16 GB | 32 GB |
| Disk | 500 GB NVMe SSD | 1 TB NVMe SSD |
| Network | 100 Mbps | 500 Mbps |

---

## Network Endpoints

| Type | Endpoint |
|---|---|
| RPC | https://rpc.layertest-5.tellor.io |
| REST API | https://api.layertest-5.tellor.io |
| Explorer | https://layertest-5.tellor.io |
| Docs | https://docs.tellor.io/tellor/running-palmito-testnet |
| GitHub | https://github.com/tellor-io/layer |

---

## Step 1 — System Verification

After SSH-ing into your server, verify the system meets requirements:

```bash
lsb_release -a          # Should be Ubuntu 22.04 or higher
uname -r                # Kernel version
lscpu | grep -E "Model name|CPU\(s\)|Thread|Socket|Core"
free -h                 # Minimum 16 GB RAM
df -h                   # Minimum 500 GB free disk
```

---

## Step 2 — System Update and Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git wget htop tmux build-essential jq make lz4 gcc unzip \
  screen btop iotop nethogs hdparm cmake perl automake autoconf libtool libssl-dev
```

---

## Step 3 — Install Go

```bash
cd $HOME
VER="1.23.0"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"

[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo 'export PATH=/usr/local/go/bin:$HOME/go/bin:$PATH' >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

Verify the installation:

```bash
go version
```

Expected output: `go version go1.23.0 linux/amd64`

---

## Step 4 — Download Binary

```bash
cd $HOME
mkdir -p $HOME/.layer/cosmovisor/upgrades/v6.1.5/bin

wget https://github.com/tellor-io/layer/releases/download/v6.1.5/layer_Linux_x86_64.tar.gz
tar -xvf layer_Linux_x86_64.tar.gz

sudo mv layerd $HOME/.layer/cosmovisor/upgrades/v6.1.5/bin/
chmod +x $HOME/.layer/cosmovisor/upgrades/v6.1.5/bin/layerd

sudo ln -sfn $HOME/.layer/cosmovisor/upgrades/v6.1.5 $HOME/.layer/cosmovisor/current
sudo ln -sfn $HOME/.layer/cosmovisor/current/bin/layerd /usr/local/bin/layerd

rm -f layer_Linux_x86_64.tar.gz
```

Verify:

```bash
layerd version
```

Expected output: `v6.1.5`

---

## Step 5 — Install Cosmovisor

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.6.0
```

Verify:

```bash
cosmovisor version
```

---

## Step 6 — Create Systemd Service

Set your moniker and port prefix:

```bash
MONIKER="YOUR_MONIKER"
PORT="27"   # Default is 26. Change to avoid conflicts if needed (e.g. 27, 28...)
```

Create the service file:

```bash
sudo tee /etc/systemd/system/layerd.service > /dev/null << EOF
[Unit]
Description=Tellor Layer Testnet Node Service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start --home $HOME/.layer
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.layer"
Environment="DAEMON_NAME=layerd"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.layer/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable layerd
```

---

## Step 7 — Initialize the Node

```bash
layerd config node tcp://localhost:${PORT}657
layerd config keyring-backend os
layerd config chain-id layertest-5
layerd init $MONIKER --chain-id layertest-5
```

Set environment variables permanently:

```bash
echo "export MONIKER=$MONIKER" >> $HOME/.bash_profile
echo "export LAYER_CHAIN_ID=\"layertest-5\"" >> $HOME/.bash_profile
echo "export LAYER_PORT=$PORT" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

---

## Step 8 — Download Genesis and Addrbook

```bash
wget -O $HOME/.layer/config/genesis.json \
  https://raw.githubusercontent.com/hazennetworksolutions/tellor-testnet/refs/heads/main/genesis.json

wget -O $HOME/.layer/config/addrbook.json \
  https://raw.githubusercontent.com/hazennetworksolutions/tellor-testnet/refs/heads/main/addrbook.json
```

Verify genesis:

```bash
sha256sum $HOME/.layer/config/genesis.json
```

---

## Step 9 — Configure Ports and Pruning

### Custom Ports

```bash
sed -i.bak -e "s%:1317%:${LAYER_PORT}317%g;
s%:8080%:${LAYER_PORT}080%g;
s%:9090%:${LAYER_PORT}090%g;
s%:9091%:${LAYER_PORT}091%g;
s%:8545%:${LAYER_PORT}545%g;
s%:8546%:${LAYER_PORT}546%g;
s%:6065%:${LAYER_PORT}065%g" $HOME/.layer/config/app.toml

sed -i.bak -e "s%:26658%:${LAYER_PORT}658%g;
s%:26657%:${LAYER_PORT}657%g;
s%:6060%:${LAYER_PORT}060%g;
s%:26656%:${LAYER_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${LAYER_PORT}656\"%;
s%:26660%:${LAYER_PORT}660%g" $HOME/.layer/config/config.toml
```

### Gas Prices

```bash
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0loya"|g' \
  $HOME/.layer/config/app.toml
```

### Pruning

```bash
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.layer/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.layer/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.layer/config/app.toml
```

### Enable Prometheus (optional)

```bash
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.layer/config/config.toml
```

### Disable Indexer (saves disk space)

```bash
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.layer/config/config.toml
```

---

## Step 10 — Configure Seeds and Peers

```bash
SEEDS="c7b175a5bafb35176cdcba3027e764a0dbd0811c@34.219.95.82:26656,05105e8bb28e8c5ace1cecacefb8d4efb0338ec6@18.218.114.74:26656,705f6154c6c6aeb0ba36c8b53639a5daa1b186f6@3.80.39.230:26656,1f6522a346209ee99ecb4d3e897d9d97633ae146@3.101.138.30:26656,3822fa2eb0052b36360a7a6e285c18cc92e26215@175.41.188.192:26656"
PEERS="c7b175a5bafb35176cdcba3027e764a0dbd0811c@34.219.95.82:26656,05105e8bb28e8c5ace1cecacefb8d4efb0338ec6@18.218.114.74:26656,705f6154c6c6aeb0ba36c8b53639a5daa1b186f6@3.80.39.230:26656,1f6522a346209ee99ecb4d3e897d9d97633ae146@3.101.138.30:26656,3822fa2eb0052b36360a7a6e285c18cc92e26215@175.41.188.192:26656"

sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" \
       $HOME/.layer/config/config.toml
```

---

## Step 11 — Start the Node

```bash
sudo systemctl restart layerd
sudo journalctl -u layerd -f --no-pager -o cat
```

Verify the service is running:

```bash
sudo systemctl status layerd --no-pager
```

The service should show `active (running)`.

Check sync status:

```bash
layerd status 2>&1 | jq .SyncInfo
```

Wait until `catching_up` is `false` before proceeding to validator registration.

---

## Step 12 — Create a Wallet

```bash
layerd keys add wallet
```

> ⚠️ **CRITICAL:** Save your mnemonic phrase in a secure location. Without it, you cannot recover your wallet.

To recover an existing wallet:

```bash
layerd keys add wallet --recover
```

Check your balance:

```bash
layerd query bank balances $(layerd keys show wallet -a)
```

---

## Step 13 — Register as a Validator

> The node must be **fully synced** before creating a validator.

### Get your pubkey:

```bash
layerd comet show-validator
```

### Create validator JSON:

```bash
cat > $HOME/validator.json << EOF
{
  "pubkey": $(layerd comet show-validator),
  "amount": "1000000loya",
  "moniker": "YOUR_MONIKER",
  "identity": "",
  "website": "",
  "security": "",
  "details": "",
  "commission-rate": "0.05",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
EOF
```

### Submit the transaction:

```bash
layerd tx staking create-validator $HOME/validator.json \
  --from wallet \
  --chain-id layertest-5 \
  --gas auto \
  --gas-adjustment 1.4 \
  --fees 500loya \
  -y
```

### Verify your validator:

```bash
layerd query staking validator \
  $(layerd keys show wallet --bech val -a)
```

---

## Monitoring the Node

### Watch live block commits:

```bash
sudo journalctl -u layerd -f --no-pager | grep "committed block"
```

### Full logs:

```bash
sudo journalctl -u layerd -f --no-pager
```

### Sync status:

```bash
layerd status 2>&1 | jq .SyncInfo
```

### Service management:

```bash
# Restart service
sudo systemctl restart layerd

# Stop service
sudo systemctl stop layerd

# Check status
sudo systemctl status layerd
```

---

## Useful Commands

### Wallet

```bash
# List wallets
layerd keys list

# Show wallet address
layerd keys show wallet -a

# Check balance
layerd query bank balances $(layerd keys show wallet -a)
```

### Staking

```bash
# Delegate tokens
layerd tx staking delegate \
  $(layerd keys show wallet --bech val -a) 1000000loya \
  --from wallet --chain-id layertest-5 \
  --gas auto --gas-adjustment 1.4 --fees 500loya -y

# Redelegate tokens
layerd tx staking redelegate \
  $(layerd keys show wallet --bech val -a) <NEW_VALOPER> 1000000loya \
  --from wallet --chain-id layertest-5 \
  --gas auto --gas-adjustment 1.4 --fees 500loya -y

# Undelegate tokens
layerd tx staking unbond \
  $(layerd keys show wallet --bech val -a) 1000000loya \
  --from wallet --chain-id layertest-5 \
  --gas auto --gas-adjustment 1.4 --fees 500loya -y
```

### Rewards

```bash
# Withdraw all rewards
layerd tx distribution withdraw-all-rewards \
  --from wallet --chain-id layertest-5 \
  --gas auto --gas-adjustment 1.4 --fees 500loya -y

# Withdraw commission
layerd tx distribution withdraw-rewards \
  $(layerd keys show wallet --bech val -a) --commission \
  --from wallet --chain-id layertest-5 \
  --gas auto --gas-adjustment 1.4 --fees 500loya -y
```

### Governance

```bash
# List proposals
layerd query gov proposals

# Vote on a proposal
layerd tx gov vote 1 yes \
  --from wallet --chain-id layertest-5 \
  --gas auto --gas-adjustment 1.4 --fees 500loya -y
```

### Validator Operations

```bash
# Edit validator
layerd tx staking edit-validator \
  --new-moniker "NEW_MONIKER" \
  --identity "" \
  --from wallet --chain-id layertest-5 \
  --gas auto --gas-adjustment 1.4 --fees 500loya -y

# Unjail validator
layerd tx slashing unjail \
  --from wallet --chain-id layertest-5 \
  --gas auto --gas-adjustment 1.4 --fees 500loya -y

# Check validator signing info
layerd query slashing signing-info $(layerd comet show-validator)
```

---

*Guide maintained by [HazenNetworkSolutions](https://hazennetworksolutions.com)*
