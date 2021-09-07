# IBC challenge for Kichain
## Cross-network transactions

The task facing us is to build an cross-chain bridge and make cross-network transactions.
To do this we need to install and configure the Cosmos IBC relayer, and we need a second network that supports IBC transactions. For this, I used the recently launched Umee test network.
___

Before I started installing and configuring the relayer, I made changes in config.toml on my nodes in both networks. In the rpc section "rpc-addr": "http://127.0.0.1:26657" changed to "rpc-addr": "http://0.0.0.0:26657".

The relayer was installed on the Kichane network node.
___

### 1. Download and install the relayer:
* git clone https://github.com/cosmos/relayer.git 
* cd relayer
* make install
* cd
* rly version
> version: 1.0.0-rc1-152-g112205b\
commit: 112205bab8224295cb046badcda512d37ea1c7b8\
ccosmos-sdk: v0.42.4\
cgo: go1.16.5 linux/amd64

### 2. Initialize the relayer's configuration:
* rly config init

### 3. Create and navigate to the folder where the settings for our networks will be stored:
* mkdir rly_config
* cd rly_config

### 4. Create a file for the Kichane network and write the necessary settings there:
* mcedit kichain-t-4_config.json\
#In the file we write the following
```
{
  "chain-id": "kichain-t-4",
  "rpc-addr": "http://0.0.0.0:26657", 
  "account-prefix": "tki",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025utki",
  "trusting-period": "48h"
}
```

### 5. Similarly for the Umee network:
* mcedit umee-betanet-1_config.json\
#In the file we write the following

```
{
  "chain-id": "umee-betanet-1",
  "rpc-addr": "http://65.108.58.202:26657", 
  "account-prefix": "umee",
  "gas-adjustment": 1.5,
  "gas-prices": "0.025uumee",
  "trusting-period": "48h"
}
```

### 6. Parsing these settings into the relayer's config:
* rly chains add -f kichain-t-4_config.json
* rly chains add -f umee-betanet-1_config.json
* cd

### 7. Either restore or create new keys for the relayer to use when signing and relaying transactions:
* rly keys add kichain-t-4 <ki_wallet_name>
* rly keys add umee-betanet-1 <umee_wallet_name>

or restore:
* rly keys restore kichain-t-4 <ki_wallet_name> "ki_mnemonic"
* rly keys restore umee-betanet-1 <umee_wallet_name> "umee_mnemonic"

### 8. Add the keys to the relayer's config:
* rly chains edit kichain-t-4 key <ki_wallet_name>
* rly chains edit umee-betanet-1 key <umee_wallet_name>

### 9. Now both of these accounts need to be topped up with test tokens to be able to make cross-network transfers:
Since I maintain validator nodes in both test networks, I had the opportunity to top up these accounts with test tokens using kid and umeed instead.

### 10. Ensure both relayer accounts are funded by querying each:
* rly q balance kichain-t-4
* rly q balance umee-betanet-1

### 11. After the tokens arrive in the relay wallets, we initialize the light clients for both networks:
* rly light init kichain-t-4 -f
* rly light init umee-betanet-1 -f

### 12. Trying to generate a new path representing a client, connection, channel and a specific port between the two networks:
* rly paths generate kichain-t-4 umee-betanet-1 transfer --port=transfer

We should see the following response
> Generated path(transfer), run 'rly paths show transfer --yaml' to see details

Check if the channel was created
* rly paths list -d

### 13. If the channel doesn't open, go to config:
* mcedit ~/.relayer/config/config.yaml

and erase in the paths: section the values in the following lines:\
`client-id: ***  connection-id: ***  channel-id: ***`

### 14. Initialize the light clients on each network again:
* rly light init kichain-t-4 -f
* rly light init umee-betanet-1 -f

### 15. Run the command:
* rly tx link transfer --debug

You may have to repeat steps 13-14 several times until after this command we will see:
> ★ Channel created: [kichain-t-4]chan{channel-61}port{transfer} -> [umee-betanet-1]chan{channel-0}port{transfer}

