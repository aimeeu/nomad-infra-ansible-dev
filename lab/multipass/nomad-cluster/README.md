# Nomad Cluster

A production-like Nomad cluster deployment with 3 servers and 2 clients using Multipass VMs. Includes optional Consul and Vault integration.

## Overview

This configuration deploys a complete Nomad cluster suitable for:
- Production-like testing and validation
- Multi-node cluster behavior testing
- High availability scenarios
- Service mesh integration with Consul Connect
- Secrets management with Vault workload identity
- Container orchestration with Docker and Podman

## Architecture

- **3 Nomad Servers**: Raft consensus cluster with WAL log store
- **2 Nomad Clients**: Worker nodes with Docker and Podman drivers
- **VM Resources**: 
  - Servers: 2 CPUs, 4GiB RAM each
  - Clients: Default CPUs, 4GiB RAM each
- **Region**: lcy1 (London)
- **Features**:
  - TLS encryption for all communication
  - ACLs enabled for security
  - Prometheus metrics enabled
  - Nomad UI with custom label
  - Optional Consul integration for service discovery
  - Optional Vault integration for secrets
  - CNI networking with Smuggle (VXLAN overlay)
  - DNS resolution via dnsmasq → CoreDNS

## What the Terraform Does

[`main.tf`](main.tf:1) provisions the infrastructure:

1. **Generates TLS Certificates** (via `tls` module):
   - Creates CA certificate
   - Stores in `.tls/` directory
   - Used for secure Nomad communication

2. **Creates Nomad Server VMs** (via `multipass-compute` module):
   - 3 instances: `nomad-server-<random-id>`
   - Specs: 2 CPUs, 4GiB RAM each
   - Ansible group: `nomad_server`

3. **Creates Nomad Client VMs** (via `multipass-compute` module):
   - 2 instances: `nomad-client-<random-id>`
   - Specs: Default CPUs, 4GiB RAM each
   - Ansible group: `nomad_client`

4. **Runs Ansible Provisioning** (via `ansible-provision` module):
   - Executes [`playbook_all.yaml`](playbook_all.yaml:1)
   - Configures servers and clients
   - Optional: Passes `consul_address` and `vault_address` variables

5. **Outputs**:
   - SSH commands for all nodes
   - Rsync commands for code deployment
   - Nomad HTTPS API endpoints
   - CA certificate (for API access)
   - Server IP addresses

## What the Ansible Playbooks Do

### [`playbook_all.yaml`](playbook_all.yaml:1)
Master playbook that orchestrates the deployment:
- Imports [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:1)
- Imports [`playbook_nomad_client.yaml`](playbook_nomad_client.yaml:1)
- Imports `playbook_localhost.yaml` (local configuration)

### [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:1)
Configures Nomad server nodes:

1. **Fact Gathering**: Delegates fact collection for controlled restarts

2. **Common Role**: Base system setup
   - Sets hostname
   - Installs: build-essential, git, jq, make, net-tools, unzip

3. **Golang Role**: Installs Go 1.25.5
   - For building Nomad from source if needed
   - GOPATH: `/home/<user>/go`

4. **TLS Role**: Generates server certificates
   - Agent-specific certificates
   - DNS: `server.lcy1.nomad`
   - IP: VM's default IPv4 address

5. **Consul Role** (optional, when `consul_address` provided):
   - Version: 1.22.6
   - Client mode
   - Config: [`consul_client.hcl.j2`](templates/consul_client.hcl.j2:1)
   - Joins specified Consul server

6. **Nomad Role**: Installs and configures Nomad
   - Version: 2.0.0-beta.1
   - Config: [`nomad_server.hcl.j2`](templates/nomad_server.hcl.j2:1)
   - TLS enabled with generated certificates

### [`playbook_nomad_client.yaml`](playbook_nomad_client.yaml:1)
Configures Nomad client nodes:

1. **Network Configuration**: Removes Ubuntu default network link file
   - Fixes issues with Smuggle CNI plugin

2. **dnsmasq Role**: DNS forwarding setup
   - Forwards `.nomad` queries to CoreDNS (port 1053)
   - Listens on localhost and VM IP
   - Upstream DNS: 1.1.1.1, 8.8.8.8, 8.8.4.4

3. **Common Role**: Base system setup
   - Installs: build-essential, git, jq, make, podman, net-tools, unzip

4. **Golang Role**: Installs Go 1.25.5

5. **CNI Role**: Configures Container Network Interface
   - Creates `vxlan.conf` for Smuggle plugin

6. **Smuggle Role**: Installs Smuggle CNI plugin
   - VXLAN overlay networking for containers

7. **Docker Role**: Installs Docker
   - Version: containerd.io 2.2.0
   - Adds user to docker group

8. **TLS Role**: Generates client certificates
   - DNS: `client.lcy1.nomad`

9. **Helper Role**: Kernel module configuration
   - Loads `bridge` module for networking

10. **Consul Role** (optional): Client mode configuration

