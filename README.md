# Side-testnet-2

Discord проекта https://discord.com/invite/sideprotocol

Документация - https://docs.side.one/infra-and-app/side-blockchain/join-testnet/side-node-installation-and-setup


Minimum Requirements
CPU: 4 cores
RAM: 8 GB
Storage: 200 GB
Network: 1 Gbp

Recommended Specifications
CPU: 8 cores
RAM: 16 GB
Storage: 500 GB
Network: 1 Gbp

#### ДЕДИК как вариант заказать здесь https://powervps.net/ru/?from=91820
#### Виртуалку как вариант заказать здесь https://pq.hosting/?from=36405


## Обновление пакетов сервера и подготовка к развертыванию ноды
```
sudo apt update && sudo apt upgrade -y
```

Setup Validator Name
First change “Имя вашего валидатора” to your chosen validator moniker and enter this command:
```
MONIKER="Имя_вашего_валидатора"
```
Install Dependencies & Install GO
Install Build Tools
```
sudo apt -qy install curl git jq lz4 build-essential
```
Install GO
```
ver="1.22.0"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

Download And Build Binaries
```
# Clone project repository
cd $HOME 
git clone https://github.com/sideprotocol/side.git
cd side
git checkout v0.6.0
```
```
# Build binaries*** make install
make build
```
```
# Prepare binaries for Cosmovisor
mkdir -p ~/.side
mkdir -p ~/.side/cosmovisor
mkdir -p ~/.side/cosmovisor/genesis
mkdir -p ~/.side/cosmovisor/genesis/bin
mkdir -p ~/.side/cosmovisor/upgrades
```
```
mv build/sided $HOME/.side/cosmovisor/genesis/bin/
rm -rf build
```
```
# Create application symlinks
sudo ln -s $HOME/.side/cosmovisor/genesis $HOME/.side/cosmovisor/current -f
sudo ln -s $HOME/.side/cosmovisor/genesis/bin/sided /usr/local/bin/sided -f

```

Set Up Cosmovisor And Create The Corresponding Service
```
# Download and install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
```
```
sided config chain-id side-testnet-2
sided config keyring-backend test
sided config node tcp://localhost:26357
```
Initialize The Node
```
# Initialize the node
sided init $MONIKER --chain-id side-testnet-2
```


```
# Create and start service
sudo tee /etc/systemd/system/sided.service > /dev/null <<EOF
[Unit]
Description=Babylon daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start --x-crisis-skip-assert-invariants
Restart=always
RestartSec=3
LimitNOFILE=infinity

Environment="DAEMON_NAME=sided"
Environment="DAEMON_HOME=${HOME}/.side"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=false"

[Install]
WantedBy=multi-user.target
EOF
```



```
sudo systemctl daemon-reload
sudo systemctl enable sided.service
```

```
# Download genesis and addrbook files
curl -L https://snapshots-testnet.nodejumper.io/side-testnet/genesis.json > $HOME/.side/config/genesis.json
curl -L https://snapshots-testnet.nodejumper.io/side-testnet/addrbook.json > $HOME/.side/config/addrbook.json
```

```
# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "d9911bd0eef9029e8ce3263f61680ef4f71a87c4@13.230.121.124:26656,693bdfec73a81abddf6f758aa49321de48456a96@13.231.67.192:26656,9c14080752bdfa33f4624f83cd155e2d3976e303@side-testnet-seed.itrocket.net:45656"|' $HOME/.side/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.005uside"|' $HOME/.side/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.side/config/app.toml
```
```
# Change ports
sed -i -e "s%:1317%:26317%; s%:8080%:26380%; s%:9090%:26390%; s%:9091%:26391%; s%:8545%:26345%; s%:8546%:26346%; s%:6065%:26365%" $HOME/.side/config/app.toml
sed -i -e "s%:26658%:26358%; s%:26657%:26357%; s%:6060%:26360%; s%:26656%:26356%; s%:26660%:26361%" $HOME/.side/config/config.toml
```

```
# Download latest chain data snapshot
curl "https://snapshots-testnet.nodejumper.io/side-testnet/side-testnet_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.side"
```

Start Service And Check The Logs
```
sudo systemctl daemon-reload
sudo systemctl enable sided.service

# Start the service and check the logs
sudo systemctl start sided.service
sudo journalctl -u sided.service -f --no-hostname -o cat
```



Becoming a Validator

```
# Create a New Key
sided keys add wallet
```
или восстанавливаем прошлый
```
sided keys add wallet --recover
```
вводим сид фразу от кошелька

Obtain Funds from the Babylon Testnet Faucet

### идем на сайт и запрашиваем токены

https://testnet.side.one/faucet

или в дискорде в ветке крана

$request side-testnet-2 side___ВАШ_АДРЕС____u2wq

### проверяем их на кошельке
```
sided q bank balances $(sided keys show wallet -a)
```

### проверяем логи


```
sudo journalctl -u sided.service -f --no-hostname -o cat
```


Create the Validator

```
sided status | jq .SyncInfo
```

https://testnet.side.explorers.guru/validators

### создаем валидатора c 1 токеном
```
sided tx staking create-validator \
--amount=1000000uside \
--pubkey=$(sided tendermint show-validator) \
--moniker "ВАШ_МОНИКЕР" \
--details "https://t.me/WingsNodeTeam" \
--website "@WingsNodeTeam" \
--chain-id side-testnet-2 \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.05 \
--min-self-delegation 1 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0.005uside \
-y
```



### делегировать себе 1 токен
```
sided tx staking delegate YOUR_TO_VALOPER_ADDRESS 1000000uside --from wallet --chain-id side-testnet-2 --gas-prices 0.005uside --gas-adjustment 1.5 --gas auto -y 
```
### отправить с кошелька на кошелек 1 токен

```
sided tx bank send АДРЕС_ОТКУДА АДРЕСС_КУДА 1000000uside --from wallet --chain-id side-testnet-2 --gas-prices 0.005uside --gas-adjustment 1.5 --gas auto -y 
```

### выйти из тюрьмы

```
sided tx slashing unjail --from wallet --chain-id side-testnet-2 --gas-prices 0.005uside --gas-adjustment 1.5 --gas auto -y 
```
