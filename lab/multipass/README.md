# Multipass Lab Environments

Local VM-based infrastructure deployments using Canonical Multipass for HashiCorp Nomad, Consul, and Vault testing and development.

## Overview

This directory contains multiple Terraform + Ansible configurations for deploying HashiCorp stack components on local Multipass VMs. Each subdirectory provides a complete, reproducible environment for different use cases.

## Available Configurations

### Nomad Deployments

| Configuration | Description | VMs | Use Case |
|--------------|-------------|-----|----------|
| [**nomad-cluster**](nomad-cluster/) | Production-like cluster | 3 servers + 2 clients | Full-featured testing, HA, service mesh |
| [**nomad-dev-cluster**](nomad-dev-cluster/) | Development cluster | 1 server + 1 client | Source code development, rapid iteration |
| [**nomad-server-cluster**](nomad-server-cluster/) | Server-only cluster | 3 servers | Server features, Raft testing, HA |
| [**nomad-single-machine**](nomad-single-machine/) | All-in-one instance | 1 VM (server + client) | Quick testing, demos, learning |

### Supporting Services

| Configuration | Description | VMs | Use Case |
|--------------|-------------|-----|----------|
| [**consul-single-machine**](consul-single-machine/) | Consul server | 1 server | Service discovery, integration testing |
| [**vault-single-machine**](vault-single-machine/) | Vault server | 1 server | Secrets management, workload identity |
| [**workstation**](workstation/) | Development VM | 1 VM | Remote development, building from source |

## Quick Start

### Prerequisites

Install required tools:

```bash
# Install Multipass
brew install multipass  # macOS
# or visit https://multipass.run for other platforms

# Install Terraform
brew install terraform

# Install Ansible
brew install ansible

# Ensure SSH key exists
ls ~/.ssh/id_rsa.pub || ssh-keygen -t rsa -b 4096
```

### Deploy Your First Environment

**Option 1: Simple All-in-One Nomad**
```bash
cd nomad-single-machine
terraform init
terraform apply
```

**Option 2: Full Nomad Cluster**
```bash
cd nomad-cluster
terraform init
terraform apply
```

**Option 3: Complete HashiCorp Stack**
```bash
# Deploy Consul
cd consul-single-machine
terraform init && terraform apply
export CONSUL_ADDR=$(terraform output -raw consul_server_address)

# Deploy Vault
cd ../vault-single-machine
terraform init && terraform apply
./scripts/init_unseal.sh --addr $(terraform output -raw vault_server_address)
export VAULT_ADDR=http://$(terraform output -raw vault_server_address):8200

# Deploy Nomad with integrations
cd ../nomad-cluster
terraform init
terraform apply \
  -var="consul_address=${CONSUL_ADDR}:8500" \
  -var="vault_address=${VAULT_ADDR}"
```

## Configuration Comparison

### Feature Matrix

| Feature | nomad-cluster | nomad-dev-cluster | nomad-server-cluster | nomad-single-machine |
|---------|---------------|-------------------|----------------------|----------------------|
| **Servers** | 3 | 1 | 3 | 1 |
| **Clients** | 2 | 1 | 0 | 1 (same VM) |
| **TLS** | ✅ | ❌ | ✅ | ✅ |
| **ACLs** | ✅ | ❌ | ✅ | ✅ |
| **Consul Integration** | Optional | ❌ | ❌ | ❌ |
| **Vault Integration** | Optional | ❌ | ❌ | ❌ |
| **Docker Driver** | ✅ | ✅ | N/A | ✅ |
| **Podman Driver** | ✅ | ❌ | N/A | ❌ |
| **CNI/Smuggle** | ✅ | ✅ | N/A | ✅ |
| **DNS (dnsmasq)** | ✅ | ❌ | N/A | ✅ |
| **Nomad UI** | ✅ | ❌ | ✅ | ✅ |
| **Go Installed** | ✅ | ✅ | ✅ | ❌ |
| **Best For** | Production testing | Source development | Server testing | Quick demos |

### Resource Requirements

