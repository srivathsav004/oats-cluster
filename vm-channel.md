# VM Deployment Guide - TRST01 RAFT Network

This guide provides complete commands for setting up the TRST01 RAFT network across multiple VMs.

## VM Architecture

- **VM1**: `peer0.trst01.example.com` + `orderer1.example.com`
- **VM2**: `peer1.trst01.example.com` + `orderer2.example.com` + `orderer3.example.com`
- **Channel**: `oatschannel`
- **Organization**: `TRST01` (`Trst01MSP`)

## Directory Structure on Each VM

```
ubuntu@vm-hyperledgernode1:~/fabric-samples/oats-cluster/oats-network1$
ubuntu@vm-hyperledgernode2:~/fabric-samples/oats-cluster/oats-network2$
```

---

## Phase 0 - Hostname Setup (All VMs)

Add to `/etc/hosts` on each VM:

### VM1:
```bash
# VM1 IP addresses (replace with actual IPs)
<VM1_IP> orderer1.example.com
<VM1_IP> peer0.trst01.example.com
<VM2_IP> orderer2.example.com
<VM2_IP> orderer3.example.com
<VM2_IP> peer1.trst01.example.com
```

### VM2:
```bash
# VM2 IP addresses (replace with actual IPs)
<VM1_IP> orderer1.example.com
<VM1_IP> peer0.trst01.example.com
<VM2_IP> orderer2.example.com
<VM2_IP> orderer3.example.com
<VM2_IP> peer1.trst01.example.com
```

---

## Phase 1 - Environment Setup (All VMs)

Run on each VM in the respective network directory:

```bash
# Set environment variables
export PATH=/usr/bin:/bin:${PWD}/../../bin:${PWD}
export DOCKER_SOCK=/var/run/docker.sock
export FABRIC_CFG_PATH=$PWD/configtx
```

---

## Phase 2 - Generate Crypto Material (VM1 Only)

```bash
cd ~/fabric-samples/oats-cluster/oats-network1

# Clean old state if exists
./network.sh down

# Generate crypto material
cryptogen generate --config=./organizations/cryptogen/crypto-config.yaml --output="organizations"
```

---

## Phase 3 - Generate Channel Block (VM1 Only)

```bash
cd ~/fabric-samples/oats-cluster/oats-network1

# Remove old artifacts and create new directory
rm -rf ./channel-artifacts
mkdir -p ./channel-artifacts

# Set environment variables
export CHANNEL_ID=oatschannel
export FABRIC_CFG_PATH=$PWD/configtx
export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem

# Generate channel block
configtxgen -profile ChannelUsingRaft \
  -outputBlock ./channel-artifacts/${CHANNEL_ID}.block \
  -channelID ${CHANNEL_ID}
```

---

## Phase 4 - Distribute Artifacts to VM2 (VM1 → VM2)

```bash
cd ~/fabric-samples/oats-cluster/oats-network1

# Copy organizations and channel artifacts to VM2
scp -r ./organizations/ ubuntu@<VM2_IP>:~/fabric-samples/oats-cluster/oats-network2/
scp -r ./channel-artifacts/ ubuntu@<VM2_IP>:~/fabric-samples/oats-cluster/oats-network2/
```

---

## Phase 5 - Start Orderers (All VMs)

### VM1 - Start orderer1:
```bash
cd ~/fabric-samples/oats-cluster/oats-network1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d orderer1.example.com
```

### VM2 - Start orderer2 and orderer3:
```bash
cd ~/fabric-samples/oats-cluster/oats-network2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d orderer2.example.com orderer3.example.com
```

---

## Phase 6 - Join Orderers to Channel (All VMs)

Set common variables on each VM:
```bash
export CHANNEL_ID=oatschannel
export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem
```

### VM1 - Join orderer1:
```bash
osnadmin channel join \
  --channelID $CHANNEL_ID \
  --config-block ./channel-artifacts/${CHANNEL_ID}.block \
  -o localhost:7053 \
  --ca-file "$ORDERER_CA" \
  --client-cert "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.crt" \
  --client-key  "$PWD/organizations/ordererOrganizations/example.com/orderers/orderer1.example.com/tls/server.key"
```

### VM2 - Join orderer2 and orderer3:
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

## Phase 7 - Start Peers and Join Channel (All VMs)

### VM1 - Start and join peer0:
```bash
cd ~/fabric-samples/oats-cluster/oats-network1

# Start peer0
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d peer0.trst01.example.com

# Set peer environment variables
export FABRIC_CFG_PATH=$PWD/compose/docker/peercfg
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Trst01MSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/trst01.example.com/users/Admin@trst01.example.com/msp
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/trst01.example.com/tlsca/tlsca.trst01.example.com-cert.pem
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_ID=peer0.trst01.example.com
export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem

# Join channel
peer channel join -b ./channel-artifacts/oatschannel.block \
  -o localhost:7050 \
  --ordererTLSHostnameOverride orderer1.example.com \
  --tls --cafile $ORDERER_CA
```

### VM2 - Start and join peer1:
```bash
cd ~/fabric-samples/oats-cluster/oats-network2

# Start peer1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d peer1.trst01.example.com

# Set peer environment variables
export FABRIC_CFG_PATH=$PWD/compose/docker/peercfg
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID=Trst01MSP
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/trst01.example.com/users/Admin@trst01.example.com/msp
export CORE_PEER_TLS_ROOTCERT_FILE=$PWD/organizations/peerOrganizations/trst01.example.com/tlsca/tlsca.trst01.example.com-cert.pem
export CORE_PEER_ADDRESS=localhost:9051
export CORE_PEER_ID=peer1.trst01.example.com
export ORDERER_CA=$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem

# Join channel
peer channel join -b ./channel-artifacts/oatschannel.block \
  -o localhost:8050 \
  --ordererTLSHostnameOverride orderer2.example.com \
  --tls --cafile $ORDERER_CA
```

---

## Phase 8 - Verification

Check containers on each VM:

```bash
docker ps
```

Expected results:
- **VM1**: `orderer1.example.com`, `peer0.trst01.example.com`
- **VM2**: `orderer2.example.com`, `orderer3.example.com`, `peer1.trst01.example.com`

---

## Phase 9 - Network Management

### Stop Network (Preserve Data):
```bash
# VM1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml stop orderer1.example.com peer0.trst01.example.com

# VM2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml stop orderer2.example.com orderer3.example.com peer1.trst01.example.com
```

### Start Network (Resume):
```bash
# VM1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d orderer1.example.com peer0.trst01.example.com

# VM2
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d orderer2.example.com orderer3.example.com peer1.trst01.example.com
```

### Full Reset (Wipe All Data):
```bash
# Both VMs
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml down --volumes --remove-orphans
```

---

## Important Notes

1. **IP Addresses**: Replace `<VM1_IP>` and `<VM2_IP>` with actual VM IP addresses
2. **Path Structure**: Commands assume `~/fabric-samples/oats-cluster/oats-network1/` and `~/fabric-samples/oats-cluster/oats-network2/`
3. **Data Persistence**: Use `stop`/`up` to preserve ledger data, `down --volumes` only for complete reset
4. **Channel Artifacts**: Generated once on VM1 and distributed to VM2
5. **Crypto Material**: Generated once on VM1 and distributed to VM2

After completing these phases, your network will be ready for chaincode deployment.
