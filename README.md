# Building BESU in private network
## 1. download
```
# 1. download
$ wget https://hyperledger.jfrog.io/artifactory/besu-binaries/besu/24.1.1/besu-24.1.1.tar.gz

# 2. unzip
$ tar -xvzf besu-24.1.1.tar.gz

# 3. check version
$ sudo mv besu-24.1.1 /opt/besu
$ echo 'export PATH=$PATH:/opt/besu/bin' >> ~/.bashrc
$ source ~/.bashrc
$ besu --version

# 4. install jdk 17 or 21
$ sudo apt update && sudo apt install openjdk-21-jdk -y

```
## 2. create directories
```
$ mkdir network-files
$ mkdir -p ./data/besu-validator-1
```
## 3. create file of make-genesis.json
```
$ vi make-genesis.json
{
  "genesis": {
    "config": {
      "chainId": 12345,
      "qbft": {
        "blockperiodseconds": 2,
        "epochlength": 30000,
        "requesttimeoutseconds": 10
      }
    },
    "gasLimit": "0x1fffffffffffff",
    "difficulty": "0x1"
  },
  "blockchain": {
    "nodes": {
      "generate": true,
      "count": 2
    }
  }
}
```
## 4. create base files for private network
```
$ besu operator generate-blockchain-config \
                --config-file=to-patch.json \
                --to=network-files
$ tree
.
├── besu-24.1.1.tar.gz
├── data
│   └── besu-validator-1
├── network-files
│   ├── genesis.json
│   └── keys
│       ├── 0xce8adef2a32e9eb57c00494b712195690d30dde9
│       │   ├── key.priv
│       │   └── key.pub
│       └── 0xef5444fbaf1ceee0b9c246b987d0c13113a9b58e
│           ├── key.priv
│           └── key.pub
└── to-patch.json
```

## 5. install besu with nerdctl
```
$ nerdctl run -d --name besu-validator-1 \
  -p 8545:8545 \
  -p 30303:30303 \
  -v $(pwd)/network-files/genesis.json:/opt/besu/genesis.json \
  -v $(pwd)/network-files/keys/0xce8adef2a32e9eb57c00494b712195690d30dde9/key.priv:/opt/besu/key.priv \
  -v $(pwd)/data/besu-validator-1:/opt/besu/data \
  hyperledger/besu:24.1.1 \
  --data-path=/opt/besu/data \
  --genesis-file=/opt/besu/genesis.json \
  --node-private-key-file=/opt/besu/key.priv \
  --rpc-http-enabled=true \
  --rpc-http-host="0.0.0.0" \
  --rpc-http-cors-origins="*" \
  --rpc-http-api=ETH,NET,WEB3,ADMIN,DEBUG,MINER,NET,PERM,TXPOOL,QBFT

$ nerdctl ps -a
```