| Configuration | Total CPUs | Total RAM | Deployment Time |
|--------------|------------|-----------|-----------------|
| nomad-cluster | 10+ | 20 GiB | 8-12 min |
| nomad-dev-cluster | 2 | 2 GiB | 3-5 min |
| nomad-server-cluster | 6+ | 12 GiB | 5-8 min |
| nomad-single-machine | 4 | 4 GiB | 5-7 min |
| consul-single-machine | 2 | 2 GiB | 2-3 min |
| vault-single-machine | 1 | 1 GiB | 2-3 min |
| workstation | 1 | 1 GiB | 3-5 min |

## Common Workflows

### 1. Learning Nomad

**Start simple, then scale up:**

```bash
# Step 1: Learn basics with single machine
cd nomad-single-machine
terraform apply

# Step 2: Understand clustering
cd ../nomad-cluster
terraform apply

# Step 3: Add service discovery
cd ../consul-single-machine
terraform apply
```

### 2. Development Workflow

**Build and test Nomad from source:**

```bash
# Deploy dev cluster
cd nomad-dev-cluster
terraform apply

# Sync your local Nomad source
rsync -r ~/Projects/nomad ubuntu@<server-ip>:/home/ubuntu/

# SSH and build
ssh <server-ip>
cd ~/nomad && make dev
sudo ./bin/nomad agent -config=~/nomad-server.hcl
```

### 3. Integration Testing

**Test Nomad with Consul and Vault:**

```bash
# Deploy all components
cd consul-single-machine && terraform apply
cd ../vault-single-machine && terraform apply
cd ../nomad-cluster && terraform apply \
  -var="consul_address=<consul-ip>:8500" \
  -var="vault_address=http://<vault-ip>:8200"

# Test workload identity
nomad job run ../../shared/jobs/example-with-vault.nomad
```

### 4. High Availability Testing

**Test failover scenarios:**

```bash
# Deploy server cluster
cd nomad-server-cluster
terraform apply

# Test leader election
ssh <leader-ip> "sudo systemctl stop nomad"
nomad server members  # Watch new leader election

# Test rolling upgrades
ansible-playbook playbook_nomad_server.yaml --limit nomad-server-0
# Verify, then continue with other servers
```

## Automation with Justfile

The [`Justfile`](Justfile:1) provides convenient commands for common operations:

```bash
# Install just
brew install just

# View available commands
just --list

# Deploy a configuration
just deploy nomad-cluster

# Destroy a configuration
just destroy nomad-cluster

# Deploy full stack
just deploy-stack
```

## Networking

### Default Network Configuration

Multipass VMs use the default network bridge:
- **Network**: 192.168.64.0/24 (macOS) or 192.168.122.0/24 (Linux)
- **DNS**: Provided by host
- **Internet**: NAT through host

### Accessing Services

**From Host Machine:**
```bash
# Direct access (if firewall allows)
curl http://192.168.64.X:4646/v1/status/leader

# Port forwarding (recommended for HTTPS)
ssh -L 4646:192.168.64.X:4646 192.168.64.X
# Then access https://localhost:4646
```

**Between VMs:**
VMs can communicate directly using their IP addresses.

## Troubleshooting

### Common Issues

**Multipass Not Starting VMs**
```bash
# Check Multipass status
multipass list
multipass version

# Restart Multipass daemon (macOS)
sudo launchctl stop com.canonical.multipassd
sudo launchctl start com.canonical.multipassd

# Check logs
multipass logs <vm-name>
```

**Terraform Apply Fails**
```bash
# Clean up state
terraform destroy
rm -rf .terraform terraform.tfstate*

# Re-initialize
terraform init
terraform apply
```

**Ansible Connection Issues**
```bash
# Test SSH connectivity
ssh ubuntu@<vm-ip>

# Check inventory
cat inventory.yaml

# Run with verbose output
ansible-playbook -vvv playbook_all.yaml
```

**Out of Disk Space**
```bash
# Check VM disk usage
multipass exec <vm-name> -- df -h

# Increase disk size (requires recreation)
multipass delete <vm-name>
multipass purge
# Edit main.tf to increase instance_disk
terraform apply
```

**Network Connectivity Issues**
```bash
# Check VM network
multipass exec <vm-name> -- ip addr
multipass exec <vm-name> -- ping -c 3 8.8.8.8

# Restart networking
multipass restart <vm-name>
```

### Getting Help