### 16. Check the created channel:
* rly paths list -d

The output should be like this (there should be check marks everywhere):\
> 0: transfer             -> chns(✔) clnts(✔) conn(✔) chan(✔) (kichain-t-4:transfer<>umee-betanet-1:transfer)

  ###### 16.1. Check networks:
* rly chains list\
> 0: kichain-t-4          -> key(✔) bal(✔) light(✔) path(✔)\
1: umee-betanet-1       -> key(✔) bal(✔) light(✔) path(✔)

  ###### 16.2. Check Path:
* rly paths show transfer\
> Path "transfer" strategy(naive):\
  SRC(kichain-t-4)\
    ClientID:     07-tendermint-13\
    ConnectionID: connection-17\
    ChannelID:    channel-61\
    PortID:       transfer\
  DST(umee-betanet-1)\
    ClientID:     07-tendermint-1\
    ConnectionID: connection-1\
    ChannelID:    channel-0\
    PortID:       transfer\
  STATUS:\
    Chains:       ✔\
    Clients:      ✔\
    Connection:   ✔\
    Channel:      ✔
    
### 17. Well, now we can try cross-network transactions:
* rly transact transfer [src-chain-id] [dst-chain-id] [amount] [dst-addr] [flags]

###### 17.1. Trying a transfer from the Kichane network to the Umee network:
* rly tx transfer kichain-t-4 umee-betanet-1 10utki umee1y50nrt5jdz6nec2h56wflsmelam6ev0dggnswd --path transfer --d

> I[2021-09-07|10:15:51.413] ✔ [kichain-t-4]@{228815} - msg(0:transfer) hash(CCAEC22E665A8F2C72091DEDB0A17C0056EE4645792E49C6A0E638293911AA3F)

https://api-challenge.blockchain.ki/txs/CCAEC22E665A8F2C72091DEDB0A17C0056EE4645792E49C6A0E638293911AA3F

###### 17.2. Trying a transfer from the Umee network to the Kichane network:
* rly tx transfer umee-betanet-1 kichain-t-4 100uumee tki1wwswak4j34zshnpcg004svg2hkmld5j6mekczv --path transfer --d

> I[2021-09-07|12:26:39.238] ✔ [umee-betanet-1]@{210073} - msg(0:transfer) hash(31D952E68A4BF5B1C1B5FE9B6FAF602921CBE17D3B24505687B943135D8734B6)

After making cross-network transfers, we can check the wallet balance and see that tokens from another network have appeared on it:
* kid query bank balances tki1wwswak4j34zshnpcg004svg2hkmld5j6mekczv
```
balances:
- amount: "101"\
  denom: ibc/ED55188C6FEA7596E71CDDE09238D9703E9052A5F346C7A9F4498ECD917242ED\
- amount: "13070"\
  denom: utki\
pagination:\
  next_key: null\
  total: "0"
```
  
### 18. To maintain the operation of the relayer, we will create relayer client:

#Create a service file:
```
sudo tee /etc/systemd/system/rlyd.service > /dev/null <<EOF 
[Unit] 
Description=relayer client 
After=network-online.target, kichaind.service
[Service] 
User=$USER 
ExecStart=$(which rly) start transfer 
Restart=always RestartSec=3 
LimitNOFILE=65535 
[Install] 
WantedBy=multi-user.target 
EOF
```
#Start the service file
* sudo systemctl daemon-reload
* sudo systemctl enable rlyd
* sudo systemctl start rlyd

#To check the logs of the relayer, use:
* journalctl -u rlyd -f

### 19. Trying transfers using kid and umeed:

```
kid tx ibc-transfer transfer transfer channel-N umee_WALLET_address 10utki ---from name_of_wallet --fees=200utki --gas=auto --chain-id kichain-t-4 --home $HOME/kichain/kid
```
https://api-challenge.blockchain.ki/txs/BE764F14A8FC0290520E250C706E0C8F445F1B57EC0F7B45ACF1187C039CD809

