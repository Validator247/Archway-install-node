# Archway-install-node

Update system

    sudo apt update
    sudo apt-get install git curl build-essential make jq gcc snapd chrony lz4 tmux unzip bc -y

Install Go

    rm -rf $HOME/go
    sudo rm -rf /usr/local/go
    cd $HOME
    curl https://dl.google.com/go/go1.20.1.linux-amd64.tar.gz | sudo tar -C/usr/local -zxvf -
    cat <<'EOF' >>$HOME/.profile
    export GOROOT=/usr/local/go
    export GOPATH=$HOME/go
    export GO111MODULE=on
    export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
    EOF
    source $HOME/.profile
    go version

Install Node

    cd $HOME
    rm -rf archway
    git clone https://github.com/archway-network/archway.git
    cd archway
    git checkout v4.0.2
    make install
    archwayd version

Initialize Node

    archwayd init NodeName --chain-id=archway-1

Download Genesis

    curl -Ls https://ss.archway.nodestake.top/genesis.json > $HOME/.archway/config/genesis.json 

Download addrbook

    curl -Ls https://ss.archway.nodestake.top/addrbook.json > $HOME/.archway/config/addrbook.json

Create Service

    sudo tee /etc/systemd/system/archwayd.service > /dev/null <<EOF
    [Unit]
    Description=archwayd Daemon
    After=network-online.target
    [Service]
    User=$USER
    ExecStart=$(which archwayd) start
    Restart=always
    RestartSec=3
    LimitNOFILE=65535
    [Install]
    WantedBy=multi-user.target
    EOF
    sudo systemctl daemon-reload
    sudo systemctl enable archwayd

Download Snapshot(optional)

    SNAP_NAME=$(curl -s https://ss.archway.nodestake.top/ | egrep -o ">20.*\.tar.lz4" | tr -d ">")
    curl -o - -L https://ss.archway.nodestake.top/${SNAP_NAME}  | lz4 -c -d - | tar -x -C $HOME/.archway

Launch Node

    sudo systemctl restart archwayd
    journalctl -u archwayd -f

Get Sync Status

    archwayd status 2>&1 | jq .SyncInfo.catching_up

Get Latest Height

    archwayd status 2>&1 | jq .SyncInfo.latest_block_height

# Add New Key & Create a validator

Add New Key

    archwayd keys add wallet

Recover Existing Key

    archwayd keys add wallet --recover

Query Wallet Balance

    archwayd q bank balances $(archwayd keys show wallet -a)

Create a validator

    archwayd tx staking create-validator \
    --amount 500000000000aarch \
    --pubkey $(archwayd tendermint show-validator) \
    --moniker "Name" \
    --chain-id archway-1 \
    --commission-rate 0.05 \
    --commission-max-rate 0.20 \
    --commission-max-change-rate 0.01 \
    --min-self-delegation 1 \
    --from=wallet \
    --gas-adjustment 1.4 \
    --gas auto \
    --gas-prices 1000000000000aarch \
    -y

# Command

 Save to wallet.backup

    archwayd keys export wallet

Import Key

    archwayd keys import wallet wallet.backup

Unjail Validator

    archwayd tx staking delegate $(archwayd keys show wallet --bech val -a) 1000000000000aarch --from wallet --chain-id archway-1 --gas-prices 1000000000000aarch --gas-adjustment 1.5 --gas auto -y

Delegate to yourself

    archwayd tx staking delegate $(archwayd keys show wallet --bech val -a) 1000000000000aarch --from wallet --chain-id archway-1 --gas-prices 1000000000000aarch --gas-adjustment 1.5 --gas auto -y

Reload systemD

    sudo systemctl enable archwayd 
    sudo systemctl daemon-reload
    sudo systemctl restart archwayd && journalctl -u archwayd -f --no-hostname -o cat

        
