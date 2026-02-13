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
    "mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
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
### 5.1 direct arguments settings
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
```
### 5.2 the way that makes config.toml
```
$ vi config.toml
# --- 데이터 및 스토리지 설정 ---
# 데이터 저장 경로
data-path="/opt/besu/data"
# 데이터 저장 방식 (BONZAI 또는 FOREST)
data-storage-format="BONZAI"
# 동기화 모드 (FAST, FULL, CHECKPOINT, SNAP)
sync-mode="X_SNAP"

# --- 네트워크 설정 ---
# P2P 통신을 위한 IP 주소 (0.0.0.0은 모든 인터페이스 허용)
p2p-interface="0.0.0.0"
# P2P 포트 (기본값 30303)
p2p-port=30303
# 부트노드 설정 (네트워크 참여를 위한 시작 지점)
# bootnodes=["enode://id@ip:port", "enode://id2@ip2:port"]

# --- API 및 보안 설정 ---
# HTTP JSON-RPC 활성화
rpc-http-enabled=true
rpc-http-host="0.0.0.0"
rpc-http-port=8545
# 허용할 API 메서드 그룹
rpc-http-api=["ETH", "NET", "WEB3", "TXPOOL", "ADMIN"]
# CORS 허용 범위 (브라우저 접속 보안)
rpc-http-cors-origins=["*"]

# WebSocket 활성화 (이벤트 구독 등에 필요)
rpc-ws-enabled=true
rpc-ws-host="0.0.0.0"
rpc-ws-port=8546

# --- 가스 및 트랜잭션 풀 ---
# 최소 가스 가격 (단위: Wei)
min-gas-price=1000000000
# 트랜잭션 풀 최대 유지 개수
tx-pool-max-size=4096

# --- 로깅 및 모니터링 ---
# 로그 레벨 (INFO, DEBUG, TRACE, ERROR)
logging="INFO"
# 프로메테우스 메트릭 활성화 (모니터링용)
metrics-enabled=true
metrics-host="0.0.0.0"
metrics-port=9545

$ nerdctl run -d --name besu-validator-1 \
  -p 8545:8545 \
  -p 30303:30303 \
  -v $(pwd)/network-files/genesis.json:/opt/besu/genesis.json \
  -v $(pwd)/network-files/keys/0x7934f8b79b86e22d9d43e5dcef71ccdb6fc1702f/key.priv:/opt/besu/key.priv \
  -v $(pwd)/data/besu-validator-1:/opt/besu/data \
  hyperledger/besu:24.1.1 \
  --config-file=config.toml
```

## 6. verify
```
$ nerdctl ps

$ nerdctl logs <CONTAINER ID> -n 100

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

# get last block number
$ curl -X POST --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' http://localhost:8545
{"jsonrpc":"2.0","id":1,"result":"0x10"}

# get enode
$ curl -X POST --data '{"jsonrpc":"2.0","method":"net_enode","params":[],"id":1}' http://localhost:8545
{"jsonrpc":"2.0","id":1,"result":"enode://5e38c4df77558817b256f8382f965bcfbe498c76e3b465220e8ca47fd03b4206c3abb9638c999080d0f6c3105ba4e0b0d2185b1368728d59bfb3a15fae8a3790@127.0.0.1:30303"}
```
