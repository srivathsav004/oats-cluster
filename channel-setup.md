# TRST01 (TRST01 org) + 3-Node RAFT (oatschannel) - OATS Cluster

This is a customized Fabric `test-network` based topology that runs:

- **Org:** `TRST01` (`Trst01MSP`)
- **Peers:** `peer0` on **VM1** and `peer1` on **VM2**
  - `peer0.trst01.example.com` on VM1 (`40.81.236.78`)
  - `peer1.trst01.example.com` on VM2 (`20.244.16.67`)
- **Orderers (RAFT):** `orderer1` on VM1 and `orderer2 + orderer3` on VM2
  - `orderer1.example.com` on VM1 (client: `7050`, admin: `7053`)
  - `orderer2.example.com` on VM2 (client: `8050`, admin: `8053`)
  - `orderer3.example.com` on VM2 (client: `9050`, admin: `9053`)
- **Channel:** `oatschannel`

Ledger goal: **3-node RAFT survives 1 failure** (stop `peer0`/VM1 later; orderers/peer1 continue).

---

## Architecture Overview

```
TRST01 Organization
├─ peer0.trst01.example.com  (VM1)  [peer: 7051]
└─ peer1.trst01.example.com  (VM2)  [peer: 9051]

RAFT Orderers (3 nodes)
├─ orderer1.example.com      (VM1)  [client: 7050, admin: 7053]
├─ orderer2.example.com      (VM2)  [client: 8050, admin: 8053]
└─ orderer3.example.com      (VM2)  [client: 9050, admin: 9053]
```

---

## All commands assume you are here

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/
```

You will use two customized network folders:

- `oats-network1/` : **VM1 services only** (`peer0` + `orderer1`)
- `oats-network2/` : **VM2 services only** (`peer1` + `orderer2` + `orderer3`)

---

## Phase 0 — VM Deployment (Copy folders to remote VMs)

### Prerequisites
- FileZilla or SCP access to both VMs
- Both VMs have Docker and Docker Compose installed
- Fabric binaries available on both VMs

### Step 1: Copy folders to VMs

From your local machine, copy the prepared network folders:

**Using FileZilla:**
- Copy `oats-network1/` to VM1 (40.81.236.78) at `~/fabric-dev/fabric-samples/oats-cluster/`
- Copy `oats-network2/` to VM2 (20.244.16.67) at `~/fabric-dev/fabric-samples/oats-cluster/`

**Using SCP:**
```bash
# Copy to VM1
scp -r oats-network1 user@40.81.236.78:~/fabric-dev/fabric-samples/oats-cluster/

# Copy to VM2  
scp -r oats-network2 user@20.244.16.67:~/fabric-dev/fabric-samples/oats-cluster/

# Copy shared binaries (if needed)
scp -r bin user@40.81.236.78:~/fabric-dev/fabric-samples/oats-cluster/
scp -r bin user@20.244.16.67:~/fabric-dev/fabric-samples/oats-cluster/
```

### Step 2: VM-specific setup commands

**On VM1 (40.81.236.78):**
```bash
# Create directory structure
mkdir -p ~/fabric-dev/fabric-samples/oats-cluster
cd ~/fabric-dev/fabric-samples/oats-cluster

# Verify folders exist
ls -la

# Set up hostname resolution (if not already configured)
sudo bash -c 'cat >> /etc/hosts << EOF
40.81.236.78 orderer1.example.com
40.81.236.78 peer0.trst01.example.com
20.244.16.67 orderer2.example.com
20.244.16.67 orderer3.example.com
20.244.16.67 peer1.trst01.example.com
EOF'
```

**On VM2 (20.244.16.67):**
```bash
# Create directory structure
mkdir -p ~/fabric-dev/fabric-samples/oats-cluster
cd ~/fabric-dev/fabric-samples/oats-cluster

# Verify folders exist
ls -la

