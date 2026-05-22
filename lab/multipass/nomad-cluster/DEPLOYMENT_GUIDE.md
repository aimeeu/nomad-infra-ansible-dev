# Nomad Cluster Deployment Guide - Using Justfile

Complete guide for deploying a production-like Nomad cluster with optional Consul and Vault integration using the automated Justfile workflow.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Understanding the Justfile](#understanding-the-justfile)
- [Deployment Methods](#deployment-methods)
- [Post-Deployment Steps](#post-deployment-steps)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Overview

The [`Justfile`](../Justfile:1) in [`lab/multipass/`](../Justfile:1) provides automated deployment commands for the complete HashiCorp stack (Consul, Vault, and Nomad). This guide focuses on deploying the Nomad cluster, either standalone or as part of the full stack.

### What Gets Deployed

**Standalone Nomad Cluster:**
- 3 Nomad servers (2 CPU, 4GiB RAM each)
- 2 Nomad clients (default CPU, 4GiB RAM each)
- TLS encryption enabled
- ACLs enabled
- Docker + Podman drivers
- CNI networking with Smuggle (VXLAN)
- dnsmasq for DNS resolution

**Full HashiStack (with `hashistack_up`):**
- 1 Consul server (service discovery)
- 1 Vault server (secrets management)
- 3 Nomad servers + 2 Nomad clients (as above)
- Vault workload identity configured
- Integrated service mesh capabilities

## Prerequisites

### Required Tools

Install all required tools before deployment:

```bash
# macOS
brew install multipass terraform ansible jq just nomad vault

# Linux (Ubuntu/Debian)
sudo snap install multipass
# Install Terraform
wget https://releases.hashicorp.com/terraform/1.9.0/terraform_1.9.0_linux_amd64.zip
unzip terraform_1.9.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Install Ansible
sudo apt update
sudo apt install -y ansible

# Install jq
sudo apt install -y jq

# Install just
curl --proto '=https' --tlsv1.2 -sSf https://just.systems/install.sh | bash -s -- --to /usr/local/bin

# Install Nomad CLI
wget https://releases.hashicorp.com/nomad/2.0.0-beta.1/nomad_2.0.0-beta.1_linux_amd64.zip
unzip nomad_2.0.0-beta.1_linux_amd64.zip
sudo mv nomad /usr/local/bin/

# Install Vault CLI
wget https://releases.hashicorp.com/vault/1.18.0/vault_1.18.0_linux_amd64.zip
unzip vault_1.18.0_linux_amd64.zip
sudo mv vault /usr/local/bin/
```

### Verify Installation

```bash
# Check all tools are installed
multipass version
terraform version
ansible --version
jq --version
just --version
nomad version
vault version
```

### SSH Key Setup

Ensure you have an SSH key pair:

```bash
# Check if key exists
ls ~/.ssh/id_rsa.pub

# If not, create one
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

### System Requirements

**Minimum Host Resources:**
- **CPU**: 12+ cores (for full stack)
- **RAM**: 24+ GiB (for full stack)
- **Disk**: 50+ GiB free space
- **OS**: macOS 10.15+, Ubuntu 20.04+, or Windows 10+ with WSL2

**For Standalone Nomad Only:**
- **CPU**: 10+ cores
- **RAM**: 20+ GiB
- **Disk**: 30+ GiB

## Understanding the Justfile

### Location and Structure

The [`Justfile`](../Justfile:1) is located at [`lab/multipass/Justfile`](../Justfile:1) and defines automated workflows.

### Key Variables

```bash
multipass_root := justfile_directory()           # /path/to/lab/multipass
consul_dir := multipass_root + "/consul-single-machine"
vault_dir := multipass_root + "/vault-single-machine"
nomad_dir := multipass_root + "/nomad-cluster"
```

### Available Commands

View all available commands:

```bash
cd lab/multipass
just --list
```

**Output:**
```
Available recipes:
    hashistack_up   # Start a Vault server, Consul server, and Nomad cluster
    hashistack_down # Stop and destroy the Vault server, Consul server, and Nomad cluster
```

### What `hashistack_up` Does

The [`hashistack_up`](../Justfile:30) command performs these actions in sequence:

1. **Preflight Checks** ([`_preflight`](../Justfile:12))
   - Verifies all required tools are installed
   - Exits with error if any tool is missing

2. **Deploy Consul Server** (lines 33-34)
   ```bash
   terraform -chdir="consul-single-machine" init
   terraform -chdir="consul-single-machine" apply -auto-approve
   ```

3. **Deploy Vault Server** (lines 35-36)
   ```bash
   terraform -chdir="vault-single-machine" init
   terraform -chdir="vault-single-machine" apply -auto-approve
   ```

4. **Initialize and Unseal Vault** (line 38)
   ```bash
   scripts/init_unseal.sh --addr "$(terraform output vault_server_address)"
   ```
   - Creates `generated_vault_init.json` with root token and unseal keys

5. **Deploy Nomad Cluster** (lines 42-46)
   ```bash
   terraform -chdir="nomad-cluster" init
   terraform -chdir="nomad-cluster" apply -auto-approve \
     -var "consul_address=$(terraform output consul_server_address)" \
     -var "vault_address=$(terraform output vault_server_address)"
   ```
   - Passes Consul and Vault addresses to Nomad configuration

6. **Configure Vault Workload Identity** (lines 49-55)
   ```bash
   terraform -chdir="nomad-cluster/workload-identity" init
   terraform -chdir="nomad-cluster/workload-identity" apply -auto-approve \
     -var "nomad_addresses=..." \
     -var "nomad_cert_pem=..." \
     -var "vault_address=..." \
     -var "vault_token=..."
   ```
   - Sets up JWT authentication for Nomad workloads to access Vault

7. **Create Environment File** (line 57)
   ```bash
   just _hashistack_refresh_envrc
   ```
   - Combines `.envrc` files from all three services
   - Creates `lab/multipass/.envrc` with all environment variables

### What `hashistack_down` Does

The [`hashistack_down`](../Justfile:60) command destroys the infrastructure:

1. **Remove Workload Identity State** (lines 66-67)
   - Deletes Terraform state files (VMs will be destroyed anyway)

2. **Destroy Nomad Cluster** (lines 69-73)
   ```bash
   terraform -chdir="nomad-cluster" destroy -auto-approve
   ```

3. **Destroy Vault Server** (lines 75-76)
   ```bash
   terraform -chdir="vault-single-machine" destroy -auto-approve
   ```

4. **Destroy Consul Server** (lines 77-78)
   ```bash
   terraform -chdir="consul-single-machine" destroy -auto-approve
   ```

5. **Clean Environment File** (line 80)
   ```bash
   just _hashistack_destroy_envrc
   ```

## Deployment Methods

### Method 1: Full HashiStack (Recommended)

Deploy Consul, Vault, and Nomad together with full integration:

```bash
# Navigate to multipass directory
cd lab/multipass

# Deploy entire stack
just hashistack_up
```

**Deployment Time:** 15-20 minutes

**What Happens:**
1. Creates 1 Consul VM (2 min)
2. Creates 1 Vault VM (2 min)
3. Initializes and unseals Vault (1 min)
4. Creates 5 Nomad VMs (3 servers + 2 clients) (5 min)
5. Configures all services with Ansible (8 min)
6. Sets up Vault workload identity (2 min)
7. Creates `.envrc` file

**Expected Output:**
```
✓ Consul server deployed at 192.168.64.5:8500
✓ Vault server deployed at https://192.168.64.6:8200
✓ Vault initialized and unsealed
✓ Nomad cluster deployed (3 servers, 2 clients)
✓ Workload identity configured
✓ Environment file created at .envrc
```

### Method 2: Standalone Nomad Cluster

Deploy only the Nomad cluster without Consul or Vault:

```bash
# Navigate to nomad-cluster directory
cd lab/multipass/nomad-cluster

# Initialize Terraform
terraform init

# Deploy cluster
terraform apply
```

**Deployment Time:** 10-12 minutes

**What Happens:**
1. Creates TLS Certificate Authority
2. Creates 3 Nomad server VMs (2 CPU, 4GiB RAM each)
3. Creates 2 Nomad client VMs (default CPU, 4GiB RAM each)
4. Generates TLS certificates for all nodes
5. Runs Ansible playbooks to configure:
   - Nomad servers with Raft consensus
   - Nomad clients with Docker + Podman
   - CNI networking and Smuggle
   - dnsmasq for DNS resolution
6. Creates local `.envrc` file

**Expected Output:**
```
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

msg = <<EOT
SSH commands:
  Nomad Servers:
    - ssh 192.168.64.10
    - ssh 192.168.64.11
    - ssh 192.168.64.12
  Nomad Client:
    - ssh 192.168.64.13
    - ssh 192.168.64.14

Nomad HTTP API:
    - https://192.168.64.10:4646
    - https://192.168.64.11:4646
    - https://192.168.64.12:4646
EOT
```

### Method 3: Nomad with Existing Consul/Vault

Deploy Nomad and connect to existing Consul and Vault servers:

```bash
# Deploy Consul first
cd lab/multipass/consul-single-machine
terraform init
terraform apply
export CONSUL_ADDR=$(terraform output -raw consul_server_address)

# Deploy Vault
cd ../vault-single-machine
terraform init
terraform apply
./scripts/init_unseal.sh --addr $(terraform output -raw vault_server_address)
export VAULT_ADDR=$(terraform output -raw vault_server_address)

# Deploy Nomad with integration
cd ../nomad-cluster
terraform init
terraform apply \
  -var="consul_address=${CONSUL_ADDR}:8500" \
  -var="vault_address=https://${VAULT_ADDR}:8200"
```

## Post-Deployment Steps

### 1. Load Environment Variables

**For Full Stack Deployment:**
```bash
cd lab/multipass
source .envrc
```

**For Standalone Nomad:**
```bash
cd lab/multipass/nomad-cluster
source .envrc
```

The `.envrc` file contains:
```bash
export NOMAD_ADDR=https://192.168.64.10:4646
export NOMAD_CACERT=/path/to/.tls/ca.pem
export NOMAD_CLIENT_CERT=/path/to/.tls/nomad-client-0.pem
export NOMAD_CLIENT_KEY=/path/to/.tls/nomad-client-0-key.pem
export NOMAD_TOKEN=placeholder-update-after-bootstrap
```

### 2. Bootstrap ACLs

ACLs are enabled but not yet bootstrapped. You must bootstrap to get the root token:

```bash
# SSH to any server
ssh $(terraform output -json nomad_server_addresses | jq -r '.[0]')

# Bootstrap ACLs
nomad acl bootstrap

# Output:
# Accessor ID  = 12345678-1234-1234-1234-123456789012
# Secret ID    = abcdef12-3456-7890-abcd-ef1234567890  # <-- This is your root token
# Name         = Bootstrap Token
# Type         = management
# Global       = true
# Create Time  = 2024-01-15 10:30:00 +0000 UTC
```

**IMPORTANT:** Save the `Secret ID` - this is your root token!

### 3. Update Environment Variables

Update `.envrc` with the real token:

```bash
# Edit .envrc
vim .envrc

# Replace placeholder with actual token
export NOMAD_TOKEN=abcdef12-3456-7890-abcd-ef1234567890

# Reload environment
source .envrc
```

### 4. Verify Cluster Status

```bash
# Check server members
nomad server members
# Output:
# Name                    Address         Port  Status  Leader  Raft Version  Build  Datacenter  Region
# nomad-server-0.lcy1     192.168.64.10   4648  alive   true    3             2.0.0  lcy1        lcy1
# nomad-server-1.lcy1     192.168.64.11   4648  alive   false   3             2.0.0  lcy1        lcy1
# nomad-server-2.lcy1     192.168.64.12   4648  alive   false   3             2.0.0  lcy1        lcy1

# Check client nodes
nomad node status
# Output:
# ID        Node Pool  DC    Name            Class   Drain  Eligibility  Status
# abc123    default    lcy1  nomad-client-0  <none>  false  eligible     ready
# def456    default    lcy1  nomad-client-1  <none>  false  eligible     ready

# Check cluster health
nomad status
```

### 5. Access Nomad UI

Open your browser to any server address:

```bash
# Get server addresses
terraform output msg

# Open in browser (accept self-signed certificate warning)
open https://192.168.64.10:4646
```

**Login:**
- Token: Use the root token from ACL bootstrap

### 6. Create ACL Policies (Optional)

Create policies for different user roles:

```bash
# Create developer policy
cat > developer-policy.hcl <<EOF
namespace "default" {
  policy = "write"
  capabilities = ["submit-job", "read-logs", "read-fs"]
}

node {
  policy = "read"
}
EOF

# Apply policy
nomad acl policy apply developer developer-policy.hcl

# Create token for developer
nomad acl token create -name="developer-token" -policy=developer
```

## Verification

### Test Job Deployment

Deploy a simple test job:

```bash
# Create test job
cat > test.nomad.hcl <<EOF
job "test-nginx" {
  datacenters = ["lcy1"]
  
  group "web" {
    count = 2
    
    network {
      port "http" {
        to = 80
      }
    }
    
    task "nginx" {
      driver = "docker"
      
      config {
        image = "nginx:alpine"
        ports = ["http"]
      }
      
      resources {
        cpu    = 100
        memory = 128
      }
    }
  }
}
EOF

# Run job
nomad job run test.nomad.hcl

# Check status
nomad job status test-nginx

# Check allocations
nomad alloc status <alloc-id>

# Stop job
nomad job stop test-nginx
```

### Test Service Discovery (with Consul)

If deployed with Consul:

```bash
# Create job with service registration
cat > web-service.nomad.hcl <<EOF
job "web" {
  datacenters = ["lcy1"]
  
  group "app" {
    network {
      port "http" { to = 8080 }
    }
    
    service {
      name     = "web"
      port     = "http"
      provider = "consul"
      
      check {
        type     = "http"
        path     = "/"
        interval = "10s"
        timeout  = "2s"
      }
    }
    
    task "nginx" {
      driver = "docker"
      config {
        image = "nginx:alpine"
        ports = ["http"]
      }
    }
  }
}
EOF

# Run job
nomad job run web-service.nomad.hcl

# Query Consul DNS
dig @192.168.64.5 -p 8600 web.service.consul

# Query via dnsmasq (from client node)
ssh 192.168.64.13 "dig web.service.consul"
```

### Test Vault Integration (with Vault)

If deployed with Vault:

```bash
# Create Vault policy
vault policy write app-policy - <<EOF
path "secret/data/app/*" {
  capabilities = ["read"]
}
EOF

# Create secret
vault kv put secret/app/config password=supersecret

# Create job that uses Vault
cat > vault-job.nomad.hcl <<EOF
job "vault-test" {
  datacenters = ["lcy1"]
  
  group "app" {
    task "test" {
      driver = "docker"
      
      vault {
        policies = ["app-policy"]
      }
      
      template {
        data = <<EOH
{{ with secret "secret/data/app/config" }}
PASSWORD={{ .Data.data.password }}
{{ end }}
EOH
        destination = "secrets/config.env"
        env = true
      }
      
      config {
        image = "alpine:latest"
        command = "sh"
        args = ["-c", "echo $PASSWORD && sleep 3600"]
      }
    }
  }
}
EOF

# Run job
nomad job run vault-job.nomad.hcl

# Check logs to verify secret was retrieved
nomad alloc logs <alloc-id>
```

## Troubleshooting

### Preflight Check Failures

**Error:** `error: missing 'ansible-playbook'`

**Solution:**
```bash
# Install Ansible
pip3 install ansible
# or
brew install ansible
```

**Error:** `error: missing 'multipass'`

**Solution:**
```bash
# macOS
brew install multipass

# Linux
sudo snap install multipass
```

### Terraform Apply Failures

**Error:** `Error creating instance: multipass failed`

**Solution:**
```bash
# Check Multipass status
multipass version
multipass list

# Restart Multipass (macOS)
sudo launchctl stop com.canonical.multipassd
sudo launchctl start com.canonical.multipassd

# Restart Multipass (Linux)
sudo snap restart multipass
```

**Error:** `Error: Failed to read SSH key`

**Solution:**
```bash
# Verify SSH key exists
ls -la ~/.ssh/id_rsa.pub

# If not, create one
ssh-keygen -t rsa -b 4096
```

### Ansible Connection Failures

**Error:** `UNREACHABLE! => {"changed": false, "msg": "Failed to connect"}`

**Solution:**
```bash
# Test SSH connectivity
ssh ubuntu@<vm-ip>

# Check VM is running
multipass list

# Check VM network
multipass exec <vm-name> -- ip addr

# Restart VM
multipass restart <vm-name>
```

### Cluster Not Forming

**Error:** Servers not joining cluster

**Solution:**
```bash
# SSH to server
ssh <server-ip>

# Check Nomad logs
sudo journalctl -u nomad -f

# Check Nomad status
sudo systemctl status nomad

# Verify TLS certificates
ls -la /etc/nomad.d/tls/

# Test connectivity to other servers
telnet <other-server-ip> 4647
```

### ACL Bootstrap Fails

**Error:** `ACL bootstrap already done`

**Solution:**
The cluster was already bootstrapped. Check for existing token:

```bash
# Look for token in generated file
cat generated_nomad_root_bootstrap_token

# Or check Nomad data directory on server
ssh <server-ip> "sudo cat /opt/nomad/data/server/acl-bootstrap-token"
```

### Vault Initialization Fails

**Error:** `Vault is already initialized`

**Solution:**
```bash
# Check for existing init file
cat vault-single-machine/generated_vault_init.json

# Get root token
cat vault-single-machine/generated_vault_init.json | jq -r .root_token

# Check unseal status
vault status
```

### Out of Resources

**Error:** VMs fail to start due to insufficient resources

**Solution:**
```bash
# Check host resources
multipass info <vm-name>

# Reduce VM resources in main.tf
# Edit instance_cpus and instance_memory

# Or destroy some VMs
multipass delete <vm-name>
multipass purge
```

### DNS Resolution Issues

**Error:** Cannot resolve `.nomad` or `.consul` domains

**Solution:**
```bash
# SSH to client
ssh <client-ip>

# Check dnsmasq status
sudo systemctl status dnsmasq

# Check dnsmasq configuration
cat /etc/dnsmasq.d/nomad.conf

# Test DNS resolution
dig @localhost web.service.nomad
dig @localhost web.service.consul

# Restart dnsmasq
sudo systemctl restart dnsmasq
```

## Cleanup

### Destroy Full Stack

```bash
cd lab/multipass
just hashistack_down
```

This will:
1. Destroy Nomad cluster
2. Destroy Vault server
3. Destroy Consul server
4. Remove `.envrc` file

### Destroy Standalone Nomad

```bash
cd lab/multipass/nomad-cluster
terraform destroy
```

### Manual Cleanup

```bash
# List all VMs
multipass list

# Delete specific VMs
multipass delete nomad-server-0 nomad-server-1 nomad-server-2
multipass delete nomad-client-0 nomad-client-1
multipass delete consul-server-0
multipass delete vault-server-0

# Purge deleted VMs
multipass purge

# Clean Terraform state
cd lab/multipass
find . -name "terraform.tfstate*" -delete
find . -name ".terraform" -type d -exec rm -rf {} +
```

## Next Steps

After successful deployment:

1. **Explore the UI**: Access Nomad UI at `https://<server-ip>:4646`
2. **Deploy Jobs**: Try example jobs from [`lab/shared/jobs/`](../../shared/jobs/)
3. **Configure Monitoring**: Deploy Prometheus and Grafana
4. **Test Failover**: Stop a server and watch leader election
5. **Scale Workloads**: Increase job counts and test autoscaling

## Related Documentation

- [Nomad Cluster README](README.md) - Detailed architecture and configuration
- [Multipass README](../README.md) - Overview of all available configurations
- [Shared Terraform Modules](../../shared/terraform/README.md) - Reusable infrastructure modules
- [Nomad Documentation](https://www.nomadproject.io/docs)
- [Just Documentation](https://just.systems/man/en/)