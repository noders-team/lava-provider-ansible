# Lava Provider Setup

This project is developed and maintained by [NODERS LLC](https://noders.team/) - Professional Validators and Web3 Developers.

## About

This repository contains Ansible playbooks for setting up and managing a Lava Network RPC provider node.

## Prerequisites

- Ubuntu 22.04 LTS (Also Verified on Ubuntu 24.04 LTS and Debian 12)
- Ansible 2.9 or later
- At least 4GB RAM
- At least 100GB free disk space
- Domain name with DNS A record pointing to your server (for SSL)
- LAVA tokens for staking

## Configuration

### 1. Configure Variables

Edit `group_vars/all.yml` with your specific settings:

```yaml
# User configuration
lava_user: lava                        # System user for Lava provider
lava_group: lava                       # System group for Lava provider

# Binary configuration
lava_binary:
  name: lavad                          # Binary name (lavad)
  git_repository: https://github.com/lavanet/lava  # Lava repository
  image_tag: v3.0.3                    # Version to install
  go_version: 1.21.5                   # Go version required
  install_dir: /opt/lava               # Installation directory
  data_dir: /opt/lava/data             # Data directory

# Provider configuration
lava_provider_config:
  chain_id: lava-testnet-2             # Lava network chain ID
  geolocation: 2                       # Your geolocation (2=Europe, 1=US-Center, etc.)
  wallet:
    name: lava-wallet                  # Wallet name
    keyring_backend: test              # Keyring backend
    password: ""                       # Wallet password (set during setup)
  log_level: info                      # Log level
  ports:
    grpc: 22001                        # gRPC port for provider
    metrics: 23001                     # Metrics port
  moniker: "CHANGE-ME"                 # Your provider moniker
  public_rpc_url: "https://lava-t-rpc.noders.services"  # Lava RPC endpoint
  total_connections: 25                # Max connections

# Chain endpoints configuration
chains:
  - name: "Arbitrum mainnet"
    endpoints:
      - chain_id: "ARB1"
        api_interface: "jsonrpc"
        network_address: "0.0.0.0:22001"
        disable_tls: true
        node_urls:
          - "http://arbitrum-mainnet:8545"

# Nginx and SSL configuration
lava_nginx:
  enabled: true                        # Enable Nginx reverse proxy
  domain: example.com                  # Your domain name
  certbot_enabled: true                # Enable SSL certificates
  certbot_email: admin@example.com     # Email for SSL certificates
```

### 2. Geolocation Options

Choose the appropriate geolocation for your provider:

- **0**: Global-strict (assigned via governance only)
- **1**: US-Center (North America) - **Default**
- **2**: Europe

### 3. Supported Chains

Configure the chains you want to provide RPC services for. Each chain requires:
- Running blockchain node
- Proper `chain_id` from Lava specs
- Appropriate `api_interface` (jsonrpc, tendermintrpc, grpc, rest)
- Unique network address and port

## Installation

### 1. Setup Provider

```bash
# Run the setup playbook
ansible-playbook playbooks/setup.yml -i localhost, --connection=local
```

This will:
- Install Go and required dependencies
- Build and install Lava binary
- Create lava user and directories
- Configure provider settings
- Set up Nginx with SSL (if enabled)
- Create systemd service

### 2. Post-Installation Setup

After installation, you need to:

1. **Import your wallet**:
```bash
sudo -u lava lavad keys add lava-wallet --keyring-backend test --recover
```

2. **Get LAVA tokens** for staking (minimum 50000000000ulava)

3. **Test your provider**:
```bash
lavad test rpcprovider --from lava-wallet --endpoints "your-domain.com:443,CHAINID" --node https://lava-t-rpc.noders.services --keyring-backend test
```

4. **Stake your provider**:
```bash
lavad tx pairing stake-provider CHAINID "50000000000ulava" "your-domain.com:443,1" 1 lava@validatoraddress -y --from lava-wallet --provider-moniker "your-moniker" --delegate-limit "0ulava" --gas-adjustment "1.5" --gas "auto" --gas-prices "0.0001ulava" --node https://lava-t-rpc.noders.services --keyring-backend test --chain-id lava-testnet-2
```

### 3. Clean Installation

To remove the installation:

```bash
# Run the cleanup playbook
ansible-playbook playbooks/clean.yml -i localhost, --connection=local
```

This will:
- Stop the Lava provider service
- Remove installation directories
- Remove lava user and group
- Clean up Nginx configuration

## Provider Management

### Systemd Service Commands

```bash
# Check service status
sudo systemctl status lava-provider

# Start the provider
sudo systemctl start lava-provider

# Stop the provider
sudo systemctl stop lava-provider

# Restart the provider
sudo systemctl restart lava-provider

# Enable service to start on boot
sudo systemctl enable lava-provider

# Disable service from starting on boot
sudo systemctl disable lava-provider
```

### Viewing Logs

```bash
# View service logs
sudo journalctl -u lava-provider -f

# View last 100 lines
sudo journalctl -u lava-provider -n 100

# View logs since last boot
sudo journalctl -u lava-provider -b
```

### Provider Management Commands

```bash
# Check provider status
lavad q pairing account-info YOUR_LAVA_ADDRESS --node https://lava-t-rpc.noders.services

# Unfreeze provider (takes 30 minutes)
lavad tx pairing unfreeze CHAINID --from lava-wallet --gas-adjustment "1.5" --gas "auto" --gas-prices "0.0001ulava" --node https://lava-t-rpc.noders.services --keyring-backend test --chain-id lava-testnet-2 -y

# Check available chains and specs
curl -X 'GET' 'https://public-rpc-testnet2.lavanet.xyz/rest/lavanet/lava/spec/show_all_chains' -H 'accept: application/json' | jq
```

### Directory Structure

```
/home/lava/.lava/
├── config/
│   └── rpcprovider.yml    # Provider configuration
├── rewards-storage/        # Rewards data
└── provider.env           # Environment variables

/opt/lava/
├── src/                   # Source code
└── data/                  # Data directory

/usr/local/bin/
└── lavad                  # Lava binary
```

## Architecture

### How Lava Provider Works

1. **Central Router**: Lava provider acts as a central routing service
2. **Chain Endpoints**: Each blockchain has its own endpoint configuration
3. **Request Flow**: `Client → Nginx → Lava Provider → Blockchain Node`
4. **Multi-Chain**: Single provider can serve multiple blockchains

### Port Configuration

- **22001**: Main provider gRPC port
- **23001**: Metrics port
- **22002, 22003...**: Individual chain endpoints
- **80/443**: Nginx HTTP/HTTPS ports

## Troubleshooting

### Common Issues

1. **Binary Not Found**
   - Check if `/usr/local/bin/lavad` exists
   - Verify PATH includes `/usr/local/bin`
   - Rebuild with `ansible-playbook playbooks/setup.yml`

2. **SSL Certificate Issues**
   - Verify domain DNS points to your server
   - Check certbot logs: `sudo journalctl -u certbot`
   - Manually request certificate: `sudo certbot --nginx -d your-domain.com`

3. **Provider Connection Issues**
   - Check if blockchain nodes are running and accessible
   - Verify node URLs in configuration
   - Test RPC endpoints manually

4. **Staking Issues**
   - Ensure you have enough LAVA tokens
   - Check wallet balance: `lavad q bank balances YOUR_ADDRESS`
   - Verify chain ID and geolocation parameters

### Log Files

- Provider logs: `journalctl -u lava-provider`
- Nginx logs: `/var/log/nginx/lava_provider_*.log`
- System logs: `journalctl -f`

### Testing Provider

```bash
# Test specific chain endpoint
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  https://your-domain.com

# Test with Lava test tool
lavad test rpcprovider --from lava-wallet \
  --endpoints "your-domain.com:443,ETH1" \
  --node https://lava-t-rpc.noders.services \
  --keyring-backend test
```

## Security Considerations

1. **Wallet Security**
   - Use strong passwords for wallet
   - Backup keyring files securely
   - Consider hardware security modules for mainnet

2. **SSL/TLS**
   - Always use SSL for public providers
   - Keep certificates updated
   - Use strong SSL configurations

3. **System Security**
   - Keep system updated
   - Configure firewall rules
   - Monitor system logs
   - Limit SSH access

4. **Provider Security**
   - Monitor provider performance
   - Set up alerting for downtime
   - Backup configuration files

## Monitoring

### Metrics

Provider exposes Prometheus metrics on the configured metrics port:
- `http://localhost:23001/metrics`

### Health Checks

- Provider health: Check systemd service status
- Nginx health: Check HTTP response codes
- Chain health: Monitor blockchain node connectivity

## Support

For issues and support:
1. Check the [official Lava documentation](https://docs.lavanet.xyz)
2. Review logs for specific error messages
3. Contact NODERS LLC via [official NODERS LLC Telegram Chat](https://t.me/noderschatru)
4. Join Lava Discord community

## NODERS LLC

- **Website**: [noders.team](https://noders.team)
- **Contact**: office@noders.team
- **Services**:
  - Professional Blockchain Validation
  - Web3 Development
  - Infrastructure Services
  - IBC Relayers
  - Nodes as a Service

With over $100M+ securely staked, 40+ mainnets supported, and 99.0% uptime since 2017, NODERS is a trusted name in the blockchain infrastructure space.
