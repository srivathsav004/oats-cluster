# OATS Backend API Setup & Documentation

This document describes the backend API setup for the OATS (Organic Agricultural Traceability System) running on a multi-VM Hyperledger Fabric network with TRST01 organization and 3-node RAFT consensus.

---

## Architecture Overview

```
TRST01 Organization (Multi-VM Setup)
├─ VM1 (40.81.236.78)
│  ├─ peer0.trst01.example.com (port 7051)
│  ├─ orderer1.example.com (client: 7050, admin: 7053)
│  └─ Backend API (port 3000)
└─ VM2 (20.244.16.67)
   ├─ peer1.trst01.example.com (port 9051)
   ├─ orderer2.example.com (client: 8050, admin: 8053)
   ├─ orderer3.example.com (client: 9050, admin: 9053)
   └─ Backend API (port 3001)
```

**Network Configuration:**
- **Channel:** `oatschannel`
- **Chaincode:** `oats-traceability`
- **Organization MSP:** `Trst01MSP`
- **Consensus:** 3-node RAFT (survives 1 failure)

---

## Backend Setup

### Prerequisites

1. Hyperledger Fabric network running (see `README_TRST01_RAFT_OATSCHANNEL.md`)
2. Chaincode deployed and committed to channel (see `chaincode-reference.md`)
3. Node.js and npm installed
4. Fabric binaries available at `oats-cluster/bin`

### Environment Configuration

Each backend instance requires a specific `.env` configuration:

#### VM1 Backend (`oats-network1/backend/.env`)

```bash
# Fabric Binary Path
FABRIC_BIN_PATH=../../bin

# Network Configuration
CHANNEL_NAME=oatschannel
CC_NAME=oats-traceability
ORDERER_ADDRESS=localhost:7050
ORDERER_TLS_HOST=orderer1.example.com
PORT=3000

# Organization Configuration (TRST01)
CORE_PEER_LOCALMSPID=Trst01MSP
CORE_PEER_ADDRESS=localhost:7051
```

#### VM2 Backend (`oats-network2/backend/.env`)

```bash
# Fabric Binary Path
FABRIC_BIN_PATH=../../bin

# Network Configuration
CHANNEL_NAME=oatschannel
CC_NAME=oats-traceability
ORDERER_ADDRESS=localhost:8050
ORDERER_TLS_HOST=orderer2.example.com
PORT=3001

# Organization Configuration (TRST01)
CORE_PEER_LOCALMSPID=Trst01MSP
CORE_PEER_ADDRESS=localhost:9051
```

### Installation & Startup

1. **Install dependencies** (run on each VM):
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1/backend  # or oats-network2/backend
npm install
```

2. **Start the backend servers**:

**VM1:**
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network1/backend
npm start
```

**VM2:**
```bash
cd ~/fabric-dev/fabric-samples/oats-cluster/oats-network2/backend
npm start
```

The APIs will be available at:
- VM1: `http://localhost:3000`
- VM2: `http://localhost:3001`

---

## API Endpoints

### Base URLs
- **VM1 API:** `http://localhost:3000`
- **VM2 API:** `http://localhost:3001`

### Common Headers
```bash
Content-Type: application/json
```

---

## Error Handling

### Common Error Responses

**400 Bad Request:**
```json
{
  "error": "Invalid request data",
  "message": "Missing required field: actorId"
}
```

**500 Internal Server Error:**
```json
{
  "error": "Failed to create actor",
  "message": "Error: error endorsing invoke: rpc error: code = Unknown desc = ..."
}
```

**404 Not Found:**
```json
{
  "error": "Actor not found",
  "message": "Actor with ID ACTOR999 does not exist"
}
```

---

## Testing the APIs

### Test Actor Creation

**VM1 API:**
```bash
curl -X POST http://localhost:3000/actors/create \
  -H "Content-Type: application/json" \
  -d '{
    "actorId": "ACTOR_VM1_001",
    "actorType": "FARMER",
    "name": "VM1 Test Farm",
    "registryReference": "REG_VM1_001",
    "contactDetails": {
      "email": "vm1@farm.com",
      "phone": "+1234567890",
      "address": "123 VM1 Farm Road"
    },
    "certificationIds": ["CERT_VM1_001"]
  }'
```

