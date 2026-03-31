# OATS Traceability Chaincode - TRST01 (Single Org) + 3-Node RAFT

This document describes how to deploy and test your **Node.js** chaincode (`oats-traceability-javascript`) on the already-running network:

- **Channel:** `oatschannel`
- **Org / MSP:** `Trst01MSP`
- **Peers:** `peer0.trst01.example.com` (VM1) and `peer1.trst01.example.com` (VM2)
- **Orderers (RAFT):** `orderer1` (VM1), `orderer2` + `orderer3` (VM2)

All commands assume you are in either:
- `oats-network1/` on **VM1**
- `oats-network2/` on **VM2**

---

## Architecture assumptions (what to run where)

### VM1 (`oats-network1/`)
- You run: **package + approve + commit** (from `peer0`)
- You may also run: installs (from `peer0`)

### VM2 (`oats-network2/`)
- You run: install only (from `peer1`)

---

## 0) Required chaincode folder

Your chaincode must exist at:

`$PWD/chaincode/oats-traceability-javascript`

With at least:
- `index.js`
- `lib/oatsTraceability.js`
- `package.json`

---

## 1) Phase A — Environment setup (run on each VM terminal)

Run this on the VM where you are executing commands (VM1 or VM2), inside `oats-network1/` or `oats-network2/`.

```bash
# Add both system paths and the Fabric binary path
# (when inside oats-network*, Fabric binaries are at oats-cluster/bin)
export PATH=/usr/bin:/bin:${PWD}/../../bin:${PWD}

# Docker socket for docker-compose / compose peercfg mounts
export DOCKER_SOCK=/var/run/docker.sock

# CLI config for peer commands
export FABRIC_CFG_PATH=$PWD/compose/docker/peercfg
unset CORE_PEER_TLS_ENABLED # optional cleanup if you have it set
```

Common variables:

```bash
export CHANNEL_NAME=oatschannel
export CC_NAME=oats-traceability
export CC_VERSION=1.0
export CC_SEQUENCE=1
```

Common orderer TLS CA (client TLS CA):

```bash
export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem
```

---

## 2) Phase B — Package chaincode (VM1 only)

### 2.1 Define chaincode source path

On **VM1**:

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
export CC_SRC_PATH=$PWD/chaincode/oats-traceability-javascript
```

### 2.2 (Optional but recommended) install Node dependencies

```bash
cd $CC_SRC_PATH
npm install --no-audit --no-fund
cd - >/dev/null
```

### 2.3 Package

```bash
peer lifecycle chaincode package ${CC_NAME}.tar.gz \
  --path "$CC_SRC_PATH" \
  --lang node \
  --label ${CC_NAME}_${CC_VERSION}

export PACKAGE_ID=$(peer lifecycle chaincode calculatepackageid ${CC_NAME}.tar.gz)
echo "PACKAGE_ID=${PACKAGE_ID}"
```

---

## 3) Phase C — Install chaincode (peer0 on VM1, peer1 on VM2)

### 3.1 Install on VM1 peer0

On **VM1**:

```bash
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Trst01MSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/trst01.example.com/users/Admin@trst01.example.com/msp
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/trst01.example.com/tlsca/tlsca.trst01.example.com-cert.pem
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_ID=peer0.trst01.example.com

peer lifecycle chaincode install ${CC_NAME}.tar.gz
```

### 3.2 Copy package to VM2 + install on peer1

From **VM1**:

```bash
scp ${CC_NAME}.tar.gz user@20.244.16.67:~/fabric-dev/fabric-samples/oats-cluster/oats-network2/
```

On **VM2**:

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2

export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Trst01MSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/trst01.example.com/users/Admin@trst01.example.com/msp
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/trst01.example.com/tlsca/tlsca.trst01.example.com-cert.pem
export CORE_PEER_ADDRESS=localhost:9051
export CORE_PEER_ID=peer1.trst01.example.com

peer lifecycle chaincode install ${CC_NAME}.tar.gz
```

---

## 4) Phase D — Approve (VM1 peer0 only)

On **VM1** (peer0 context):