1. **Check Configuration README**: Each subdirectory has detailed documentation
2. **Review Logs**: 
   - Multipass: `multipass logs <vm-name>`
   - Nomad: `ssh <vm-ip> "sudo journalctl -u nomad -f"`
   - Consul: `ssh <vm-ip> "sudo journalctl -u consul -f"`
3. **Terraform Debug**: `TF_LOG=DEBUG terraform apply`
4. **Ansible Debug**: `ansible-playbook -vvv playbook.yaml`

## Cleanup

### Destroy Single Configuration

```bash
cd <configuration-directory>
terraform destroy
```

### Destroy All VMs

```bash
# List all VMs
multipass list

# Delete specific VMs
multipass delete nomad-server-* nomad-client-* consul-server-* vault-server-*

# Purge deleted VMs
multipass purge
```

### Clean Terraform State

```bash
# Remove all Terraform state files
find . -name "terraform.tfstate*" -delete
find . -name ".terraform" -type d -exec rm -rf {} +
```

## Best Practices

### Resource Management

1. **Start Small**: Begin with single-machine configs, scale up as needed
2. **Monitor Resources**: Use `multipass info` to check VM resource usage
3. **Clean Up**: Destroy unused environments to free resources
4. **Snapshots**: Use `multipass snapshot` before major changes

### Development Workflow

1. **Use Dev Cluster**: For source code development, use `nomad-dev-cluster`
2. **Test in Full Cluster**: Validate in `nomad-cluster` before production
3. **Iterate Quickly**: Use rsync for fast code synchronization
4. **Version Control**: Keep Terraform state in version control (with .gitignore)

### Security

⚠️ **These configurations are for development/testing only!**

- Default passwords and tokens are used
- TLS certificates are self-signed
- ACL tokens are pre-generated
- No production-grade security measures

**Never use these configurations in production!**

## Advanced Topics

### Custom VM Images

Create custom Multipass images with pre-installed software:

```bash
# Launch base VM
multipass launch --name base-image

# Install software
multipass exec base-image -- sudo apt-get update
multipass exec base-image -- sudo apt-get install -y <packages>

# Create snapshot
multipass snapshot base-image

# Use in Terraform (requires custom provider configuration)
```

### Multi-Region Setup

Deploy multiple Nomad regions:

```bash
# Region 1: lcy1 (London)
cd nomad-cluster
terraform apply

# Region 2: nyc1 (New York)
cd ../nomad-cluster-nyc
# Edit templates to use region "nyc1"
terraform apply

# Configure federation
nomad server join <region1-server-ip>
```

### Monitoring Stack

Deploy monitoring for your clusters:

```bash
# Deploy Nomad cluster
cd nomad-cluster
terraform apply

# Deploy monitoring jobs
nomad job run ../../shared/jobs/monitoring/prometheus-server.nomad.hcl
nomad job run ../../shared/jobs/monitoring/grafana-server.nomad.hcl
```

## Contributing

When adding new configurations:

1. Follow the existing directory structure
2. Include comprehensive README.md
3. Use shared Terraform modules
4. Use shared Ansible roles
5. Test on clean Multipass installation
6. Document resource requirements
7. Include troubleshooting section

## Related Documentation

- [Multipass Documentation](https://multipass.run/docs)
- [Nomad Documentation](https://www.nomadproject.io/docs)
- [Consul Documentation](https://www.consul.io/docs)
- [Vault Documentation](https://www.vaultproject.io/docs)
- [Terraform Documentation](https://www.terraform.io/docs)
- [Ansible Documentation](https://docs.ansible.com/)

## Directory Structure

```
lab/multipass/
├── README.md                      # This file
├── Justfile                       # Task automation
├── consul-single-machine/         # Consul server
├── nomad-cluster/                 # Full Nomad cluster
├── nomad-dev-cluster/             # Development cluster
├── nomad-server-cluster/          # Server-only cluster
├── nomad-single-machine/          # All-in-one Nomad
├── vault-single-machine/          # Vault server
└── workstation/                   # Development VM
```

## Support

For issues specific to:
- **Multipass**: Check [Multipass GitHub](https://github.com/canonical/multipass)
- **Terraform**: Check [Terraform Registry](https://registry.terraform.io/)
- **Ansible**: Check [Ansible Galaxy](https://galaxy.ansible.com/)
- **Nomad/Consul/Vault**: Check [HashiCorp Forums](https://discuss.hashicorp.com/)