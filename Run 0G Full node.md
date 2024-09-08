# Run 0G Full node - Sync from 0 - Auto upgrade
This guide will help you install 0gchaind v.0.25 and automatically upgrade to the new version. To run 0gchaind and sync from block 0, you must start with version v.0.2.5

### System updates, installation of required environments:
```
sudo apt update
sudo apt install curl git make jq build-essential gcc unzip wget lz4 aria2 -y
```
### Install Go
```
cd $HOME && \
ver="1.22.0" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile && \
source ~/.bash_profile && \
go version
```
### Download and build Binary
```
cd $HOME
rm -rf 0g-chain
git clone https://github.com/0glabs/0g-chain.git
cd 0g-chain
git checkout v0.2.5
git submodule update --init
make install
0gchaind version
```
### Node init
Replace: `Your_Node_name`
```
0gchaind config keyring-backend os
0gchaind config chain-id zgtendermint_16600-2
0gchaind init "Your_Node_name" --chain-id zgtendermint_16600-2
0gchaind config node tcp://localhost:26657
```
### Download genesis.json file
```
rm ~/.0gchain/config/genesis.json
wget -P ~/.0gchain/config https://vps5.josephtran.xyz/0g/genesis.json
```
### PEERS, SEED setting
```
SEEDS="81987895a11f6689ada254c6b57932ab7ed909b6@54.241.167.190:26656,010fb4de28667725a4fef26cdc7f9452cc34b16d@54.176.175.48:26656,e9b4bc203197b62cc7e6a80a64742e752f4210d5@54.193.250.204:26656,68b9145889e7576b652ca68d985826abd46ad660@18.166.164.232:26656,8f21742ea5487da6e0697ba7d7b36961d3599567@og-testnet-seed.itrocket.net:47656"
PEERS="80fa309afab4a35323018ac70a40a446d3ae9caf@og-testnet-peer.itrocket.net:11656,90490155eb1e28a00cb9000657ef53cf9822e9e2@185.245.182.248:12656,6d0e4af8b817dbb81266d6c6710033896f1d65cb@158.220.103.216:12656,881b2297ac90fdf6803136101c1b33eeb52a0bcc@213.199.37.74:12656,2de20431412255201b960a0713c3a3f6fdbeb7e7@173.249.19.219:12656,9b6346424a9b1357bae659a51dbbb8d1c4d1366f@173.249.58.134:12656,055e3e65fd72102f389372564e0107e3ee5022fa@167.86.95.218:12656,0ada3d654c01607d585793943b37335a97a56691@213.239.195.210:12656,3f4ee55632cbd8694c7e5d173f10d7d7b23a5ec1@138.201.185.45:12656,2d780e7cae16cf25dfb992e294d5f672ae8a65ac@185.190.140.189:12656,85eec3750270e50ea73c46b1caa72e7110fa7b1b@156.67.81.129:12656,d619b3c8a0cc49b52ce68b45d8ebe2b9060a3f0a@149.50.111.193:12656,6ea4a3942152a33a50c54cc60aa311fd43cc71d7@144.91.93.99:12656,d7c847d92cf2714d3018cecd6476b6ef86b4240b@66.94.113.206:12656,85f1a5c5e62bbe59d9764453bf4624dc261a53f7@38.242.237.56:12656,1754dac0846c42ebe21fe1935eda0311d567d6a9@45.14.194.144:12656,7e49c7c5d8cf1a4f79d3a2c4a2c3597d144e638e@156.67.81.135:12656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.0gchain/config/config.toml
```
### Config pruning, set gas price, enable prometheus, disable indexer
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.0gchain/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.0gchain/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.0gchain/config/app.toml
```
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.00025ua0gi"|g' $HOME/.0gchain/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.0gchain/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.0gchain/config/config.toml
```
### Download cosmovisor script to install
```
wget https://raw.githubusercontent.com/0glabs/0g-chain/dev/networks/testnet/init-cosmovisor.sh
```
Check Script content 
`nano init-cosmovisor.sh`
```
#!/bin/bash

if [ -z "$1" ]; then
    echo "Usage: $0 <0G Home>"
    exit 1
fi

go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest

export DAEMON_NAME=0gchaind
echo "export DAEMON_NAME=0gchaind" >> ~/.profile
export DAEMON_HOME=$1
echo "export DAEMON_HOME=$1" >> ~/.profile
cosmovisor init $(whereis -b 0gchaind | awk '{print $2}')
mkdir $DAEMON_HOME/cosmovisor/backup
echo "export DAEMON_DATA_BACKUP_DIR=$DAEMON_HOME/cosmovisor/backup" >> ~/.profile
echo "export DAEMON_ALLOW_DOWNLOAD_BINARIES=true" >> ~/.profile
```
### Run script
Before running this script, make sure Go is loaded in your current shell session. If you haven't loaded Go yet, please run the following command:
```
source ~/.profile
chmod +x init-cosmovisor.sh
./init-cosmovisor.sh /root/.0gchain
```
Reload profile
```
source ~/.profile
```
### Create service file
```
sudo tee /etc/systemd/system/0gd.service > /dev/null <<EOF
[Unit]
Description=Cosmovisor 0G Node
After=network.target

[Service]
User=root
Type=simple
ExecStart=/root/go/bin/cosmovisor run start --log_output_console
Restart=on-failure
LimitNOFILE=65535
Environment="DAEMON_NAME=0gchaind"
Environment="DAEMON_HOME=/root/.0gchain"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="DAEMON_DATA_BACKUP_DIR=/root/.0gchain/cosmovisor/backup"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF
```
### Start service
```
sudo systemctl daemon-reload && \
sudo systemctl enable 0gd && \
sudo systemctl restart 0gd && sudo systemctl status 0gd
```
### Check log
```
sudo journalctl -u 0gd -f -o cat
```
```
tail -f /root/.0gchain/log/chain.log
```
