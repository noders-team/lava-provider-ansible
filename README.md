# Lava Provider Ansible Playbook

This Ansible playbook automates the deployment of a **Lava Protocol RPC Provider** on Ubuntu/Debian systems. It handles everything from binary compilation to service configuration and SSL setup.

## 📋 Requirements

- **Target Server**: Ubuntu 20.04+ or Debian 11+
- **Ansible**: Version 2.9+
- **Privileges**: SSH access with sudo privileges
- **Resources**: Minimum 4GB RAM, 2 CPU cores, 100GB disk space
- **Blockchain Nodes**: Access to blockchain RPC endpoints you want to serve

## 🚀 Quick Start

### 1. Clone Repository

```bash
git clone <repository-url>
cd lava-provider
```

### 2. Configure Your Setup

**Edit `group_vars/all.yml`:**

```yaml
# REQUIRED CHANGES:
lava_provider_config:
  wallet:
    name: "your-wallet-name"  # Change this
  moniker: "your-provider-name"  # Change this
  
chains:
  - name: "Ethereum Mainnet"
    endpoints:
      - chain_id: "ETH1"
        api_interface: "jsonrpc"
        network_address: "127.0.0.1:22005"
        disable_tls: true
        node_urls:
          - "https://YOUR-ETH-NODE-URL"  # Replace with your Ethereum RPC

# For SSL (optional):
lava_nginx:
  domain: "your-domain.com"  # Change this
  certbot_enabled: true      # Enable SSL
  certbot_email: "your@email.com"  # Change this
```

### 3. Create Inventory

**Create `inventory/hosts.yml`:**

```yaml
all:
  hosts:
    your-server:
      ansible_host: YOUR_SERVER_IP
      ansible_user: ubuntu
      ansible_ssh_private_key_file: ~/.ssh/your-key.pem
```

### 4. Deploy

```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup.yml
```

## ⚙️ Configuration Guide

### Blockchain Endpoints Configuration

The most important part is configuring your blockchain endpoints in `group_vars/all.yml`:

#### Basic Structure

```yaml
chains:
  - name: "Human Readable Name"
    endpoints:
      - chain_id: "LAVA_CHAIN_ID"       # Official Lava chain identifier
        api_interface: "jsonrpc"         # API type: jsonrpc, rest, tendermintrpc, grpc
        network_address: "127.0.0.1:PORT"  # Your provider listening address
        disable_tls: true                # true for local, false for external HTTPS
        node_urls:                       # Your actual blockchain nodes
          - "https://your-node-url"
          - "wss://your-websocket-url"
```

#### Supported Chain IDs

