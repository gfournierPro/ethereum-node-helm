# Ethereum Node Helm Chart

Production-grade Ethereum full node deployment on Kubernetes with:
- **Execution Layer**: Geth (Go Ethereum)
- **Consensus Layer**: Lighthouse
- **Multi-network support**: Mainnet, Goerli, Sepolia
- **GitOps-ready**: Flux CD compatible
- **Monitoring**: Prometheus metrics built-in

## 🚀 Quick Start

```bash
# Add the Helm repository
helm repo add ethereum-node https://your-org.github.io/ethereum-node-helm

# Install mainnet node
helm install eth-mainnet ethereum-node/ethereum-node \
  --namespace ethereum \
  --create-namespace \
  --values values-mainnet.yaml

# Check deployment status
kubectl get pods -n ethereum
kubectl logs -f eth-mainnet-execution-0 -n ethereum
```