# Set up hostname resolution (if not already configured)
sudo bash -c 'cat >> /etc/hosts << EOF
40.81.236.78 orderer1.example.com
40.81.236.78 peer0.trst01.example.com
20.244.16.67 orderer2.example.com
20.244.16.67 orderer3.example.com
20.244.16.67 peer1.trst01.example.com
EOF'
```

---

## Phase 0 — Local Development Setup (Alternative)

If running locally on a single machine for development:

### Hostname/IP mapping (must exist on both VMs)

Ensure these hostnames resolve correctly (example via `/etc/hosts`):

```bash
# Local development (single machine)
127.0.0.1 orderer1.example.com
127.0.0.1 orderer2.example.com
127.0.0.1 orderer3.example.com
127.0.0.1 peer0.trst01.example.com
127.0.0.1 peer1.trst01.example.com

# VM1 (40.81.236.78) - when deployed on actual VMs
40.81.236.78 orderer1.example.com
40.81.236.78 peer0.trst01.example.com

# VM2 (20.244.16.67) - when deployed on actual VMs
20.244.16.67 orderer2.example.com
20.244.16.67 orderer3.example.com
20.244.16.67 peer1.trst01.example.com
```

---

## Phase 1 — Environment setup (PATH + Docker socket)

### For VM Deployment (run on each VM)

**On VM1 (40.81.236.78):**
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1

# Add both system paths and the Fabric binary path
export PATH=/usr/bin:/bin:${PWD}/../../bin:${PWD}

# Set Docker socket
export DOCKER_SOCK=/var/run/docker.sock

# Verify Docker is running
sudo systemctl status docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
```

**On VM2 (20.244.16.67):**
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2

# Add both system paths and the Fabric binary path
export PATH=/usr/bin:/bin:${PWD}/../../bin:${PWD}

# Set Docker socket
export DOCKER_SOCK=/var/run/docker.sock

# Verify Docker is running
sudo systemctl status docker
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
```

### For Local Development (single machine)

Run this in your terminal on each VM (from inside either `oats-network1/` or `oats-network2/`):

```bash
# Add both system paths and the Fabric binary path
export PATH=/usr/bin:/bin:${PWD}/../../bin:${PWD}

# Set Docker socket
export DOCKER_SOCK=/var/run/docker.sock
```

Notes:
- `PATH` includes `${PWD}/../../bin` because the Fabric binaries live at `oats-cluster/bin` relative to `oats-network*`.
- Setting `DOCKER_SOCK` avoids compose warnings and ensures the peercfg mounts work.

---

## Phase 2 — (If needed) Create the network folders

One-time setup from `fabric-samples/`:

```bash
cd ~/fabric-dev/fabric-samples/
mkdir oats-cluster
cd oats-cluster
cp -r ../test-network oats-network1
cp -r ../test-network oats-network2
```

---

## Phase 3 — Generate crypto (ONLY ONCE, run on VM1)

1) Go to VM1 network folder:

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
```

2) If you have old state (optional):

```bash
./network.sh down
```

3) Generate crypto (single org `TRST01`, 2 peers + 3 orderers):

```bash
cryptogen generate --config=./organizations/cryptogen/crypto-config.yaml --output="organizations"
```

---

## Phase 4 — Generate the channel block (run on VM1)

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1

rm -rf ./channel-artifacts
mkdir -p ./channel-artifacts

export CHANNEL_ID=oatschannel
export FABRIC_CFG_PATH=$PWD/configtx
export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem

configtxgen -profile ChannelUsingRaft \
  -outputBlock ./channel-artifacts/${CHANNEL_ID}.block \
  -channelID ${CHANNEL_ID}
```

---

## Phase 5 — Distribute artifacts to VM2

On VM1:

```bash
scp -r ./organizations/ user@20.244.16.67:~/fabric-dev/fabric-samples/oats-cluster/oats-network2/
scp -r ./channel-artifacts/ user@20.244.16.67:~/fabric-dev/fabric-samples/oats-cluster/oats-network2/
```

---

## Phase 6 — Start orderers first (start tomorrow = reuse this step)

### VM1: start `orderer1.example.com`
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d orderer1.example.com
```

