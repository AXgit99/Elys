**Manual Installation**
**Official Documentation**
**Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)**

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```

**install go, if needed**
```
cd $HOME
VER="1.21.3"
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
echo "export ELYS_CHAIN_ID="elystestnet-1"" >> $HOME/.bash_profile
echo "export ELYS_PORT="38"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binary**
```
cd $HOME
rm -rf elys
git clone https://github.com/elys-network/elys.git
cd elys
git checkout v0.41.1
make install
```

# config and init app
elysd config node tcp://localhost:${ELYS_PORT}657
elysd config keyring-backend os
elysd config chain-id elystestnet-1
elysd init "test" --chain-id elystestnet-1

# download genesis and addrbook
wget -O $HOME/.elys/config/genesis.json https://server-4.itrocket.net/testnet/elys/genesis.json
wget -O $HOME/.elys/config/addrbook.json  https://server-4.itrocket.net/testnet/elys/addrbook.json

# set seeds and peers
SEEDS="ae7191b2b922c6a59456588c3a262df518b0d130@elys-testnet-seed.itrocket.net:38656"
PEERS="0977dd5475e303c99b66eaacab53c8cc28e49b05@elys-testnet-peer.itrocket.net:38656,cc9c11f2c95ce2163d35b6cf9471ac9d61b7b9ac@65.108.131.146:26676,e1b058e5cfa2b836ddaa496b10911da62dcf182e@164.152.161.168:36656,5ad39c5d8fdefcc6eb740b9df62417991316d109@95.217.113.104:36656,40ec65e34f5800854c577bc9386ce82ed3fb4740@144.76.97.251:44656,60939e5760138c1db7cd3c587780ab6a643638e1@65.109.104.111:56102,ba32dca92f614ec2df20ea4e7a10ce4fa85edc46@51.79.18.14:26656,ae29d8da169214e201c03789858b4228b56a004a@148.251.177.108:22056,6012544a6bbdcab033abf731418fcc3351c37cff@49.12.150.42:26676,78aa6b222ae1f619bef03a9d98cb958dfcccc3a8@46.4.5.45:22056,eb0d0e8fe3010b86c09c5b94debd8c4719677422@167.235.12.38:07656,2583a957a6e337314f133b17b8147f5588c9a006@65.109.84.33:26656,bbf8ef70a32c3248a30ab10b2bff399e73c6e03c@65.21.198.100:21256,0f6914c83ae7eae97ec045ce518f11c567c8a2a0@167.235.13.19:27656,0b06ebe3437af5443ab8a4cf6881380fb29a35bb@142.132.194.124:11004"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.elys/config/config.toml


# set custom ports in app.toml
sed -i.bak -e "s%:1317%:${ELYS_PORT}317%g;
s%:8080%:${ELYS_PORT}080%g;
s%:9090%:${ELYS_PORT}090%g;
s%:9091%:${ELYS_PORT}091%g;
s%:8545%:${ELYS_PORT}545%g;
s%:8546%:${ELYS_PORT}546%g;
s%:6065%:${ELYS_PORT}065%g" $HOME/.elys/config/app.toml

# set custom ports in config.toml file
sed -i.bak -e "s%:26658%:${ELYS_PORT}658%g;
s%:26657%:${ELYS_PORT}657%g;
s%:6060%:${ELYS_PORT}060%g;
s%:26656%:${ELYS_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${ELYS_PORT}656\"%;
s%:26660%:${ELYS_PORT}660%g" $HOME/.elys/config/config.toml

# config pruning
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.elys/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.elys/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.elys/config/app.toml

# set minimum gas price, enable prometheus and disable indexing
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0018ibc/2180E84E20F5679FCC760D8C165B60F42065DEF7F46A72B447CFF1B7DC6C0A65,0.00025ibc/E2D2F6ADCC68AA3384B2F5DFACCA437923D137C14E86FB8A10207CF3BED0C8D4,0.00025uelys"|g' $HOME/.elys/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.elys/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.elys/config/config.toml

# create service file
sudo tee /etc/systemd/system/elysd.service > /dev/null <<EOF
[Unit]
Description=Elys node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.elys
ExecStart=$(which elysd) start --minimum-gas-prices="0.0018ibc/2180E84E20F5679FCC760D8C165B60F42065DEF7F46A72B447CFF1B7DC6C0A65,0.00025ibc/E2D2F6ADCC68AA3384B2F5DFACCA437923D137C14E86FB8A10207CF3BED0C8D4,0.00025uelys" --home $HOME/.elys
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

# reset and download snapshot
elysd tendermint unsafe-reset-all --home $HOME/.elys
if curl -s --head curl https://server-4.itrocket.net/testnet/elys/elys_2024-08-17_9287751_snap.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://server-4.itrocket.net/testnet/elys/elys_2024-08-17_9287751_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.elys
    else
  echo "no snapshot founded"
fi

# enable and start service
sudo systemctl daemon-reload
sudo systemctl enable elysd
sudo systemctl restart elysd && sudo journalctl -u elysd -f
Automatic Installation
pruning: custom: 100/0/10 | indexer: null

source <(curl -s https://itrocket.net/api/testnet/elys/autoinstall/)
Create wallet
# to create a new wallet, use the following command. don’t forget to save the mnemonic
elysd keys add $WALLET

# to restore exexuting wallet, use the following command
elysd keys add $WALLET --recover

# save wallet and validator address
WALLET_ADDRESS=$(elysd keys show $WALLET -a)
VALOPER_ADDRESS=$(elysd keys show $WALLET --bech val -a)
echo "export WALLET_ADDRESS="$WALLET_ADDRESS >> $HOME/.bash_profile
echo "export VALOPER_ADDRESS="$VALOPER_ADDRESS >> $HOME/.bash_profile
source $HOME/.bash_profile

# check sync status, once your node is fully synced, the output from above will print "false"
elysd status 2>&1 | jq 

# before creating a validator, you need to fund your wallet and check balance
elysd query bank balances $WALLET_ADDRESS 
Create validator
Moniker
Identity
Details
I love blockchain ❤️
Amount, uelys
1000000
Commission rate
0.1
Commission max rate
0.2
Commission max change rate
0.01
Website
elysd tx staking create-validator \
--amount 1000000uelys \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(elysd tendermint show-validator) \
--moniker "test" \
--identity "" \
--website "" \
--details "I love blockchain ❤️" \
--chain-id elystestnet-1 \
--gas auto --gas-adjustment 1.5 --fees 100uelys \
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
sudo ufw allow ${ELYS_PORT}656/tcp
sudo ufw enable
Delete node
sudo systemctl stop elysd
sudo systemctl disable elysd
sudo rm -rf /etc/systemd/system/elysd.service
sudo rm $(which elysd)
sudo rm -rf $HOME/.elys
sed -i "/ELYS_/d" $HOME/.bash_profile