Check the latest supported chains at [Lava Network](https://lava.lavanet.xyz):

**EVM Chains:**
- `ETH1` - Ethereum Mainnet
- `ARBITRUM` - Arbitrum Mainnet  
- `POLYGON` - Polygon Mainnet
- `AVAX` - Avalanche C-Chain
- `BSC` - Binance Smart Chain
- `OPTM` - Optimism Mainnet
- `BASE` - Base Mainnet
- `BLAST` - Blast Mainnet

**Cosmos Chains:**
- `COSMOSHUB` - Cosmos Hub
- `OSMOSIS` - Osmosis
- `JUNO` - Juno Network
- `AXELAR` - Axelar Network
- `EVMOS` - Evmos

**Other Chains:**
- `SOLANA` - Solana
- `NEAR` - Near Protocol
- `APT1` - Aptos
- `STRK` - StarkNet

#### API Interface Types

- **`jsonrpc`**: For EVM chains, Solana, Near
- **`rest`**: For Cosmos SDK chains REST API
- **`tendermintrpc`**: For Cosmos SDK chains RPC
- **`grpc`**: For Cosmos SDK chains gRPC

#### Example Configurations

**Ethereum Mainnet:**
```yaml
- name: "Ethereum Mainnet"
  endpoints:
    - chain_id: "ETH1"
      api_interface: "jsonrpc"
      network_address: "127.0.0.1:22005"
      disable_tls: true
      node_urls:
        - "https://eth-mainnet.gateway.pokt.network/v1/YOUR_APP_ID"
        - "wss://ethereum-mainnet-rpc.allthatnode.com/YOUR_API_KEY"
```

**Cosmos Hub (Multiple APIs):**
```yaml
- name: "Cosmos Hub Mainnet"
  endpoints:
    # REST API
    - chain_id: "COSMOSHUB"
      api_interface: "rest"
      network_address: "127.0.0.1:26005"
      disable_tls: true
      node_urls:
        - "https://cosmos-rest.publicnode.com"
    
    # Tendermint RPC
    - chain_id: "COSMOSHUB"
      api_interface: "tendermintrpc"
      network_address: "127.0.0.1:26005"
      disable_tls: true
      node_urls:
        - "https://cosmos-rpc.publicnode.com"
        - "wss://cosmos-rpc.publicnode.com/websocket"
    
    # gRPC
    - chain_id: "COSMOSHUB"
      api_interface: "grpc"
      network_address: "127.0.0.1:26005"
      disable_tls: true
      node_urls:
        - "cosmos-grpc.publicnode.com:443"
```

### Wallet Configuration

```yaml
lava_provider_config:
  wallet:
    name: "provider-wallet"     # Your wallet identifier
    keyring_backend: test       # test, file, or os
    password: ""               # For file backend only
```

### Network Configuration

```yaml
lava_provider_config:
  chain_id: lava-mainnet-1           # lava-mainnet-1 or lava-testnet-2
  geolocation: 2                     # 1=US, 2=EU, etc.
  public_rpc_url: "https://public-rpc.lavanet.xyz:443"
  moniker: "your-provider-name"      # Your provider identifier
```

### SSL Configuration

```yaml
lava_nginx:
  enabled: true
  domain: "provider.yourdomain.com"
  certbot_enabled: true              # Enable Let's Encrypt
  certbot_email: "admin@yourdomain.com"
```

## 🏃 Post-Installation

### 1. Wallet Setup

The playbook automatically creates a wallet during installation. If you need to import an existing wallet:

```bash
sudo -u lava-provider lavap keys delete provider-wallet --keyring-backend test --home /home/lava-provider/.lava-provider
sudo -u lava-provider lavap keys add provider-wallet --keyring-backend test --home /home/lava-provider/.lava-provider --recover
```

### 2. Fund Your Wallet

Get LAVA tokens for staking (minimum 50000000000ulava = 50,000 LAVA):

```bash
# Check wallet address
sudo -u lava-provider lavap keys show provider-wallet --keyring-backend test --home /home/lava-provider/.lava-provider

# Fund this address from your main wallet or faucet
```

### 3. Test Your Provider

```bash
sudo -u lava-provider lavap test rpcprovider \
  --from "provider-wallet" \
  --endpoints "127.0.0.1:22005,jsonrpc,ETH1" \
  --keyring-backend test \
  --home /home/lava-provider/.lava-provider \
  --chain-id lava-mainnet-1 \
  --node https://public-rpc.lavanet.xyz:443
```

### 4. Monitor Your Provider

```bash
# Check service status
sudo systemctl status lava-provider

# View logs
sudo journalctl -fu lava-provider

# Check active endpoints
sudo journalctl -u lava-provider | grep "active endpoints"
```

### 5. Stake Your Provider

Once testing is successful, stake your provider on the Lava Network:

```bash
sudo -u lava-provider lavap tx pairing stake-provider \
  ETH1 \
  50000000000ulava \
  "your-provider-domain.com:443,1" \
  1 \
  --from provider-wallet \
  --keyring-backend test \
  --home /home/lava-provider/.lava-provider \
  --chain-id lava-mainnet-1 \
  --node https://public-rpc.lavanet.xyz:443 \
  --gas-adjustment 1.5 \
  --gas auto \
  --yes
```

## 🛠️ Management Commands

### Service Management

```bash
# Restart provider
sudo systemctl restart lava-provider

# Stop provider
sudo systemctl stop lava-provider

# Check status
sudo systemctl status lava-provider
```

### Configuration Updates

After modifying `group_vars/all.yml`:

```bash
# Update configuration only
ansible-playbook -i inventory/hosts.yml playbooks/setup.yml --tags config

# Full redeployment
ansible-playbook -i inventory/hosts.yml playbooks/setup.yml
```

### Clean Installation

```bash
ansible-playbook -i inventory/hosts.yml playbooks/clean.yml
```

## 🔍 Troubleshooting

### Common Issues

**1. Binary Build Fails**
```bash
# Check Go installation
which go
go version

# Manual binary download
wget https://github.com/lavanet/lava/releases/download/v3.0.3/lavap_v3.0.3_linux_amd64
sudo mv lavap_v3.0.3_linux_amd64 /usr/local/bin/lavap
sudo chmod +x /usr/local/bin/lavap
```

**2. Service Won't Start**
```bash
# Check configuration
sudo -u lava-provider lavap validate-config /home/lava-provider/.lava-provider/config/rpcprovider.yml

# Check wallet exists
sudo -u lava-provider lavap keys list --keyring-backend test --home /home/lava-provider/.lava-provider
```

**3. No Active Endpoints**
```bash
# Verify node connectivity
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  https://your-eth-node-url

# Check endpoint configuration
cat /home/lava-provider/.lava-provider/config/rpcprovider.yml
```

**4. Protocol Version Mismatch**
If you see errors like "minimum protocol version mismatch":
```bash
# Update to newer Lava version in group_vars/all.yml
lava_binary:
  image_tag: v5.4.1  # Or latest version

# Rebuild and restart
ansible-playbook -i inventory/hosts.yml playbooks/setup.yml --tags binary
sudo systemctl restart lava-provider
```

### Useful Links

- [Lava Protocol Documentation](https://docs.lavanet.xyz)
- [Provider Setup Guide](https://docs.lavanet.xyz/provider-setup)
- [Chain Specifications](https://lava.lavanet.xyz)
- [Discord Support](https://discord.gg/lavanetxyz)

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request