11. **Nomad Role**: Installs Nomad with plugins
    - Version: 2.0.0-beta.1
    - Config: [`nomad_client.hcl.j2`](templates/nomad_client.hcl.j2:1)
    - Plugins: nomad-driver-podman 0.6.4
    - TLS enabled

## Configuration Details

### Nomad Server ([`nomad_server.hcl.j2`](templates/nomad_server.hcl.j2:1))

Key settings:
- **Region**: lcy1
- **Log Level**: DEBUG with location tracking
- **Telemetry**: Prometheus metrics enabled
- **Server Mode**: 
  - Bootstrap expect: 3 servers
  - Retry join: All server IPs
  - WAL log store (64MB segments)
- **ACLs**: Enabled
- **UI**: Enabled with "local lcy1" purple label
- **Consul Integration** (optional): Connects to local Consul agent
- **Vault Integration** (optional): 
  - Workload identity enabled
  - JWT auth with 1h TTL
  - Audience: vault.io

### Nomad Client ([`nomad_client.hcl.j2`](templates/nomad_client.hcl.j2:1))

Key settings:
- **Region**: lcy1
- **Log Level**: DEBUG
- **Telemetry**: Prometheus metrics enabled
- **Client Mode**: Enabled
- **Server Join**: Retry join all server IPs
- **ACLs**: Enabled
- **Plugins**:
  - raw_exec: Enabled
  - docker: Enabled (via Docker installation)
  - podman: Enabled (via plugin)
- **Consul Integration** (optional): Local agent connection
- **Vault Integration** (optional): JWT auth backend

### Consul Client ([`consul_client.hcl.j2`](templates/consul_client.hcl.j2:1))

When Consul integration is enabled:
- **Mode**: Client (not server)
- **Datacenter**: lcy1
- **Retry Join**: Connects to provided Consul server address
- **Connect**: Service mesh enabled

## Prerequisites

