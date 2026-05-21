# Nomad Server Cluster

A 3-node Nomad server cluster without clients, designed for testing server-only features, HA scenarios, and Raft consensus behavior.

## Overview

This configuration deploys a server-only Nomad cluster ideal for:
- Testing Nomad server features without client workloads
- Raft consensus and leader election testing
- Server upgrade procedures
- High availability scenarios
- Server performance benchmarking

## Architecture

- **3 Nomad Servers**: Raft consensus cluster with TLS
- **0 Nomad Clients**: No worker nodes
- **VM Resources**: 4GiB RAM per server
- **Region**: lcy1 (London)
- **Features**:
  - TLS encryption enabled
  - ACLs enabled
  - WAL log store
  - Prometheus metrics
  - Nomad UI

## What the Terraform Does

[`main.tf`](main.tf:1) provisions server infrastructure:

1. **Creates 3 Nomad Server VMs** (via `multipass-compute` module):
   - Instances: `nomad-server-<random-id>` (x3)
   - Specs: Default CPUs, 4GiB RAM each
   - Ansible group: `nomad_server`

2. **Runs Ansible Provisioning** (via `ansible-provision` module):
   - Executes [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:1)
   - Configures all 3 servers

3. **Runs Localhost Provisioning**:
   - Executes `playbook_localhost.yaml`
   - Local environment setup

4. **Outputs**:
   - SSH commands for all servers
   - Rsync commands for code deployment

## What the Ansible Playbook Does

### [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:1)

Configures Nomad servers with controlled restart capability:

1. **Fact Gathering**: Delegates fact collection for all servers

2. **Common Role**: Base system setup
   - Sets hostname
   - Installs: build-essential, git, jq, make, net-tools, unzip

3. **Golang Role**: Installs Go 1.25.5
   - For building Nomad from source

4. **TLS Role**: Generates server certificates
   - Agent-specific certificates
   - DNS: `server.lcy1.nomad`
   - Stores in `.tls/` directory

5. **Nomad Role**: Installs and configures Nomad
   - Version: 2.0.0-beta.1
   - Config template: [`nomad_server.hcl.j2`](templates/nomad_server.hcl.j2:1)
   - TLS enabled with generated certificates
   - Region: lcy1

## Prerequisites

- **Multipass**: Install from [multipass.run](https://multipass.run)
- **Terraform**: v1.0 or later
- **Ansible**: v2.9 or later
- **SSH Key**: `~/.ssh/id_rsa.pub` must exist

## Deployment Instructions

### 1. Deploy

```bash
cd lab/multipass/nomad-server-cluster
terraform init
terraform apply
```

### 2. Access Servers

```bash
# SSH into any server
ssh 192.168.64.X

# Check server status
nomad server members
```

### 3. Bootstrap ACLs

```bash
ssh 192.168.64.X
nomad acl bootstrap
# Save the management token
```

### 4. Access Nomad UI

```bash
# Port forward to access UI
ssh -L 4646:192.168.64.X:4646 192.168.64.X
```

Open browser to `https://localhost:4646`

## Use Cases

### Test Leader Election

```bash
# Stop leader
ssh <leader-ip> "sudo systemctl stop nomad"

# Watch new leader election
nomad server members
```

### Test Rolling Upgrades

```bash
# Update one server at a time
ansible-playbook playbook_nomad_server.yaml --limit nomad-server-0
# Verify cluster health
nomad server members
# Repeat for other servers
```

### Test Raft Performance

```bash
# Monitor Raft metrics
curl -k https://192.168.64.X:4646/v1/metrics?format=prometheus | grep raft
```

## Cleanup

```bash
terraform destroy
```

## Related Configurations

- [`nomad-cluster`](../nomad-cluster/): Full cluster with clients
- [`nomad-single-machine`](../nomad-single-machine/): All-in-one instance