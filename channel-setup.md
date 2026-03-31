# TRST01 (1 Org) + 3-Node RAFT (oatschannel) - OATS Cluster

This repo layout implements a Fabric test network with:

- **Org:** `TRST01` (`Trst01MSP`)
- **Peers:** 2 peers under the same org
  - `peer0.trst01.example.com` on **VM1** (`40.81.236.78`)
  - `peer1.trst01.example.com` on **VM2** (`20.244.16.67`)
- **Orderers:** 3-node **RAFT** ordering service
  - `orderer1.example.com` on **VM1** (client port `7050`, admin `7053`)
  - `orderer2.example.com` on **VM2** (client port `8050`, admin `8053`)
  - `orderer3.example.com` on **VM2** (client port `9050`, admin `9053`)
- **Channel:** `oatschannel`

The channel is bootstrapped using `configtxgen` to generate a `channel-artifacts/oatschannel.block`, and each peer joins the channel using `peer channel join`.

## Hostnames and IPs (must resolve on both VMs)

Ensure these hostnames resolve to the correct VM IPs on **both** VMs (e.g., `/etc/hosts`):

```bash
# VM1
40.81.236.78 orderer1.example.com
40.81.236.78 peer0.trst01.example.com

# VM2
20.244.16.67 orderer2.example.com
20.244.16.67 orderer3.example.com
20.244.16.67 peer1.trst01.example.com
```

## Ports used (per service)

Peers:
- `peer0.trst01.example.com`:
  - peer: `7051`
  - operations/metrics: `9444`
- `peer1.trst01.example.com`:
  - peer: `9051`
  - operations/metrics: `9445`

Orderers (RAFT):
- `orderer1.example.com`:
  - client: `7050`
  - admin: `7053`
  - operations/metrics: `9443`
- `orderer2.example.com`:
  - client: `8050`
  - admin: `8053`
  - operations/metrics: `9446` (remapped host port when using both stacks locally)
- `orderer3.example.com`:
  - client: `9050`
  - admin: `9053`
  - operations/metrics: `9447` (remapped host port when using both stacks locally)

Notes:
- The **join and transaction path uses the client ports** (`7050/8050/9050`) and TLS host overrides.
- The operations/metrics remapping (`9446/9447`) is only to avoid port conflicts when running both `oats-network1/` and `oats-network2/` on the same Docker host locally.

## Folder layout

All work happens under:

`~/fabric-dev/fabric-samples/oats-cluster/`

Two network directories:
- `oats-network1/` : VM1 services only (`peer0` + `orderer1`)
- `oats-network2/` : VM2 services only (`peer1` + `orderer2` + `orderer3`)

## Assumptions / prerequisites

On **each VM**, you have:
- Docker + `docker-compose`
- Fabric binaries in `~/fabric-dev/fabric-samples/oats-cluster/../bin` (the scripts in this sample expect `../bin` to contain `peer`, `cryptogen`, `configtxgen`, `osnadmin`, etc.)
- The crypto folders exist after running `cryptogen` on VM1 (and after copying to VM2)

## Environment setup (do once per terminal)

Run this in the active terminal on each VM (from inside `oats-network1/` or `oats-network2/`):

```bash
# Add both system paths and the Fabric binaries for this sample.
# (Fabric binaries live at oats-cluster/bin, i.e. $PWD/../../bin when inside oats-network1/ or oats-network2/)
export PATH=/usr/bin:/bin:${PWD}/../../bin:${PWD}

# Avoid docker-compose warnings and ensure scripts/compose can reach the docker socket.
export DOCKER_SOCK=/var/run/docker.sock
```

## Step-by-step (copy/paste runbook)

### 0) Create the folders (one-time, already done for this repo)

From `fabric-samples/`:

```bash
mkdir oats-cluster
cd oats-cluster
cp -r ../test-network oats-network1
cp -r ../test-network oats-network2
```

### 1) Generate crypto material (run ONCE on VM1)

Run from the VM1 directory:

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
```

If you have old state:
```bash
./network.sh down
```

Generate crypto (single org TRST01 + orderers orderer1/2/3):
```bash
cryptogen generate --config=./organizations/cryptogen/crypto-config.yaml --output="organizations"
```

Generate the **channel** configuration block (`oatschannel.block`):
```bash
rm -rf ./channel-artifacts
mkdir -p ./channel-artifacts

export CHANNEL_ID=oatschannel
export FABRIC_CFG_PATH=$PWD/configtx
export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem

