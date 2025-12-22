# Ethereum Node Helm Chart

Production-grade Ethereum full node deployment on Kubernetes with:
- **Execution Layer**: Geth (Go Ethereum)
- **Consensus Layer**: Lighthouse
- **Multi-network support**: Mainnet, Goerli, Sepolia
- **GitOps-ready**: Flux CD compatible
- **Monitoring**: Prometheus metrics built-in

## � Prerequisites

- Kubernetes cluster (v1.24+)
- Helm 3.x
- `kubectl` configured
- Storage class available in your cluster

## �🚀 Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/gfournierPro/ethereum-node-helm.git
cd ethereum-node-helm
```

### 2. Lint the Chart

```bash
helm lint ./charts/ethereum-node
```

### 3. Install the Chart

**For Production (Mainnet):**
```bash
helm install eth-mainnet ./charts/ethereum-node \
  --namespace ethereum \
  --create-namespace \
  --values ./charts/ethereum-node/values-mainnet.yaml
```

**For Local Development:**
```bash
helm install eth-mainnet ./charts/ethereum-node \
  --namespace ethereum \
  --create-namespace \
  --values ./charts/ethereum-node/values-mainnet-local.yaml
```

**For Sepolia Testnet:**
```bash
helm install eth-sepolia ./charts/ethereum-node \
  --namespace sepolia \
  --create-namespace \
  --values ./charts/ethereum-node/values-sepolia.yaml
```

### 4. Verify Installation

```bash
# Check pods status
kubectl get pods -n ethereum -l app.kubernetes.io/instance=eth-mainnet

# Check PVCs
kubectl get pvc -n ethereum -l app.kubernetes.io/instance=eth-mainnet

# Watch pods
kubectl get pods -n ethereum -w
```

## 🔧 Useful Commands

### Pod Management

```bash
# Get all pods
kubectl get pods -n ethereum

# Describe a specific pod
kubectl describe pod eth-mainnet-ethereum-node-execution-0 -n ethereum
kubectl describe pod eth-mainnet-ethereum-node-consensus-0 -n ethereum

# Restart pods (delete to trigger recreation)
kubectl delete pod eth-mainnet-ethereum-node-execution-0 -n ethereum
kubectl delete pod eth-mainnet-ethereum-node-consensus-0 -n ethereum
```

### Logs

```bash
# Execution layer (Geth) logs
kubectl logs -f eth-mainnet-ethereum-node-execution-0 -n ethereum

# Consensus layer (Lighthouse) logs
kubectl logs -f eth-mainnet-ethereum-node-consensus-0 -n ethereum -c lighthouse

# Init container logs
kubectl logs eth-mainnet-ethereum-node-execution-0 -n ethereum -c generate-jwt
kubectl logs eth-mainnet-ethereum-node-consensus-0 -n ethereum -c wait-for-execution
kubectl logs eth-mainnet-ethereum-node-consensus-0 -n ethereum -c copy-jwt

# Previous container logs (after crash)
kubectl logs eth-mainnet-ethereum-node-consensus-0 -n ethereum -c lighthouse --previous
```

### Sync Status

```bash
# Check execution layer sync status
kubectl exec -it eth-mainnet-ethereum-node-execution-0 -n ethereum -- \
  geth attach /data/geth.ipc --exec 'eth.syncing'

# Check consensus layer sync status
kubectl exec -it eth-mainnet-ethereum-node-consensus-0 -n ethereum -- \
  wget -q -O - http://localhost:5052/eth/v1/node/syncing

# Get current block number
kubectl exec -it eth-mainnet-ethereum-node-execution-0 -n ethereum -- \
  geth attach /data/geth.ipc --exec 'eth.blockNumber'
```

### RPC Access

```bash
# Port-forward execution layer RPC
kubectl port-forward -n ethereum svc/eth-mainnet-ethereum-node-execution 8545:8545

# Test RPC (in another terminal)
curl -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

# Port-forward consensus layer HTTP API
kubectl port-forward -n ethereum svc/eth-mainnet-ethereum-node-consensus 5052:5052

# Test consensus API
curl http://localhost:5052/eth/v1/node/version
curl http://localhost:5052/eth/v1/node/syncing
```

### Storage & Resources

```bash
# Check disk usage
kubectl exec eth-mainnet-ethereum-node-execution-0 -n ethereum -- df -h /data
kubectl exec eth-mainnet-ethereum-node-consensus-0 -n ethereum -- df -h /data

# Check PVC status
kubectl get pvc -n ethereum

# Check storage classes
kubectl get storageclass

# Check node resources
kubectl describe node | grep -A 5 "Allocated resources"
```

### Helm Operations

```bash
# Upgrade release
helm upgrade eth-mainnet ./charts/ethereum-node \
  --namespace ethereum \
  --values ./charts/ethereum-node/values-mainnet-local.yaml

# Dry-run (preview changes)
helm upgrade eth-mainnet ./charts/ethereum-node \
  --namespace ethereum \
  --values ./charts/ethereum-node/values-mainnet-local.yaml \
  --dry-run

