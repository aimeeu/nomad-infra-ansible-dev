# Nomad Cluster - Production-Like Multi-Node Deployment

A production-like Nomad cluster with 3 servers and 2 clients, featuring TLS encryption, ACLs, and optional Consul/Vault integration. Ideal for testing high-availability scenarios, service mesh integration, and container orchestration.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Directory Structure](#directory-structure)
- [Infrastructure Components](#infrastructure-components)
- [Ansible Playbooks Explained](#ansible-playbooks-explained)
- [Configuration Templates](#configuration-templates)
- [Deployment Guide](#deployment-guide)
- [Nomad and Consul Integration](#nomad-and-consul-integration)
- [Post-Deployment Configuration](#post-deployment-configuration)
- [Usage Examples](#usage-examples)
- [Troubleshooting](#troubleshooting)

## Architecture Overview

### Cluster Topology

```
┌─────────────────────────────────────────────────────────────┐
│                     Nomad Cluster (lcy1)                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Nomad Server │  │ Nomad Server │  │ Nomad Server │      │
│  │   (Leader)   │  │  (Follower)  │  │  (Follower)  │      │
│  │              │  │              │  │              │      │
│  │ 2 CPU        │  │ 2 CPU        │  │ 2 CPU        │      │
│  │ 4GiB RAM     │  │ 4GiB RAM     │  │ 4GiB RAM     │      │
│  │              │  │              │  │              │      │
│  │ Consul Agent │  │ Consul Agent │  │ Consul Agent │      │
│  │ (optional)   │  │ (optional)   │  │ (optional)   │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                 │                 │               │
│         └─────────────────┼─────────────────┘               │
│                           │ Raft Consensus                  │
│         ┌─────────────────┼─────────────────┐               │
│         │                 │                 │               │
│  ┌──────▼───────┐  ┌──────▼───────┐                        │
│  │ Nomad Client │  │ Nomad Client │                        │
│  │              │  │              │                        │
│  │ Default CPU  │  │ Default CPU  │                        │
│  │ 4GiB RAM     │  │ 4GiB RAM     │                        │
│  │              │  │              │                        │
│  │ Docker       │  │ Docker       │                        │
│  │ Podman       │  │ Podman       │                        │
│  │ CNI/Smuggle  │  │ CNI/Smuggle  │                        │
│  │ dnsmasq      │  │ dnsmasq      │                        │
│  │ Consul Agent │  │ Consul Agent │                        │
│  │ (optional)   │  │ (optional)   │                        │
│  └──────────────┘  └──────────────┘                        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Key Features

- **High Availability**: 3-server Raft consensus cluster
- **Security**: TLS encryption + ACLs enabled
- **Observability**: Prometheus metrics, debug logging
- **Container Runtimes**: Docker + Podman drivers
- **Networking**: CNI with Smuggle (VXLAN overlay)
- **DNS**: dnsmasq → CoreDNS integration
- **Service Discovery**: Optional Consul integration
- **Secrets Management**: Optional Vault workload identity
- **Storage**: WAL-based Raft log store (64MB segments)

### Region Configuration

- **Region**: `lcy1` (London)
- **Datacenter**: `lcy1` (when using Consul)
- **UI Label**: "local lcy1" (purple background)

## Directory Structure

```
nomad-cluster/
├── main.tf                          # Terraform infrastructure definition
├── ansible.cfg                      # Ansible configuration
├── inventory.yaml                   # Generated Ansible inventory (by Terraform)
├── playbook_all.yaml               # Master orchestration playbook
├── playbook_nomad_server.yaml      # Server configuration playbook
├── playbook_nomad_client.yaml      # Client configuration playbook
├── playbook_localhost.yaml         # Local environment setup playbook
├── templates/                       # Jinja2 configuration templates
│   ├── nomad_server.hcl.j2         # Nomad server configuration
│   ├── nomad_client.hcl.j2         # Nomad client configuration
│   └── consul_client.hcl.j2        # Consul client configuration
├── workload-identity/              # Vault workload identity setup scripts
├── .tls/                           # Generated TLS certificates (by Terraform)
│   ├── ca.pem                      # CA certificate
│   ├── ca-key.pem                  # CA private key
│   ├── nomad-server-*.pem          # Server certificates
│   ├── nomad-server-*-key.pem      # Server private keys
│   ├── nomad-client-*.pem          # Client certificates
│   └── nomad-client-*-key.pem      # Client private keys
├── .envrc                          # Generated environment variables (by Ansible)
└── generated_nomad_root_bootstrap_token  # Placeholder token file (by Ansible)
```

## Infrastructure Components

### Complete File Explanations

#### [`main.tf`](main.tf) - Terraform Infrastructure

Provisions the complete infrastructure using reusable modules.

**Variables**:
- `consul_address`: Optional Consul server address (format: `IP:8500`)
- `vault_address`: Optional Vault server address (format: `https://IP:8200`)

**Modules**:
1. **`ca`**: Creates TLS Certificate Authority
2. **`nomad_server`**: Provisions 3 server VMs
3. **`nomad_client`**: Provisions 2 client VMs
4. **`ansible_provision`**: Runs Ansible playbooks

**Outputs**: SSH commands, API endpoints, CA certificate

#### [`ansible.cfg`](ansible.cfg) - Ansible Settings

```ini
[defaults]
roles_path=../../shared/ansible/roles  # Shared roles location
host_key_checking=False                # Skip SSH host key verification
interpreter_python=auto_silent         # Auto-detect Python
```

#### `inventory.yaml` - Dynamic Inventory

Generated by Terraform. Contains VM IPs and Ansible groups (`nomad_server`, `nomad_client`).

#### [`.envrc`](.envrc) - Environment Variables

Generated by [`playbook_localhost.yaml`](playbook_localhost.yaml). Contains:
- `NOMAD_TOKEN`: ACL token (update after bootstrap)
- `NOMAD_ADDR`: Server API endpoint
- `NOMAD_CACERT`: CA certificate path
- `NOMAD_CLIENT_CERT`: Client certificate path
- `NOMAD_CLIENT_KEY`: Client key path

**Usage**: `source .envrc` before using Nomad CLI

#### `.tls/` - TLS Certificates

Generated by Terraform and Ansible:
- `ca.pem`, `ca-key.pem`: Certificate Authority
- `nomad-server-*.pem`: Server certificates (DNS: `server.lcy1.nomad`)
- `nomad-client-*.pem`: Client certificates (DNS: `client.lcy1.nomad`)

## Ansible Playbooks Explained

### [`playbook_all.yaml`](playbook_all.yaml) - Master Orchestration

Coordinates deployment in correct order:
1. Configure servers ([`playbook_nomad_server.yaml`](playbook_nomad_server.yaml))
2. Configure clients ([`playbook_nomad_client.yaml`](playbook_nomad_client.yaml))
3. Setup local environment ([`playbook_localhost.yaml`](playbook_localhost.yaml))

### [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml) - Server Configuration

**Roles Applied** (in order):

1. **common** - Base system setup
   - Sets hostname
   - Installs: build-essential, git, jq, make, net-tools, unzip

2. **gantsign.golang** - Go compiler
   - Version: 1.25.5
   - GOPATH: `/home/<user>/go`

3. **tls** - TLS certificates
   - Generates server certificates
   - DNS: `server.lcy1.nomad`
   - IP: VM's primary IP

4. **consul** (optional) - Service discovery
   - Version: 1.22.6
   - Mode: Client
   - Joins specified Consul server

5. **nomad** - Nomad server
   - Version: 2.0.0-beta.1
   - Mode: Server with Raft consensus
   - TLS enabled, ACLs enabled

### [`playbook_nomad_client.yaml`](playbook_nomad_client.yaml) - Client Configuration

**Roles Applied** (in order):

1. **dnsmasq** - DNS forwarding
   - Forwards `.nomad` to CoreDNS (port 1053)
   - Upstream: 1.1.1.1, 8.8.8.8, 8.8.4.4

2. **common** - Base system + Podman
   - Installs: build-essential, git, jq, make, podman, net-tools, unzip

3. **gantsign.golang** - Go compiler

4. **cni** - Container networking
   - Version: 1.9.0
   - Creates `vxlan.conf` for Smuggle

5. **smuggle** - VXLAN overlay networking

6. **geerlingguy.docker** - Docker runtime
   - Version: containerd.io 2.2.0
   - Adds user to docker group

7. **tls** - Client certificates
   - DNS: `client.lcy1.nomad`

8. **helper** - Kernel modules
   - Loads `bridge` module

9. **consul** (optional) - Service discovery client

10. **nomad** - Nomad client
    - Version: 2.0.0-beta.1
    - Drivers: raw_exec, docker, podman
    - Plugin: nomad-driver-podman 0.6.4

### [`playbook_localhost.yaml`](playbook_localhost.yaml) - Local Environment

**What it does**:
1. Creates `generated_nomad_root_bootstrap_token` with placeholder
2. Creates `.envrc` with environment variables for CLI access

## Configuration Templates

### [`templates/nomad_server.hcl.j2`](templates/nomad_server.hcl.j2)

**Key Sections**:
- **Data/Binding**: Data dir, bind address, region
- **Logging**: DEBUG level with location tracking
- **Telemetry**: Prometheus metrics enabled
- **Server**: Raft consensus, bootstrap expect 3, WAL log store
- **ACL**: Enabled (requires bootstrap)
- **Consul**: Optional integration
- **UI**: Enabled with purple "local lcy1" label
- **Vault**: Optional workload identity

### [`templates/nomad_client.hcl.j2`](templates/nomad_client.hcl.j2)

**Key Sections**:
- **Data/Binding**: Data dir, bind address, region, plugin dir
- **Logging**: DEBUG level
- **Telemetry**: Prometheus metrics
- **Client**: Enabled, retry join servers
- **ACL**: Enabled
- **Consul**: Optional integration
- **Drivers**: raw_exec enabled
- **Vault**: Optional JWT auth

### [`templates/consul_client.hcl.j2`](templates/consul_client.hcl.j2)

**Key Sections**:
- **Mode**: Client (not server)
- **Datacenter**: lcy1
- **Retry Join**: Connects to specified Consul server
- **Connect**: Service mesh enabled

## Deployment Guide

### Prerequisites

```bash
# Install tools
brew install multipass terraform  # macOS
pip3 install ansible

# Verify SSH key
ls ~/.ssh/id_rsa.pub
```

### Scenario 1: Standalone Nomad

```bash
cd lab/multipass/nomad-cluster
terraform init
terraform apply
```

### Scenario 2: Nomad + Consul

```bash
# Deploy Consul first
cd ../consul-single-machine
terraform apply
# Note consul_server_address output

# Deploy Nomad with Consul
cd ../nomad-cluster
terraform apply -var="consul_address=192.168.64.5:8500"
```

### Scenario 3: Full Stack (Nomad + Consul + Vault)

```bash
# Deploy Consul
cd ../consul-single-machine
terraform apply

# Deploy Vault
cd ../vault-single-machine
terraform apply

# Initialize Vault
ssh <vault-ip>
vault operator init -key-shares=1 -key-threshold=1
vault operator unseal <key>

# Deploy Nomad
cd ../nomad-cluster
terraform apply \
  -var="consul_address=<consul-ip>:8500" \
  -var="vault_address=https://<vault-ip>:8200"
```

## Nomad and Consul Integration

### Service Registration Flow

```
Nomad Job → Consul Agent → Consul Server → DNS/API
```

### Example with Service Discovery

```hcl
job "web" {
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
        path     = "/health"
        interval = "10s"
      }
    }
    
    task "server" {
      driver = "docker"
      config {
        image = "nginx:alpine"
        ports = ["http"]
      }
    }
  }
}
```

### DNS Resolution

```bash
# Query Consul DNS
dig @localhost -p 8600 web.service.consul

# Query via dnsmasq
dig web.service.consul
```

## Post-Deployment Configuration

### 1. Bootstrap ACLs

```bash
ssh $(terraform output -json nomad_server_addresses | jq -r '.[0]')
nomad acl bootstrap
# Save the Secret ID!
```

### 2. Configure Local CLI

```bash
# Update .envrc with real token
vim .envrc
source .envrc

# Verify
nomad server members
nomad node status
```

### 3. Create ACL Policies

```hcl
# developer-policy.hcl
namespace "default" {
  policy = "write"
}
```

```bash
nomad acl policy apply developer developer-policy.hcl
nomad acl token create -name="dev" -policy=developer
```

## Usage Examples

### Simple Web Service

```hcl
job "nginx" {
  datacenters = ["lcy1"]
  
  group "web" {
    count = 3
    
    network {
      port "http" { to = 80 }
    }
    
    task "server" {
      driver = "docker"
      config {
        image = "nginx:alpine"
        ports = ["http"]
      }
    }
  }
}
```

### Using Podman

```hcl
task "app" {
  driver = "podman"
  config {
    image = "docker.io/library/redis:alpine"
  }
}
```

### With Vault Secrets

```hcl
task "app" {
  vault {
    policies = ["app-policy"]
  }
  
  template {
    data = <<EOH
{{ with secret "secret/data/app" }}
DB_PASS={{ .Data.data.password }}
{{ end }}
EOH
    destination = "secrets/config.env"
    env = true
  }
}
```

## Troubleshooting

### VMs Not Starting

```bash
multipass list
multipass logs nomad-server-*
sudo launchctl restart com.canonical.multipassd
```

### Cluster Not Forming

```bash
# Check logs
ssh <server-ip> "sudo journalctl -u nomad -f"

# Verify TLS
openssl x509 -in .tls/nomad-server-0.pem -text -noout

# Test connectivity
ssh <server-ip> "telnet <other-server-ip> 4647"
```

### Client Not Connecting

```bash
# Check status
ssh <client-ip> "sudo systemctl status nomad"

# Check logs
ssh <client-ip> "sudo journalctl -u nomad -f"

# Verify can reach servers
ssh <client-ip> "telnet <server-ip> 4647"
```

### Docker/Podman Issues

```bash
ssh <client-ip> "docker ps"
ssh <client-ip> "podman ps"
ssh <client-ip> "sudo systemctl status docker"
```

### DNS Not Working

```bash
ssh <client-ip> "dig @localhost web.service.nomad"
ssh <client-ip> "sudo systemctl status dnsmasq"
```

## Cleanup

```bash
terraform destroy
# or
multipass delete nomad-*
multipass purge
```

## Related Configurations

- [`consul-single-machine`](../consul-single-machine/): Standalone Consul server
- [`vault-single-machine`](../vault-single-machine/): Standalone Vault server
- [`nomad-dev-cluster`](../nomad-dev-cluster/): Simplified development cluster