configtxgen -profile ChannelUsingRaft \
  -outputBlock ./channel-artifacts/${CHANNEL_ID}.block \
  -channelID ${CHANNEL_ID}
```

### 2) Distribute crypto + channel artifacts to VM2

On VM1:

```bash
scp -r ./organizations/ user@20.244.16.67:~/fabric-dev/fabric-samples/oats-cluster/oats-network2/
scp -r ./channel-artifacts/ user@20.244.16.67:~/fabric-dev/fabric-samples/oats-cluster/oats-network2/
```

If you are simulating locally (single machine), you can copy directly:

```bash
rm -rf ../oats-network2/channel-artifacts
cp -r ./channel-artifacts/ ../oats-network2/
cp -r ./organizations/ ../oats-network2/
```

### 3) Start orderers first

On VM1 (start only `orderer1.example.com`):
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1

docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d orderer1.example.com
```

On VM2 (start `orderer2.example.com` + `orderer3.example.com`):
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2

docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d orderer2.example.com orderer3.example.com
```

### 4) Start peers + join them to the channel

#### VM1: peer0 joins
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1

export FABRIC_CFG_PATH=$PWD/compose/docker/peercfg
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Trst01MSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/trst01.example.com/users/Admin@trst01.example.com/msp
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/trst01.example.com/tlsca/tlsca.trst01.example.com-cert.pem
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_ID=peer0.trst01.example.com

docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d peer0.trst01.example.com

# Must have ORDERER_CA set:
export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem

peer channel join -b ./channel-artifacts/oatschannel.block \
  -o localhost:7050 \
  --ordererTLSHostnameOverride orderer1.example.com \
  --tls --cafile $ORDERER_CA
```

#### VM2: peer1 joins
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2

export FABRIC_CFG_PATH=$PWD/compose/docker/peercfg
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Trst01MSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/trst01.example.com/users/Admin@trst01.example.com/msp
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/trst01.example.com/tlsca/tlsca.trst01.example.com-cert.pem
export CORE_PEER_ADDRESS=localhost:9051
export CORE_PEER_ID=peer1.trst01.example.com

docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d peer1.trst01.example.com

export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem

peer channel join -b ./channel-artifacts/oatschannel.block \
  -o localhost:8050 \
  --ordererTLSHostnameOverride orderer2.example.com \
  --tls --cafile $ORDERER_CA
```

## Verification (quick checks)

After peers join, you should see containers running:
```bash
docker ps
```

You should see:
- VM1: `peer0.trst01.example.com`, `orderer1.example.com`
- VM2: `peer1.trst01.example.com`, `orderer2.example.com`, `orderer3.example.com`

## Fault tolerance test idea (as per your intended goal)

After channel is live and both peers are joined:

1. Stop `peer0` (VM1) and keep `orderer2 + orderer3` (VM2) running:
```bash
docker stop peer0.trst01.example.com
```

2. Confirm peer1 remains able to run chaincode transactions (once you deploy chaincode).

## Notes / gotchas

1. Hostname resolution matters:
   - `orderer1.example.com`, `orderer2.example.com`, `orderer3.example.com`,
     `peer0.trst01.example.com`, `peer1.trst01.example.com`
   must resolve correctly from the perspective of the VM where commands run.
2. Local simulation port conflicts:
   - If you start both `oats-network1/` and `oats-network2/` on one machine, ensure operations/metrics ports don’t collide.
   - This repo’s `oats-network2/compose/compose-test-net.yaml` remaps orderer2/3 metrics ports to `9446/9447` for local runs.

## Stop / Start Tomorrow (preserve ledger data)

The ledger data lives in **named Docker volumes** mounted at:
- peers: `/var/hyperledger/production`
- orderers: `/var/hyperledger/production`

To preserve ledger/state, use `stop` (or `docker stop`) and **do not** run `docker-compose down --volumes` unless you explicitly want to delete ledger data.

### Stop today

VM1 (`oats-network1/`):
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
export DOCKER_SOCK=/var/run/docker.sock
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml stop \
  orderer1.example.com peer0.trst01.example.com
```

VM2 (`oats-network2/`):
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2
export DOCKER_SOCK=/var/run/docker.sock
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml stop \
  orderer2.example.com orderer3.example.com peer1.trst01.example.com
```

### Start tomorrow

VM1:
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
export DOCKER_SOCK=/var/run/docker.sock
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d \
  orderer1.example.com peer0.trst01.example.com
```

VM2:
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2
export DOCKER_SOCK=/var/run/docker.sock
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d \
  orderer2.example.com orderer3.example.com peer1.trst01.example.com
```

