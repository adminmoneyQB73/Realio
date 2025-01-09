Manual Installation
Official Documentation
Recommended Hardware: 4 Cores, 8GB RAM, 240GB of storage (NVME)

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.19.3"
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
echo "export REALIO_CHAIN_ID="realionetwork_3301-1"" >> $HOME/.bash_profile
echo "export REALIO_PORT="23"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm -rf realio-network
git clone https://github.com/realiotech/realio-network.git
cd realio-network
git checkout v1.0.2
make install
```

**config and init app**
```
realio-networkd config node tcp://localhost:${REALIO_PORT}657
realio-networkd config keyring-backend os
realio-networkd config chain-id realionetwork_3301-1
realio-networkd init "test" --chain-id realionetwork_3301-1
```

**download genesis and addrbook**
```
wget -O $HOME/.realio-network/config/genesis.json https://server-2.itrocket.net/mainnet/realio/genesis.json
wget -O $HOME/.realio-network/config/addrbook.json  https://server-2.itrocket.net/mainnet/realio/addrbook.json
```

**set seeds and peers**
```
SEEDS="a5039f260cd848facfbe7fbe62bf3e8adfce9c98@realio-mainnet-seed.itrocket.net:23656"
PEERS="2815cc1437461f808a7f022c0df679fa27918dbc@realio-mainnet-peer.itrocket.net:23656,7eabd881aeda0d4ebba22bd9d5adf79cbe5ab1eb@159.89.224.12:46656,271f194229b4ee9be89777daa3ef8201553865cc@65.108.232.168:36656,41d4a00a29c34a3e5e8084d6edff5be4f53389ab@65.109.18.169:12056,92bd4dd1784f4a7860a2b0d2a5bc8f16d9e5edaa@65.109.103.177:37656,944bc1fd43690447dd500065ad4eb272cfbfefd3@144.76.155.11:23656,56971ee9336dda008d64c3997207eac8e441c3ef@95.216.12.106:21096,32c0e5b5e341e7d68c16377f65c05c1734846a55@65.108.230.113:21096,6f5d99fac932cf4c01a3e408c5f3c1ad57ea507b@138.68.175.105:26656,ae925f32ead79c394bb7f6f8abb5345f7a16b064@167.235.102.45:16656,1f64a6df3783cb10e5bbc67db3bed7aee57f2988@65.21.136.219:21096"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.realio-network/config/config.toml
```
**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${REALIO_PORT}317%g;
s%:8080%:${REALIO_PORT}080%g;
s%:9090%:${REALIO_PORT}090%g;
s%:9091%:${REALIO_PORT}091%g;
s%:8545%:${REALIO_PORT}545%g;
s%:8546%:${REALIO_PORT}546%g;
s%:6065%:${REALIO_PORT}065%g" $HOME/.realio-network/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${REALIO_PORT}658%g;
s%:26657%:${REALIO_PORT}657%g;
s%:6060%:${REALIO_PORT}060%g;
s%:26656%:${REALIO_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${REALIO_PORT}656\"%;
s%:26660%:${REALIO_PORT}660%g" $HOME/.realio-network/config/config.toml
```

# config pruning
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.realio-network/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.realio-network/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.realio-network/config/app.toml

# set minimum gas price, enable prometheus and disable indexing
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0ario"|g' $HOME/.realio-network/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.realio-network/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.realio-network/config/config.toml

# create service file
sudo tee /etc/systemd/system/realio-networkd.service > /dev/null <<EOF
[Unit]
Description=Realio node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.realio-network
ExecStart=$(which realio-networkd) start --home $HOME/.realio-network
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

# reset and download snapshot
realio-networkd tendermint unsafe-reset-all --home $HOME/.realio-network
if curl -s --head curl https://server-2.itrocket.net/mainnet/realio/realio_2024-12-29_10366132_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-2.itrocket.net/mainnet/realio/realio_2024-12-29_10366132_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.realio-network
    else
  echo "no snapshot found"
fi

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable realio-networkd
sudo systemctl restart realio-networkd && sudo journalctl -u realio-networkd -fo cat
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/mainnet/realio/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
realio-networkd keys add $WALLET

# to restore exexuting wallet, use the following command
realio-networkd keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(realio-networkd keys show $WALLET -a)
VALOPER_ADDRESS=$(realio-networkd keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
realio-networkd status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
realio-networkd query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.realio-network/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://realio-mainnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

  if ! [[ "$local_height" =~ ^[0-9]+$ ]] || ! [[ "$network_height" =~ ^[0-9]+$ ]]; then
    echo -e "\033[1;31mError: Invalid block height data. Retrying...\033[0m"
    sleep 5
    continue
  fi

  blocks_left=$((network_height - local_height))
  if [ "$blocks_left" -lt 0 ]; then
    blocks_left=0
  fi

  echo -e "\033[1;33mNode Height:\033[1;34m $local_height\033[0m \033[1;33m| Network Height:\033[1;36m $network_height\033[0m \033[1;33m| Blocks Left:\033[1;31m $blocks_left\033[0m"

  sleep 5
done
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, ario
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
realio-networkd tx staking create-validator \
--amount 1000000ario \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(realio-networkd tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id realionetwork_3301-1 \
 --gas auto --gas-adjustment 1.3 --fees 300000000ario \
-y
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
sudo ufw allow ${REALIO_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop realio-networkd
sudo systemctl disable realio-networkd
sudo rm -rf /etc/systemd/system/realio-networkd.service
sudo rm $(which realio-networkd)
sudo rm -rf $HOME/.realio-network
sed -i "/REALIO_/d" $HOME/.bash_profile
