# Republic AI Validator Node Installation Guide

This repository contains the comprehensive guide for setting up and maintaining a **Republic AI Validator Node**, including the latest **v0.2.1** binary update.

---

## 💻 Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **CPU** | 4+ Cores | 8+ Cores     |
| **RAM** | 16 GB   | 32 GB       |
| **Disk** | 500GB NVMe SSD | 1TB NVMe SSD |
| **OS** | Ubuntu 24.04 | Ubuntu 24.04 |

---

## 🛠 1. System Preparation
Update your server and install the necessary development tools.

```bash
sudo apt update -y && sudo apt upgrade -y
sudo apt install htop ca-certificates zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev tmux iptables curl nvme-cli git wget make jq libleveldb-dev build-essential pkg-config ncdu tar clang bsdmainutils lsb-release libssl-dev libreadline-dev libffi-dev jq gcc screen file nano btop unzip lz4 -y
```

### 🚀 2. Install Go & Cosmovisor
Republic AI runs on the Cosmos SDK; therefore, Go and Cosmovisor are required for smooth upgrades.
```bash

cd $HOME
wget [https://go.dev/dl/go1.22.3.linux-amd64.tar.gz](https://go.dev/dl/go1.22.3.linux-amd64.tar.gz)
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.22.3.linux-amd64.tar.gz
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile

go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest

```


⚙️ 3. Environment Variables
Set your wallet name and custom port (Example: 43).
```bash

echo "export REPUBLIC_WALLET='wallet'" >> $HOME/.bash_profile
echo "export REPUBLIC_PORT='43'" >> $HOME/.bash_profile
source $HOME/.bash_profile

```

📦 4. Binary & Node Initialization
Download the initial binary and initialize the node.
```bash

VERSION="v0.1.0"
mkdir -p $HOME/.republicd/cosmovisor/genesis/bin

curl -L "[https://media.githubusercontent.com/media/RepublicAI/networks/main/testnet/releases/$](https://media.githubusercontent.com/media/RepublicAI/networks/main/testnet/releases/$){VERSION}/republicd-linux-amd64" -o republicd
chmod +x republicd

mv republicd $HOME/.republicd/cosmovisor/genesis/bin/
ln -s $HOME/.republicd/cosmovisor/genesis $HOME/.republicd/cosmovisor/current
sudo ln -s $HOME/.republicd/cosmovisor/genesis/bin/republicd /usr/local/bin/republicd

```
# Initialize (Replace YOUR_NODE_NAME)
```bash
republicd init YOUR_NODE_NAME --chain-id raitestnet_77701-1 --home $HOME/.republicd

```

📄 5. Genesis & Configuration
Download the genesis file and configure port settings.
```bash


curl -s [https://raw.githubusercontent.com/RepublicAI/networks/main/testnet/genesis.json](https://raw.githubusercontent.com/RepublicAI/networks/main/testnet/genesis.json) > $HOME/.republicd/config/genesis.json

# Port Mapping
sed -i.bak -e "s%:26658%:${REPUBLIC_PORT}658%g; s%:26657%:${REPUBLIC_PORT}657%g; s%:6060%:${REPUBLIC_PORT}060%g; s%:26656%:${REPUBLIC_PORT}656%g; s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${REPUBLIC_PORT}656\"%; s%:26660%:${REPUBLIC_PORT}660%g" $HOME/.republicd/config/config.toml

# App Mapping
sed -i.bak -e "s%:1317%:${REPUBLIC_PORT}317%g; s%:8080%:${REPUBLIC_PORT}080%g; s%:9090%:${REPUBLIC_PORT}090%g; s%:9091%:${REPUBLIC_PORT}091%g; s%:8545%:${REPUBLIC_PORT}545%g; s%:8546%:${REPUBLIC_PORT}546%g; s%:6065%:${REPUBLIC_PORT}065%g" $HOME/.republicd/config/app.toml

# Pruning & Indexer (Disk Optimization)
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.republicd/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.republicd/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"10\"/" $HOME/.republicd/config/app.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.republicd/config/config.toml
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "250000000arai"|g' $HOME/.republicd/config/app.toml

```

🛡️ 6. Systemd Service
Create the service file for Cosmovisor.
```bash

sudo tee /etc/systemd/system/republicd.service > /dev/null <<EOF
[Unit]
Description=Republic AI Node
After=network-online.target

[Service]
User=$USER
Environment="DAEMON_NAME=republicd"
Environment="DAEMON_HOME=$HOME/.republicd"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
ExecStart=$HOME/go/bin/cosmovisor run start --home $HOME/.republicd --chain-id raitestnet_77701-1
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable republicd
sudo systemctl start republicd
```


