# 0g-evmos Testnet Manual Node Setup

## Update packages
```
sudo apt update && sudo apt upgrade -y
```

## Install dependencies
```
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```

## Install GO Version 1.21.3
```
cd $HOME && \
ver="1.21.3" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```
## Download and build binaries
```
git clone https://github.com/0glabs/0g-evmos.git
cd 0g-evmos
git checkout v1.0.0-testnet
make install
evmosd version
```
## Config app
```
# Customize if you need
echo 'export MONIKER="Node"' >> ~/.bash_profile
echo 'export CHAIN_ID="zgtendermint_9000-1"' >> ~/.bash_profile
echo 'export WALLET_NAME="wallet"' >> ~/.bash_profile
echo 'export RPC_PORT="26657"' >> ~/.bash_profile
source $HOME/.bash_profile
```
## Init app
```
cd $HOME
evmosd init $MONIKER --chain-id $CHAIN_ID
evmosd config chain-id $CHAIN_ID
evmosd config node tcp://localhost:$RPC_PORT
evmosd config keyring-backend os # You can change this
```
## Download genesis
```
wget https://github.com/0glabs/0g-evmos/releases/download/v1.0.0-testnet/genesis.json -O $HOME/.evmosd/config/genesis.json
```
## Set peers, gas prices and seeds
```
PEERS="1248487ea585730cdf5d3c32e0c2a43ad0cda973@peer-zero-gravity-testnet.trusted-point.com:26326" && \
SEEDS="8c01665f88896bca44e8902a30e4278bed08033f@54.241.167.190:26656,b288e8b37f4b0dbd9a03e8ce926cd9c801aacf27@54.176.175.48:26656,8e20e8e88d504e67c7a3a58c2ea31d965aa2a890@54.193.250.204:26656,e50ac888b35175bfd4f999697bdeb5b7b52bfc06@54.215.187.94:26656" && \
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.evmosd/config/config.toml
```
## Custom Port
```
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:10958\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:10957\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:10960\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:10956\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":10966\"%" $HOME/.evmosd/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:10917\"%; s%^address = \":8080\"%address = \":10980\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:10990\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:10991\"%; s%:8545%:10945%; s%:8546%:10946%; s%:6065%:10965%" $HOME/.evmosd/config/app.toml
```
## Config pruning
```
sed -i.bak -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.evmosd/config/app.toml
sed -i.bak -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.evmosd/config/app.toml
sed -i.bak -e "s/^pruning-interval *=.*/pruning-interval = \"10\"/" $HOME/.evmosd/config/app.toml
```
## Set indexer "null"
```
sed -i "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.00252aevmos\"/" $HOME/.evmosd/config/app.toml
```
## Create service
```
sudo tee /etc/systemd/system/ogd.service > /dev/null <<EOF
[Unit]
Description=OG Node
After=network.target

[Service]
User=$USER
Type=simple
ExecStart=$(which evmosd) start --home $HOME/.evmosd
Restart=on-failure
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
## Register and start service
```
sudo systemctl daemon-reload
sudo systemctl enable ogd
sudo systemctl restart ogd && sudo journalctl -u ogd -f -o cat
```
## Post installation
When installation is finished please load variables into system
```
source $HOME/.bash_profile
```
Next you have to make sure your validator is syncing blocks. You can use command below to check synchronization status
```
evmosd status 2>&1 | jq .SyncInfo
```
### Option (1) Create wallet
To create new wallet you can use command below. Don’t forget to save the mnemonic
```
evmosd keys add $WALLET
```

(OPTIONAL) To recover your wallet using seed phrase
```
evmosd keys add $WALLET --recover
```

To get current list of wallets
```
evmosd keys list
```

### Option (2) Import Metamask wallet
```
evmosd keys unsafe-import-eth-key $WALLET <private-key-eth> --keyring-backend file
```
### Create validator
Before creating validator please make sure that you have at least 1 planq (1 evmosd is equal to 100000000000000000aevmos ) and your node is synchronized

To check your wallet balance:
```
evmosd query bank balances $PLANQD_WALLET_ADDRESS
```
> If your wallet does not show any balance than probably your node is still syncing. Please wait until it finish to synchronize and then continue

To create your validator run command below
```
evmosd tx staking create-validator \
  --amount=10000000000000000aevmos  \
  --from=$WALLET \
  --pubkey=$(evmosd tendermint show-validator) \
  --moniker=$NODENAME \
  --chain-id=zgtendermint_9000-1 \
  --identity=<your-id>\
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1000000" \
  --gas="500000" \
  --gas-prices="99999aevmos" \
  --gas-adjustment="1.15"
  -y
```

## Reset chain data
```
evmosd tendermint unsafe-reset-all --home $HOME/.evmosd --keep-addr-book
```


### Basic Firewall security
Start by checking the status of ufw.
```
sudo ufw status
```

### Check your validator key
```
[[ $(evmosd q staking validator $EVMOSD_VALOPER_ADDRESS -oj | jq -r .consensus_pubkey.key) = $(evmosd status | jq -r .ValidatorInfo.PubKey.value) ]] && echo -e "\n\e[1m\e[32mTrue\e[0m\n" || echo -e "\n\e[1m\e[31mFalse\e[0m\n"
```

## Usefull commands
### Service management
Check logs
```
journalctl -fu evmosd -o cat
```

Start service
```
sudo systemctl start evmosd
```

Stop service
```
sudo systemctl stop evmosd
```

Restart service
```
sudo systemctl restart evmosd
```

### Node info
Synchronization info
```
evmosd status 2>&1 | jq .SyncInfo
```

Validator info
```
evmosd status 2>&1 | jq .ValidatorInfo
```

Node info
```
evmosd status 2>&1 | jq .NodeInfo
```

Show node id
```
evmosd tendermint show-node-id
```
### Voting
```
evmosd tx gov vote 1 yes --from $WALLET --chain-id=zgtendermint_9000-1
```

### Staking, Delegation and Rewards
Delegate stake
```
evmosd tx staking delegate $EVMOSDD_VALOPER_ADDRESS 1000000000000aevmos --from=$WALLET --chain-id=zgtendermint_9000-1 --gas=auto
```

Redelegate stake from validator to another validator
```
evmosd tx staking redelegate <srcValidatorAddress> <destValidatorAddress> 1000000000000aevmos --from=$WALLET --chain-id=zgtendermint_9000-1 --gas=auto
```

Withdraw all rewards
```
evmosd tx distribution withdraw-all-rewards --from=$WALLET --chain-id=zgtendermint_9000-1 --gas=auto
```

Withdraw rewards with commision
```
evmosd tx distribution withdraw-rewards $EVMOSD_VALOPER_ADDRESS --from=$WALLET --commission --chain-id=zgtendermint_9000-1
```
### Validator management
Edit validator
```
evmosd tx staking edit-validator \
  --moniker=$NODENAME \
  --identity=<your_keybase_id> \
  --website="<your_website>" \
  --details="<your_validator_description>" \
  --chain-id=zgtendermint_9000-1 \
  --from=$WALLET
```

Unjail validator
```
evmosd tx slashing unjail \
  --broadcast-mode=block \
  --from=$WALLET \
  --chain-id=zgtendermint_9000-1 \
  --gas=auto
```

### Delete node
This commands will completely remove node from server. Use at your own risk!
```
sudo systemctl stop evmosd && \
sudo systemctl disable evmosd && \
rm /etc/systemd/system/evmosd.service && \
sudo systemctl daemon-reload && \
cd $HOME && \
rm -rf 0g-evmos && \
rm -rf .evmosd && \
rm -rf $(which evmosd)