```bash
export ORDERER_ADDRESS=localhost:7050
export ORDERER_TLS_HOST=orderer1.example.com

peer lifecycle chaincode approveformyorg \
  -o ${ORDERER_ADDRESS} --ordererTLSHostnameOverride ${ORDERER_TLS_HOST} \
  --tls --cafile "${ORDERER_CA}" \
  --channelID ${CHANNEL_NAME} \
  --name ${CC_NAME} \
  --version ${CC_VERSION} \
  --package-id ${PACKAGE_ID} \
  --sequence ${CC_SEQUENCE} \
  --signature-policy "OR('Trst01MSP.peer')"
```

---

## 5) Phase E — Commit (VM1 peer0 only)

On **VM1**:

```bash
export CORE_PEER_ADDRESS=localhost:7051
export PEER0_TLS_CA=$PWD/organizations/peerOrganizations/trst01.example.com/peers/peer0.trst01.example.com/tls/ca.crt

peer lifecycle chaincode commit \
  -o ${ORDERER_ADDRESS} --ordererTLSHostnameOverride ${ORDERER_TLS_HOST} \
  --tls --cafile "${ORDERER_CA}" \
  --channelID ${CHANNEL_NAME} \
  --name ${CC_NAME} \
  --version ${CC_VERSION} \
  --sequence ${CC_SEQUENCE} \
  --peerAddresses localhost:7051 \
  --tlsRootCertFiles "${PEER0_TLS_CA}" \
  --signature-policy "OR('Trst01MSP.peer')"
```

Verify committed state:

```bash
peer lifecycle chaincode querycommitted --channelID ${CHANNEL_NAME} --name ${CC_NAME}
```

---

## 6) Phase F — Invoke + Query (examples)

Pick a peer context (recommend VM1 / peer0):

### 6.1 Invoke: `CreateActor`

On **VM1** (peer0 env):

```bash
export ORDERER_ADDRESS=localhost:7050
export ORDERER_TLS_HOST=orderer1.example.com

peer chaincode invoke \
  -o ${ORDERER_ADDRESS} --ordererTLSHostnameOverride ${ORDERER_TLS_HOST} \
  --tls --cafile "${ORDERER_CA}" \
  -C ${CHANNEL_NAME} -n ${CC_NAME} \
  --peerAddresses localhost:7051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/trst01.example.com/peers/peer0.trst01.example.com/tls/ca.crt" \
  -c '{"Args":["CreateActor","ACT-FARMER-001","Farmer","Green Valley Farms","FARM-REG-001","{\"phone\":\"+91-9876543210\",\"email\":\"farmer@greenvalley.com\"}","[\"CERT-ORG-001\",\"CERT-FAIR-001\"]"]}'
```

### 6.2 Query: `GetActor`

```bash
peer chaincode query -C ${CHANNEL_NAME} -n ${CC_NAME} \
  -c '{"Args":["GetActor","ACT-FARMER-001"]}'
```

---

## 7) Stop today / Start tomorrow (preserving chaincode + ledgers)

The chaincode definition and all ledger data are stored in Docker **volumes** mounted into peer/orderer containers. To keep everything and just pause/restart:

### 7.1 Stop network (VM1 and VM2)

**VM1:**

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml stop \
  orderer1.example.com peer0.trst01.example.com
```

**VM2:**

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml stop \
  orderer2.example.com orderer3.example.com peer1.trst01.example.com
```

### 7.2 Start network again (tomorrow)

**VM1:**

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d \
  orderer1.example.com peer0.trst01.example.com
```

**VM2:**

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d \
  orderer2.example.com orderer3.example.com peer1.trst01.example.com
```

In this flow:
- You **do not** need to re-run `cryptogen`, `configtxgen`, `osnadmin channel join`, `peer channel join`, or the chaincode lifecycle commands.
- All chaincode definitions and ledgers persist.

### 7.3 Full reset (only if you want to wipe everything)

If you intentionally want a clean slate (delete ledgers + chaincode state):

```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml down --volumes --remove-orphans

cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml down --volumes --remove-orphans
```

Then re-run:
- Network bring-up (see `README_TRST01_RAFT_OATSCHANNEL.md`)
- This chaincode lifecycle guide from Phase A.

---

## 8) Notes (important)

1. **Endorsement policy:** this is **single org TRST01**, so we approve/commit once from VM1 (peer0). Both peers must still have the chaincode **installed** to endorse/query.
2. **Preserving data:** only `down --volumes` wipes ledgers/definitions; `stop`/`up` preserves everything.
3. **Channel membership:** after peer joins, `oatschannel.block` should live in both `oats-network1/` and `oats-network2/` to ease re-joins or new peers.