#⬆️ 7. Upgrade to v0.2.1 (Latest Update)
If you have an existing node, follow these steps to upgrade to the latest version.
```bash

# 1. Stop Node
sudo systemctl stop republicd

# 2. Download New Binary
cd $HOME/.republicd/cosmovisor/genesis/bin
wget -O republicd_v021 [https://github.com/RepublicAI/networks/releases/download/v0.2.1/republicd-linux-amd64](https://github.com/RepublicAI/networks/releases/download/v0.2.1/republicd-linux-amd64)

# 3. Verify Checksum
echo "d10991b623fa62ee0f3c42ad15abbaa246f89d73a82736fad090e607b7cc4b8f  republicd_v021" | sha256sum -c

# 4. Swap Binaries
chmod +x republicd_v021
sudo mv republicd republicd_v0.1.0_backup
sudo mv republicd_v021 republicd

# 5. Update Global Link
cd /usr/local/bin
sudo mv republicd republicd_backup_pre_v0.2.1
sudo cp $HOME/.republicd/cosmovisor/genesis/bin/republicd /usr/local/bin/republicd
chmod +x /usr/local/bin/republicd

# 6. Restart
sudo systemctl daemon-reload
sudo systemctl start republicd
journalctl -u republicd -f

```


💰 8. Validator Setup
Once your node is fully synced:
```bash

# Create Wallet
republicd keys add $REPUBLIC_WALLET

# Create Validator (Ensure you have testnet tokens)
PUBKEY=$(jq -r '.pub_key.value' $HOME/.republicd/config/priv_validator_key.json)

cat > validator.json << EOF
{
  "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"$PUBKEY"},
  "amount": "20000000000000000000arai",
  "moniker": "YOUR_MONIKER",
  "identity": "",
  "website": "",
  "security": "contact@email.com",
  "details": "Validator Description",
  "commission-rate": "0.05",
  "commission-max-rate": "0.15",
  "commission-max-change-rate": "0.02",
  "min-self-delegation": "1"
}
EOF
```

```bash
republicd tx staking create-validator validator.json \
--from $REPUBLIC_WALLET \
--chain-id raitestnet_77701-1 \
--gas auto \
--gas-adjustment 1.5 \
--gas-prices "1000000000arai" \
--node tcp://localhost:${REPUBLIC_PORT}657 \
-y
```

🗳️ 9. Staking & Delegation
To increase your validator's voting power and enter the active set, you must delegate tokens to your valoper address.

Token Unit Conversion

In the Republic network, transactions use the arai unit:

10 Tokens: 10000000000000000000arai

1 Token: 1000000000000000000arai

Self-Delegation Command

Replace YOUR_VALOPER_ADDRESS with your actual validator address.

```bash
republicd tx staking delegate YOUR_VALOPER_ADDRESS \
10000000000000000000arai \
--from $REPUBLIC_WALLET \
--chain-id raitestnet_77701-1 \
--gas auto \
--gas-adjustment 1.5 \
--gas-prices 1000000000arai \
--node tcp://localhost:${REPUBLIC_PORT}657 \
-y
```

How to Find Your Valoper Address

To see your own validator operator address, run:

```bash
republicd keys show $REPUBLIC_WALLET --bech val -a
Alternatively, search for your moniker on the Explorer: 👉 Republic AI Explorer - Staking
```

🛠️ 10. Common Maintenance Commands
Withdraw Staking Rewards

To claim all available rewards from your validator:

```bash
republicd tx distribution withdraw-all-rewards \
--from $REPUBLIC_WALLET \
--chain-id raitestnet_77701-1 \
--gas auto \
--gas-adjustment 1.5 \
--gas-prices 1000000000arai \
--node tcp://localhost:${REPUBLIC_PORT}657 \
-y
```

Unjail Your Validator

If your node stops signing blocks and gets "jailed," use this command to return to the active set:

```bash
republicd tx slashing unjail \
--from $REPUBLIC_WALLET \
--chain-id raitestnet_77701-1 \
--gas auto \
--gas-adjustment 1.5 \
--gas-prices 1000000000arai \
--node tcp://localhost:${REPUBLIC_PORT}657 \
-y
```

Edit Validator Details

To update your moniker, website, or identity (Keybase):

```bash
republicd tx staking edit-validator \
--new-moniker "NEW_NAME" \
--website "https://your-website.com" \
--identity "YOUR_KEYBASE_ID" \
--details "New description" \
--from $REPUBLIC_WALLET \
--chain-id raitestnet_77701-1 \
--node tcp://localhost:${REPUBLIC_PORT}657 \
-y
```