### VM2: start `orderer2.example.com` and `orderer3.example.com`
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d orderer2.example.com orderer3.example.com
```

---

## Phase 7 — Join orderers to the channel (VM1 generates, joins are per-VM)

Set common var on each VM:

```bash
export CHANNEL_ID=oatschannel
export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem
```

### VM1: join orderer1
```bash
osnadmin channel join \
  --channelID $CHANNEL_ID \
  --config-block ./channel-artifacts/${CHANNEL_ID}.block \
  -o localhost:7053 \
  --ca-file "$ORDERER_CA" \
  --client-cert "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.crt" \
  --client-key  "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.key"
```

### VM2: join orderer2 and orderer3
```bash
osnadmin channel join \
  --channelID $CHANNEL_ID \
  --config-block ./channel-artifacts/${CHANNEL_ID}.block \
  -o localhost:8053 \
  --ca-file "$ORDERER_CA" \
  --client-cert "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt" \
  --client-key  "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.key"

osnadmin channel join \
  --channelID $CHANNEL_ID \
  --config-block ./channel-artifacts/${CHANNEL_ID}.block \
  -o localhost:9053 \
  --ca-file "$ORDERER_CA" \
  --client-cert "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt" \
  --client-key  "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.key"
```

---

## Phase 8 — Start peers + join peers

### VM1 peer0: start + join
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d peer0.trst01.example.com

export FABRIC_CFG_PATH=$PWD/compose/docker/peercfg
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Trst01MSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/trst01.example.com/users/Admin@trst01.example.com/msp
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/trst01.example.com/tlsca/tlsca.trst01.example.com-cert.pem
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_ID=peer0.trst01.example.com

export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem

peer channel join -b ./channel-artifacts/oatschannel.block \
  -o localhost:7050 \
  --ordererTLSHostnameOverride orderer1.example.com \
  --tls --cafile $ORDERER_CA
```

### VM2 peer1: start + join
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d peer1.trst01.example.com

export FABRIC_CFG_PATH=$PWD/compose/docker/peercfg
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Trst01MSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/trst01.example.com/users/Admin@trst01.example.com/msp
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/trst01.example.com/tlsca/tlsca.trst01.example.com-cert.pem
export CORE_PEER_ADDRESS=localhost:9051
export CORE_PEER_ID=peer1.trst01.example.com

export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem

peer channel join -b ./channel-artifacts/oatschannel.block \
  -o localhost:8050 \
  --ordererTLSHostnameOverride orderer2.example.com \
  --tls --cafile $ORDERER_CA
```

---

## Phase 9 — Verification

Check containers:

```bash
docker ps
```

You should see:
- VM1: `orderer1.example.com`, `peer0.trst01.example.com`
- VM2: `orderer2.example.com`, `orderer3.example.com`, `peer1.trst01.example.com`

---

## Phase 10 — Stop today / Start tomorrow (preserve ledger data)

Ledger and membership are stored in Docker named volumes mounted at `/var/hyperledger/production`.
To preserve them, use `stop` (NOT `down --volumes`).

### VM1 (stop)
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml stop orderer1.example.com peer0.trst01.example.com
```

### VM2 (stop)
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml stop orderer2.example.com orderer3.example.com peer1.trst01.example.com
```

### VM1 (start tomorrow)
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d orderer1.example.com peer0.trst01.example.com
```

### VM2 (start tomorrow)
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d orderer2.example.com orderer3.example.com peer1.trst01.example.com
```

Important:
- You should **not** need to re-run `cryptogen`, `configtxgen`, `osnadmin channel join`, or `peer channel join` when using `stop`/`up` (ledger data and channel membership remain).

---

## (Optional) What if you want a full reset?

Only do this if you want to wipe ledgers and start from scratch:

```bash
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml down --volumes --remove-orphans
```

Then re-run from Phase 3.