**VM2 API:**
```bash
curl -X POST http://localhost:3001/actors/create \
  -H "Content-Type: application/json" \
  -d '{
    "actorId": "ACTOR_VM2_001",
    "actorType": "PROCESSOR",
    "name": "VM2 Processing Plant",
    "registryReference": "REG_VM2_001",
    "contactDetails": {
      "email": "vm2@processor.com",
      "phone": "+1234567891",
      "address": "456 VM2 Processing Street"
    },
    "certificationIds": ["CERT_VM2_001"]
  }'
```

---

## High Availability & Failover

### Multi-VM Redundancy

The backend setup supports high availability through:

1. **Peer Redundancy:** Each VM runs a different peer (peer0 on VM1, peer1 on VM2)
2. **Orderer RAFT:** 3-node consensus tolerates 1 orderer failure
3. **API Redundancy:** Both backends can serve requests independently

### Failover Testing

To test failover capabilities:

1. **Stop VM1 services:**
```bash
# On VM1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml stop peer0.trst01.example.com orderer1.example.com
```

2. **Verify VM2 continues working:**
```bash
# VM2 API should still respond
curl http://localhost:3001/health
```

3. **Restart VM1 services:**
```bash
# On VM1
docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up -d peer0.trst01.example.com orderer1.example.com
```

---

## Monitoring & Logging

### Backend Logs

Each backend instance provides detailed logs:

```bash
# View logs in real-time
npm start

# Or run in background with logging
npm start > backend.log 2>&1 &
tail -f backend.log
```

### Health Monitoring

Regular health checks should be performed:

```bash
# Monitor VM1 API
watch -n 30 curl -s http://localhost:3000/health

# Monitor VM2 API  
watch -n 30 curl -s http://localhost:3001/health
```

---

## Troubleshooting

### Common Issues

1. **"spawn peer ENOENT"** - Fabric binary path incorrect
   - Fix: Ensure `FABRIC_BIN_PATH=../../bin` in `.env`

2. **"connection refused"** - Peer not running
   - Fix: Start the peer container with docker-compose

3. **"access denied: creator is malformed"** - Chaincode not deployed
   - Fix: Deploy chaincode following `chaincode-reference.md`

4. **MSPID mismatch** - Wrong organization ID
   - Fix: Ensure `CORE_PEER_LOCALMSPID=Trst01MSP` in `.env`

### Debug Commands

```bash
# Check peer status
docker ps | grep peer

# Check chaincode status
peer lifecycle chaincode querycommitted --channelID oatschannel --name oats-traceability

# Test direct chaincode invoke
peer chaincode invoke -o localhost:7050 --ordererTLSHostnameOverride orderer1.example.com \
  --tls --cafile "$PWD/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem" \
  -C oatschannel -n oats-traceability \
  --peerAddresses localhost:7051 \
  --tlsRootCertFiles "$PWD/organizations/peerOrganizations/trst01.example.com/peers/peer0.trst01.example.com/tls/ca.crt" \
  -c '{"Args":["GetActor","ACTOR001"]}'
```

---

## Production Considerations

### Security

1. **TLS Encryption:** All Fabric communications use TLS
2. **API Authentication:** Consider adding API keys or JWT tokens
3. **Network Security:** Use firewalls to restrict access to peer/orderer ports

### Performance

1. **Connection Pooling:** Backend maintains persistent connections to peers
2. **Async Processing:** Consider message queues for high-volume transactions
3. **Caching:** Cache frequent queries to reduce ledger reads

### Backup & Recovery

1. **Ledger Backup:** Regular backups of Docker volumes
2. **Configuration Backup:** Version control all configuration files
3. **Disaster Recovery:** Document recovery procedures for complete VM failure

---

## API Rate Limits & Best Practices

### Recommended Limits

- **Actor Creation:** 10 requests/minute per API instance
- **Asset Creation:** 20 requests/minute per API instance  
- **Event Creation:** 50 requests/minute per API instance
- **Query Operations:** 100 requests/minute per API instance

### Best Practices

1. **Batch Operations:** Use bulk operations where possible
2. **Idempotency:** Include request IDs for retry safety
3. **Error Handling:** Implement exponential backoff for retries
4. **Monitoring:** Set up alerts for error rates and response times

---

## Version History

- **v1.0** - Initial backend API setup with multi-VM support
- **v1.1** - Added comprehensive error handling and logging
- **v1.2** - Enhanced failover and monitoring capabilities

---

## Support & Maintenance

For issues and maintenance:

1. Check this documentation first
2. Review backend logs for detailed error messages
3. Verify network connectivity between VMs
4. Ensure all Fabric containers are running
5. Validate chaincode deployment status

**Contact:** DevOps team for infrastructure issues, Development team for API issues.