- **Multipass**: Install from [multipass.run](https://multipass.run)
- **Terraform**: v1.0 or later
- **Ansible**: v2.9 or later
- **SSH Key**: `~/.ssh/id_rsa.pub` must exist
- **Optional**: Running Consul server (for integration)
- **Optional**: Running Vault server (for integration)

## Deployment Instructions

### 1. Basic Deployment (Nomad Only)

```bash
cd lab/multipass/nomad-cluster
terraform init
terraform apply
```

### 2. Deployment with Consul Integration

First, deploy a Consul server:
```bash
cd lab/multipass/consul-single-machine
terraform init
terraform apply
# Note the consul_server_address output
```

Then deploy Nomad cluster:
```bash
cd lab/multipass/nomad-cluster
terraform init
terraform apply -var="consul_address=<consul-server-ip>:8500"
```

### 3. Deployment with Vault Integration

Deploy Vault server, then:
```bash
terraform apply \
  -var="consul_address=<consul-ip>:8500" \
  -var="vault_address=https://<vault-ip>:8200"
```

### 4. Access the Cluster

After deployment, Terraform outputs connection details:

```
SSH commands:
  Nomad Servers:
    - ssh 192.168.64.X
    - ssh 192.168.64.Y
    - ssh 192.168.64.Z
  Nomad Client:
    - ssh 192.168.64.A
    - ssh 192.168.64.B

Nomad HTTP API:
    - https://192.168.64.X:4646
    - https://192.168.64.Y:4646
    - https://192.168.64.Z:4646
```

### 5. Bootstrap ACLs

SSH into any server and bootstrap:
```bash
ssh 192.168.64.X
nomad acl bootstrap
```

Save the Secret ID (management token).

### 6. Configure Local Nomad CLI

Export the CA certificate:
```bash
terraform output -raw nomad_ca_cert_pem > nomad-ca.pem
```

Set environment variables:
```bash
export NOMAD_ADDR=https://192.168.64.X:4646
export NOMAD_CACERT=nomad-ca.pem
export NOMAD_TOKEN=<management-token-from-bootstrap>
```

### 7. Verify Cluster

```bash
nomad server members
nomad node status
```

Expected output:
```
# Server members
Name                    Address       Port  Status  Leader  Raft Version
nomad-server-abc123.lcy1  192.168.64.X  4648  alive   true    3

# Node status
ID        DC    Name                 Class   Drain  Eligibility  Status
abc123    lcy1  nomad-client-xyz     <none>  false  eligible     ready
def456    lcy1  nomad-client-uvw     <none>  false  eligible     ready
```

## Usage Examples

### Deploy a Job

Create `example.nomad`:
```hcl
job "example" {
  datacenters = ["lcy1"]
  
  group "web" {
    count = 3
    
    network {
      port "http" {
        to = 8080
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

Deploy:
```bash
nomad job run example.nomad
nomad job status example
```

### Access Nomad UI

Open browser to any server IP:
```
https://192.168.64.X:4646
```

Accept the self-signed certificate warning, then enter your management token.

### Deploy with Podman Driver

```hcl
task "app" {
  driver = "podman"
  
  config {
    image = "docker.io/library/nginx:alpine"
    ports = ["http"]
  }
}
```

### Use Service Discovery (with Consul)

```hcl
service {
  name = "web"
  port = "http"
  provider = "consul"
  
  check {
    type     = "http"
    path     = "/health"
    interval = "10s"
    timeout  = "2s"
  }
}
```

### Use Vault Secrets (with Vault)

```hcl
task "app" {
  vault {
    policies = ["app-policy"]
  }
  
  template {
    data = <<EOH
{{ with secret "secret/data/app" }}
DB_PASSWORD={{ .Data.data.password }}
{{ end }}
EOH
    destination = "secrets/config.env"
    env = true
  }
}
```

## Advanced Operations

### Rolling Updates

Update servers one at a time:
```bash
# Update server 1
ansible-playbook playbook_nomad_server.yaml --limit nomad-server-0

# Verify cluster health
nomad server members

# Repeat for other servers
```

### Scale Clients

Modify [`main.tf`](main.tf:22):
```hcl
instance_count = 5  # Increase from 2 to 5
```

Then:
```bash
terraform apply
```

### Deploy Monitoring Stack

```bash
# From the nomad-cluster directory
nomad job run ../../shared/jobs/monitoring/prometheus-server.nomad.hcl
nomad job run ../../shared/jobs/monitoring/grafana-server.nomad.hcl
```

### Rsync Code for Development

Use the rsync commands from Terraform output to sync local Nomad source:
```bash
rsync -r --exclude 'nomad/ui/node_modules/*' \
  /path/to/nomad \
  ubuntu@192.168.64.X:/home/ubuntu/
```

## Cleanup

### Destroy Infrastructure

```bash
terraform destroy
```

Type `yes` when prompted.

### Manual VM Cleanup (if needed)

```bash
multipass list
multipass delete nomad-server-* nomad-client-*
multipass purge
```

## Configuration Files

- [`main.tf`](main.tf:1): Terraform infrastructure definition
- [`ansible.cfg`](ansible.cfg:1): Ansible configuration
- [`inventory.yaml`](inventory.yaml:1): Generated Ansible inventory
- [`playbook_all.yaml`](playbook_all.yaml:1): Master playbook
- [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:1): Server configuration
- [`playbook_nomad_client.yaml`](playbook_nomad_client.yaml:1): Client configuration
- [`templates/nomad_server.hcl.j2`](templates/nomad_server.hcl.j2:1): Server config template
- [`templates/nomad_client.hcl.j2`](templates/nomad_client.hcl.j2:1): Client config template
- [`templates/consul_client.hcl.j2`](templates/consul_client.hcl.j2:1): Consul client template

## Troubleshooting

### VMs Not Starting
```bash
multipass list
multipass info nomad-server-*
multipass logs nomad-server-*
```

### Nomad Server Not Joining Cluster
```bash
ssh 192.168.64.X "sudo journalctl -u nomad -f"
ssh 192.168.64.X "nomad server members"
```

### Nomad Client Not Connecting
```bash
ssh 192.168.64.A "sudo systemctl status nomad"
ssh 192.168.64.A "nomad node status"
```

### TLS Certificate Issues
```bash
# Verify certificates exist
ls -la .tls/

# Check certificate validity
openssl x509 -in .tls/nomad-server-0.pem -text -noout
```

### Docker/Podman Issues
```bash
ssh 192.168.64.A "docker ps"
ssh 192.168.64.A "podman ps"
ssh 192.168.64.A "sudo systemctl status docker"
```

### DNS Resolution Issues
```bash
ssh 192.168.64.A "dig @localhost web.service.nomad"
ssh 192.168.64.A "sudo systemctl status dnsmasq"
```

## Performance Tuning

### Increase VM Resources

Modify [`main.tf`](main.tf:1):
```hcl
module "nomad_server" {
  instance_cpus   = 4      # Increase from 2
  instance_memory = "8GiB" # Increase from 4GiB
}
```

### Optimize Raft Performance

Edit [`nomad_server.hcl.j2`](templates/nomad_server.hcl.j2:28):
```hcl
raft_logstore {
  backend = "wal"
  wal {
    segment_size_mb = 128  # Increase from 64
  }
}
```

## Next Steps

- **Deploy Workloads**: Use job specifications from `../../shared/jobs/`
- **Enable Monitoring**: Deploy Prometheus and Grafana
- **Configure Autoscaling**: Deploy Nomad Autoscaler
- **Multi-Region**: Deploy additional clusters in different regions
- **Production Hardening**: Implement proper ACL policies and TLS certificate management

## Related Configurations

- [`consul-single-machine`](../consul-single-machine/): Standalone Consul server
- [`vault-single-machine`](../vault-single-machine/): Standalone Vault server
- [`nomad-dev-cluster`](../nomad-dev-cluster/): Simplified development cluster
- [`nomad-single-machine`](../nomad-single-machine/): All-in-one Nomad instance