# View rendered templates
helm template eth-mainnet ./charts/ethereum-node \
  --values ./charts/ethereum-node/values-mainnet-local.yaml

# Uninstall release
helm uninstall eth-mainnet -n ethereum

# Delete PVCs (data will be lost!)
kubectl delete pvc --all -n ethereum

# List releases
helm list -n ethereum

# Get release history
helm history eth-mainnet -n ethereum

# Rollback to previous revision
helm rollback eth-mainnet 1 -n ethereum
```

### Debugging

```bash
# Check events
kubectl get events -n ethereum --sort-by='.lastTimestamp' | tail -20

# Check events for a specific pod
kubectl get events -n ethereum --field-selector involvedObject.name=eth-mainnet-ethereum-node-execution-0

# Describe pod for detailed info
kubectl describe pod eth-mainnet-ethereum-node-consensus-0 -n ethereum

# Check JWT secret
kubectl get secret eth-mainnet-ethereum-node-jwt -n ethereum -o jsonpath='{.data.jwt\.hex}' | base64 -d

# Shell into container
kubectl exec -it eth-mainnet-ethereum-node-execution-0 -n ethereum -- sh
```

## 📁 Values Files

| File | Description |
|------|-------------|
| `values.yaml` | Default values (production settings) |
| `values-mainnet.yaml` | Mainnet-specific configuration |
| `values-mainnet-local.yaml` | Local development (reduced resources) |
| `values-sepolia.yaml` | Sepolia testnet configuration |

### Key Configuration Options

```yaml
# Storage class (must exist in your cluster)
executionLayer:
  storage:
    storageClass: "local-path"  # or "gp3", "fast-ssd", etc.

consensusLayer:
  storage:
    storageClass: "local-path"

# Disable StorageClass creation (use existing)
storageClass:
  create: false

# Disable monitoring (if Prometheus Operator not installed)
monitoring:
  enabled: false
  serviceMonitor:
    enabled: false
```

## 🔍 Troubleshooting

### Pods Stuck in Pending

```bash
# Check why pod is pending
kubectl describe pod <pod-name> -n ethereum | grep -A 10 Events

# Common issues:
# - Insufficient memory/CPU → reduce resource requests
# - StorageClass not found → check available storage classes
# - PVC not binding → check PVC events
```

### Consensus Layer Crashing

```bash
# Check logs
kubectl logs eth-mainnet-ethereum-node-consensus-0 -n ethereum -c lighthouse

# Common issues:
# - Invalid JWT → check JWT secret format (must be 64 hex chars)
# - OOMKilled → increase memory limits
# - Invalid arguments → check extraArgs in values
```

### JWT Authentication Issues

```bash
# Verify JWT is valid hex (64 characters, 0-9 and a-f only)
kubectl get secret eth-mainnet-ethereum-node-jwt -n ethereum -o jsonpath='{.data.jwt\.hex}' | base64 -d | wc -c
# Should output: 64

# Regenerate JWT
kubectl delete secret eth-mainnet-ethereum-node-jwt -n ethereum
helm upgrade eth-mainnet ./charts/ethereum-node --namespace ethereum --values ./charts/ethereum-node/values-mainnet-local.yaml
kubectl delete pod -l app.kubernetes.io/instance=eth-mainnet -n ethereum
```

## 📊 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                       │
│  ┌───────────────────────┐    ┌───────────────────────────┐ │
│  │   Execution Layer     │    │   Consensus Layer         │ │
│  │   (Geth StatefulSet)  │    │   (Lighthouse StatefulSet)│ │
│  │                       │    │                           │ │
│  │  ┌─────────────────┐  │    │  ┌─────────────────────┐  │ │
│  │  │ geth container  │◄─┼────┼──│ lighthouse container│  │ │
│  │  │   Port 8545     │  │JWT │  │     Port 5052       │  │ │
│  │  │   Port 8551     │──┼────┼─►│     Port 9000       │  │ │
│  │  └─────────────────┘  │    │  └─────────────────────┘  │ │
│  │          │            │    │           │               │ │
│  │  ┌───────▼───────┐    │    │  ┌────────▼────────┐     │ │
│  │  │  PVC (data)   │    │    │  │   PVC (data)    │     │ │
│  │  └───────────────┘    │    │  └─────────────────┘     │ │
│  └───────────────────────┘    └───────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## ⚠️ Important Notes

- **Initial sync takes 6-24 hours** depending on network and hardware
- **Mainnet requires ~1TB+ storage** for full sync (use snap sync)
- **Sepolia requires ~100GB storage**
- **Ensure SSD storage** for optimal performance
- **Monitor disk usage** regularly during sync

## 📖 Resources

- [Geth Documentation](https://geth.ethereum.org/docs)
- [Lighthouse Documentation](https://lighthouse-book.sigmaprime.io/)
- [Ethereum on Kubernetes](https://ethereum.org/en/developers/docs/nodes-and-clients/run-a-node/)

## 📝 License

MIT License