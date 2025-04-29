**Manual Installation**
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
VER="1.22.3"
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
echo "export PELL_CHAIN_ID="ignite_186-1"" >> $HOME/.bash_profile
echo "export PELL_PORT="58"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
wget -O pellcored https://github.com/0xPellNetwork/network-config/releases/download/v1.2.1/pellcored-v1.2.1-linux-amd64
chmod +x pellcored
mv pellcored ~/go/bin/
WASMVM_VERSION=v2.1.2
export LD_LIBRARY_PATH=~/.pellcored/lib
mkdir -p $LD_LIBRARY_PATH
wget "https://github.com/CosmWasm/wasmvm/releases/download/$WASMVM_VERSION/libwasmvm.$(uname -m).so" -O "$LD_LIBRARY_PATH/libwasmvm.$(uname -m).so"
echo "export LD_LIBRARY_PATH=$HOME/.pellcored/lib:$LD_LIBRARY_PATH" >> $HOME/.bash_profile
source ~/.bash_profile
```

**config and init app**
```
pellcored config node tcp://localhost:${PELL_PORT}657
pellcored config keyring-backend os
pellcored config chain-id ignite_186-1
pellcored init "test" --chain-id ignite_186-1
```

**download genesis and addrbook**
```
wget -O $HOME/.pellcored/config/genesis.json https://server-5.itrocket.net/testnet/pell/genesis.json
wget -O $HOME/.pellcored/config/addrbook.json  https://server-5.itrocket.net/testnet/pell/addrbook.json
```

**set seeds and peers**
```
SEEDS="5f10959cc96b5b7f9e08b9720d9a8530c3d08d19@pell-testnet-seed.itrocket.net:58656"
PEERS="d003cb808ae91bad032bb94d19c922fe094d8556@pell-testnet-peer.itrocket.net:58656,85ae72fe4ab8b669299f6da1baaaba8a6c664ec3@185.232.70.33:17656,2b2932bd000204b75d2675d84e0e6e690fcc9b41@31.165.179.107:26656,7fd83fe2a75067fd04aa8471a4ad2396134ee234@65.21.45.194:36656,e96ce110267ffffbbc9d8711cab37f2e34861f21@135.181.46.158:57656,73270186a4ed6a4136a2c02274867c0c41c304dd@46.4.91.76:30356,58c2ffb3e16f61462ebf26730cbc27b458ea82c0@34.44.209.45:26656,f487313950783ed57c5fcb9be1998b50a41bd931@37.27.127.220:57656,48c48532950e51fba80a1031d5a58c627737ed84@65.109.82.111:25656,4cad46992872f86da794f47ab662592bf9ca500a@135.181.79.242:57656,cc9acce34ac781b76be5c9f7d9f1ed307331de06@94.72.118.149:57656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.pellcored/config/config.toml
```

**set custom ports in app.toml**
```
sed -i.bak -e "s%:1317%:${PELL_PORT}317%g;
s%:8080%:${PELL_PORT}080%g;
s%:9090%:${PELL_PORT}090%g;
s%:9091%:${PELL_PORT}091%g;
s%:8545%:${PELL_PORT}545%g;
s%:8546%:${PELL_PORT}546%g;
s%:6065%:${PELL_PORT}065%g" $HOME/.pellcored/config/app.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${PELL_PORT}658%g;
s%:26657%:${PELL_PORT}657%g;
s%:6060%:${PELL_PORT}060%g;
s%:26656%:${PELL_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${PELL_PORT}656\"%;
s%:26660%:${PELL_PORT}660%g" $HOME/.pellcored/config/config.toml
```

**config pruning**
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.pellcored/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.pellcored/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"19\"/" $HOME/.pellcored/config/app.toml
```

**set minimum gas price, enable prometheus and disable indexing**
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0apell"|g' $HOME/.pellcored/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.pellcored/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.pellcored/config/config.toml
```

**create service file**
```
sudo tee /etc/systemd/system/pellcored.service > /dev/null <<EOF
[Unit]
Description=Pell node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.pellcored
ExecStart=$(which pellcored) start --home $HOME/.pellcored
Environment=LD_LIBRARY_PATH=$HOME/.pellcored/lib/
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**reset and download snapshot**
```
pellcored tendermint unsafe-reset-all --home $HOME/.pellcored
if curl -s --head curl https://server-5.itrocket.net/testnet/pell/pell_2025-04-09_1843275_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-5.itrocket.net/testnet/pell/pell_2025-04-09_1843275_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.pellcored
    else
  echo "no snapshot found"
fi
```

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable pellcored
sudo systemctl restart pellcored && sudo journalctl -u pellcored -fo cat
Automatic Installation
pruning: custom: 100/0/19 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/pell/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
pellcored keys add $WALLET

# to restore exexuting wallet, use the following command
pellcored keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(pellcored keys show $WALLET -a)
VALOPER_ADDRESS=$(pellcored keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
pellcored status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
pellcored query bank balances $WALLET_ADDRESS 
Node Sync Status Checker
#!/bin/bash
rpc_port=$(grep -m 1 -oP '^laddr = "\K[^"]+' "$HOME/.pellcored/config/config.toml" | cut -d ':' -f 3)
while true; do
  local_height=$(curl -s localhost:$rpc_port/status | jq -r '.result.sync_info.latest_block_height')
  network_height=$(curl -s https://pell-testnet-rpc.itrocket.net/status | jq -r '.result.sync_info.latest_block_height')

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
Amount, apell
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
echo "{\"pubkey\":{\"@type\":\"/cosmos.crypto.ed25519.PubKey\",\"key\":\"$(pellcored comet show-validator | grep -Po '\"key\":\s*\"\K[^"]*')\"},
    \"amount\": \"1000000apell\",
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
pellcored tx staking create-validator validator.json \
    --from $WALLET \
    --chain-id ignite_186-1 \
	--fees 30apell --gas 300000
	
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
sudo ufw allow ${PELL_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop pellcored
sudo systemctl disable pellcored
sudo rm -rf /etc/systemd/system/pellcored.service
sudo rm $(which pellcored)
sudo rm -rf $HOME/.pellcored
sed -i "/PELL_/d" $HOME/.bash_profile
