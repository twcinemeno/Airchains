Manual Installation
Official Documentation
```
Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)
```
**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.21.6"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export AIRCHAIN_CHAIN_ID="varanasi-1"" >> $HOME/.bash_profile
echo "export AIRCHAIN_PORT="19"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
wget -O junctiond https://github.com/airchains-network/junction/releases/download/v0.3.1/junctiond-linux-amd64
chmod +x junctiond
mv junctiond $HOME/go/bin/
```

**config and init app**
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.cargo/env
junctiond init $MONIKER --chain-id $CHAIN_ID 
sed -i -e "s|^node *=.*|node = \"tcp://localhost:${AIRCHAIN_PORT}657\"|" $HOME/.junctiond/config/client.toml
```

**download genesis and addrbook**
```
wget -O $HOME/.junctiond/config/genesis.json https://server-3.itrocket.net/testnet/airchains/genesis.json
wget -O $HOME/.junctiond/config/addrbook.json  https://server-3.itrocket.net/testnet/airchains/addrbook.json
```

**set seeds and peers**
```
SEEDS="97cadd453fa35cee05f72611fdb15a49112575cb@airchains-testnet-seed.itrocket.net:19656"
PEERS="79f26210777e84efb600bf776c32615a72675d9f@airchains-testnet-peer.itrocket.net:19656,db686fcfdf0b4676d601d5beb11faee5ad96bff1@37.27.71.199:28656,eef0b9627b4f7e5b3f7ea04a5b2afd16136ee86d@5.9.65.165:22656,8c229309660496e71b8a9d1edee46a18693b8e70@65.109.111.234:19656,0b4e78189c9148dda5b1b98c6e46b764337558a3@91.227.33.18:19656,b57745eecc8c9638a3599c81f82dd69720df0ed8@94.130.164.82:26756,b43f7c96bb780d9ac535d3c1f78092cf8c455e85@104.36.23.246:26656,4aaa6f76a1009feccffa90e8a00dd6343ca9b01f@152.53.49.146:19656,3650f3737940af2d6cc8d17244706505648ff639@212.56.32.148:14156,ca0a4b67fd6ffd6a70ea8d0e3c8d284de0f8222f@37.27.132.57:19656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.junctiond/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${AIRCHAIN_PORT}317%g;
s%:8080%:${AIRCHAIN_PORT}080%g;
s%:9090%:${AIRCHAIN_PORT}090%g;
s%:9091%:${AIRCHAIN_PORT}091%g;
s%:8545%:${AIRCHAIN_PORT}545%g;
s%:8546%:${AIRCHAIN_PORT}546%g;
s%:6065%:${AIRCHAIN_PORT}065%g" $HOME/.junctiond/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${AIRCHAIN_PORT}658%g;
s%:26657%:${AIRCHAIN_PORT}657%g;
s%:6060%:${AIRCHAIN_PORT}060%g;
s%:26656%:${AIRCHAIN_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${AIRCHAIN_PORT}656\"%;
s%:26660%:${AIRCHAIN_PORT}660%g" $HOME/.junctiond/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.junctiond/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.junctiond/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.junctiond/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.001uamf"|g' $HOME/.junctiond/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.junctiond/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.junctiond/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/junctiond.service > /dev/null <<EOF
[Unit]
Description=Airchains node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.junctiond
ExecStart=$(which junctiond) start --home $HOME/.junctiond
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

# reset and download snapshot
junctiond tendermint unsafe-reset-all --home $HOME/.junctiond
if curl -s --head curl https://server-3.itrocket.net/testnet/airchains/airchains_2025-04-30_587601_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-3.itrocket.net/testnet/airchains/airchains_2025-04-30_587601_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.junctiond
    else
  echo "no snapshot found"
fi

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable junctiond
sudo systemctl restart junctiond && sudo journalctl -u junctiond -fo cat
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/airchains/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
junctiond keys add $WALLET

# to restore exexuting wallet, use the following command
junctiond keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(junctiond keys show $WALLET -a)
VALOPER_ADDRESS=$(junctiond keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
junctiond status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
junctiond query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.junctiond/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://airchains-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"
  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, uamf
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
cd $HOME
# Create validator.json file
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(junctiond comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000uamf\",
    \"moniker\": \"test\",
    \"identity\": \"\",
    \"website\": \"\",
    \"security\": \"\",
    \"details\": \"I love blockchain ❤️\",
    \"commission-rate\": \"0.1\",
    \"commission-max-rate\": \"0.2\",
    \"commission-max-change-rate\": \"0.01\",
    \"min-self-delegation\": \"1\"
}" > validator.json
# Create a validator using the JSON configuration
junctiond tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id varanasi-1 \
	--fees 5000uamf 
	
Monitoring
If you want to have set up a monitoring and alert system use our cosmos nodes monitoring guide with tenderduty

Security
To protect you keys please don`t share your privkey, mnemonic and follow basic security rules

Set up ssh keys for authentication
You can use this guide to configure ssh authentication and disable password authentication on your server

Firewall security
Set the default to allow outgoing connections, deny all incoming, allow ssh and node p2p port

sudo ufw default allow outgoing 
sudo ufw default deny incoming 
sudo ufw allow ssh/tcp 
sudo ufw allow ${AIRCHAIN_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop junctiond
sudo systemctl disable junctiond
sudo rm -rf /etc/systemd/system/junctiond.service
sudo rm $(which junctiond)
sudo rm -rf $HOME/.junctiond
sed -i "/AIRCHAIN_/d" $HOME/.bash_profile
