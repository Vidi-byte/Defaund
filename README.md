# Defund
v0.2.1

#### Setup validator name (replace the field "YOUR_MONIKER_GOES_HERE" with your own)
```
MONIKER="YOUR_MONIKER_GOES_HERE"
```
### Install dependencies
#### Update system and install build tools
```
sudo apt update
sudo apt install curl git jq lz4 build-essential -y
```
#### Install GO
```
sudo rm -rf /usr/local/go
sudo curl -Ls https://golang.org/dl/go1.19.4.linux-amd64.tar.gz | sudo tar -C /usr/local -xz
tee -a $HOME/.profile > /dev/null << EOF
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source $HOME/.profile
```
#### Download and build binaries
```
cd $HOME
rm -rf defund
git clone https://github.com/defund-labs/defund.git
cd defund
git checkout v0.2.1
make build
mkdir -p $HOME/.defund/cosmovisor/genesis/bin
mv build/defundd $HOME/.defund/cosmovisor/genesis/bin/
rm -rf build
```
#### Install Cosmovisor and create a service
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.4.0
sudo tee /etc/systemd/system/defundd.service > /dev/null << EOF
[Unit]
Description=defund-testnet node service
After=network-online.target
[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.defund"
Environment="DAEMON_NAME=defundd"
Environment="UNSAFE_SKIP_BACKUP=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable defundd
ln -s $HOME/.defund/cosmovisor/genesis $HOME/.defund/cosmovisor/current
sudo ln -s $HOME/.defund/cosmovisor/current/bin/defundd /usr/local/bin/defundd
```
#### Initialize the node
```
defundd config chain-id defund-private-3
defundd config keyring-backend test
defundd config node tcp://localhost:40657
defundd init $MONIKER --chain-id defund-private-3
curl -Ls https://snapshots.kjnodes.com/defund-testnet/genesis.json > $HOME/.defund/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/defund-testnet/addrbook.json > $HOME/.defund/config/addrbook.json
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@defund-testnet.rpc.kjnodes.com:40659\"|" $HOME/.defund/config/config.toml
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0ufetf\"|" $HOME/.defund/config/app.toml
sed -i -e "s|^pruning *=.*|pruning = \"custom\"|" $HOME/.defund/config/app.toml
sed -i -e "s|^pruning-keep-recent *=.*|pruning-keep-recent = \"100\"|" $HOME/.defund/config/app.toml
sed -i -e "s|^pruning-keep-every *=.*|pruning-keep-every = \"0\"|" $HOME/.defund/config/app.toml
sed -i -e "s|^pruning-interval *=.*|pruning-interval = \"19\"|" $HOME/.defund/config/app.toml
sed -i.bak -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:40658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:40657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:40060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:40656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":40660\"%" $HOME/.defund/config/config.toml
sed -i.bak -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:40317\"%; s%^address = \":8080\"%address = \":40080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:40090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:40091\"%; s%^address = \"0.0.0.0:8545\"%address = \"0.0.0.0:40545\"%; s%^ws-address = \"0.0.0.0:8546\"%ws-address = \"0.0.0.0:40546\"%" $HOME/.defund/config/app.toml
```
#### Download latest chain snapshot
```
curl -L https://snapshots.kjnodes.com/defund-testnet/snapshot_latest.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.defund
```
#### Start service and check the logs
```
sudo systemctl start defundd && journalctl -u defundd -f --no-hostname -o cat
```
#### Check node synchronization, if results false – node is synchronized
```
curl -s localhost:26657/status | jq .result.sync_info.catching_up
```
## Below enter one of the commands to create a new wallet or restore the old one depending on what you need. 
#### Add new key 
```
defundd keys add wallet
```
#### Recover existing key
```
defundd keys add wallet --recover
```
#### Wait 1-2 hours until the node is fully synchronized and create a validator 
## Please make sure you have adjusted moniker, identity, details and website to match your values. (You can leave just "YOUR_MONIKER_NAME" and other values such as "YOUR_KEYBASE_ID", "YOUR_DETAILS", "YOUR_WEBSITE_URL" just erase if you do not have them.)  
```
defundd tx staking create-validator \
--amount=1000000ufetf \
--pubkey=$(defundd tendermint show-validator) \
--moniker="YOUR_MONIKER_NAME" \
--identity="YOUR_KEYBASE_ID" \
--details="YOUR_DETAILS" \
--website="YOUR_WEBSITE_URL"
--chain-id=defund-private-3 \
--commission-rate=0.05 \
--commission-max-rate=0.20 \
--commission-max-change-rate=0.01 \
--min-self-delegation=1 \
--from=wallet \
--gas-adjustment=1.4 \
--gas=auto \
-y
```
