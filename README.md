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
      "berlinBlock": 0,
      "londonBlock": 0,
      "zeroBaseFee": true,
      "baseFeePerGas": "0x0",
      "qbft": {
        "blockperiodseconds": 2,
        "epochlength": 30000,
        "requesttimeoutseconds": 10
      }
    },
    "gasLimit": "0x1fffffffffffff",
    "difficulty": "0x1",
    "nonce": "0x0",
    "timestamp": "0x58e81039",
    "gasLimit": "0x1fffffffffffff",
    "difficulty": "0x1",
    "mixHash": "0x63746963616c2062797a616e74696e65206661756c7420746f6c6572616e6365",
    "coinbase": "0x0000000000000000000000000000000000000000",
  },
  "blockchain": {
    "nodes": {
      "generate": true,
      "count": 1
    }
  }
}
```
## 4. create base files for private network
```
$ besu operator generate-blockchain-config \
                --config-file=make-genesis.json \
                --to=network-files
$ tree
.
├── besu-24.1.1.tar.gz
├── data
│   └── besu-validator-1
├── network-files
│   ├── genesis.json
│   └── keys
│       └── 0x7934f8b79b86e22d9d43e5dcef71ccdb6fc1702f
│           ├── key.priv
│           └── key.pub
└── make-genesis.json

# make extraData just in case
$ vi make-encode.json
[
  "0x7934f8b79b86e22d9d43e5dcef71ccdb6fc1702f"  # this is an address from the priv/pub key above  
]
$ besu rlp encode --type=QBFT_EXTRA_DATA --from=make-encode.json
0xf83aa00000000000000000000000000000000000000000000000000000000000000000d594ce8adef2a32e9eb57c00494b712195690d30dde9c080c0
```

## 5. install besu with nerdctl
```
$ nerdctl run -d --name besu-validator-1 \
  -p 8545:8545 \
  -p 30303:30303 \
  -v $(pwd)/network-files/genesis.json:/opt/besu/genesis.json \
  -v $(pwd)/network-files/keys/0x7934f8b79b86e22d9d43e5dcef71ccdb6fc1702f/key.priv:/opt/besu/key.priv \
  -v $(pwd)/data/besu-validator-1:/opt/besu/data \
  hyperledger/besu:24.1.1 \
  --data-path=/opt/besu/data \
  --genesis-file=/opt/besu/genesis.json \
  --node-private-key-file=/opt/besu/key.priv \
  --rpc-http-enabled=true \
  --rpc-http-host="0.0.0.0" \
  --rpc-http-cors-origins="*" \
  --rpc-http-api=ETH,NET,WEB3,ADMIN,DEBUG,MINER,NET,PERM,TXPOOL,QBFT

$ nerdctl ps

$ nerdctl logs <CONTAINER ID> -n 100
```

## 6. verify
```
$ tree ./data
./data
└── besu-validator-1
    ├── DATABASE_METADATA.json
    ├── besu.networks
    ├── besu.ports
    ├── caches
    │   ├── CACHE_METADATA.json
    │   └── logBloom-0.cache
    ├── database
    │   ├── 000004.log
    │   ├── CURRENT
    │   ├── IDENTITY
    │   ├── LOCK
    │   ├── LOG
    │   ├── MANIFEST-000005
    │   ├── OPTIONS-000055
    │   └── OPTIONS-000057
    └── uploads

$ curl -X POST --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' http://localhost:8545
{"jsonrpc":"2.0","id":1,"result":"0x10"}
```
