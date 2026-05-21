# Nomad Dev Cluster

A simplified Nomad development cluster (1 server, 1 client) optimized for Nomad source code development and debugging. No TLS, ACLs, or production features - just the essentials for rapid iteration.

## Overview

This configuration deploys a minimal Nomad cluster specifically designed for:
- **Nomad Core Development**: Building and testing Nomad from source
- **Rapid Iteration**: Quick setup/teardown cycles
- **IDE Debugging**: Compatible with debuggers like Delve
- **Feature Testing**: Isolated environment for new features
- **Learning**: Simple setup for understanding Nomad internals

## Architecture

- **1 Nomad Server**: Single server node (no HA)
- **1 Nomad Client**: Single worker node
- **VM Resources**: Default (1 CPU, 1GiB RAM per VM)
- **Security**: None (no TLS, no ACLs) - development only!
- **Features**:
  - Prometheus metrics enabled
  - Debug logging enabled
  - Docker driver
  - CNI networking
  - Git/GitHub integration configured
  - SSH keys copied for development

## What the Terraform Does

[`main.tf`](main.tf:1) provisions minimal infrastructure:

1. **Creates Nomad Server VM** (via `multipass-compute` module):
   - Instance name: `nomad-server-<random-id>`
   - Default specs: 1 CPU, 1GiB RAM
   - Ansible group: `nomad_server`

2. **Creates Nomad Client VM** (via `multipass-compute` module):
   - Instance name: `nomad-client-<random-id>`
   - Default specs: 1 CPU, 1GiB RAM
   - Ansible group: `nomad_client`

3. **Runs Ansible Provisioning** (via `ansible-provision` module):
   - Executes [`playbook_all.yaml`](playbook_all.yaml:1)
   - Configures development environment
   - No TLS certificate generation

4. **Outputs**:
   - SSH commands for both nodes
   - Rsync commands for syncing Nomad source code

## What the Ansible Playbooks Do

### [`playbook_all.yaml`](playbook_all.yaml:1)
Master playbook that orchestrates the deployment:
- Imports [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:1)
- Imports [`playbook_nomad_client.yaml`](playbook_nomad_client.yaml:1)
- Imports `playbook_localhost.yaml` (local configuration)

### [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:1)
Configures the Nomad server for development:

**Variables to Customize** (lines 4-7):
```yaml
github_email: "your-email@users.noreply.github.com"
ssh_key_private_filename: "id_rsa"
ssh_key_public_filename: "id_rsa.pub"
```

**Roles Applied**:

1. **Common Role**: Base system setup
   - Sets hostname
   - Installs: jq, net-tools, unzip

2. **Golang Role**: Installs Go 1.25.5
   - Required for building Nomad from source
   - GOPATH: `/home/<user>/go`

3. **Helper Role**: Development environment setup
   - Installs: build-essential, git, make
   - Creates Nomad config: `~/nomad-server.hcl`
   - Copies SSH keys from local machine
   - Configures Git:
     - Sets email
     - Configures SSH for GitHub (instead of HTTPS)

**Tasks**:
- Adds GitHub SSH keys to known_hosts

### [`playbook_nomad_client.yaml`](playbook_nomad_client.yaml:1)
Configures the Nomad client for development:

**Variables to Customize** (lines 4-7):
Same as server playbook

**Roles Applied**:

1. **Common Role**: Base system setup

2. **Golang Role**: Installs Go 1.25.5

3. **CNI Role**: Installs Container Network Interface plugins

4. **Docker Role**: Installs Docker
   - Adds user to docker group

5. **Helper Role**: Development environment setup
   - Installs build tools
   - Creates Nomad config: `~/nomad-client.hcl`
   - Copies SSH keys
   - Configures Git
   - Loads bridge kernel module

**Tasks**:
- Adds GitHub SSH keys to known_hosts

## Configuration Details

### Nomad Server ([`nomad_server.hcl.j2`](templates/nomad_server.hcl.j2:1))

Minimal server configuration:
```hcl
data_dir   = "/var/lib/nomad"
bind_addr  = "<vm-ip>"

log_level            = "DEBUG"
log_include_location = true
log_file             = "/var/log/nomad.log"
enable_debug         = true

telemetry {
  publish_allocation_metrics = true
  publish_node_metrics       = true
  prometheus_metrics         = true
}

server {
  enabled          = true
  bootstrap_expect = 1
  
  server_join {
    retry_join = ["<server-ip>"]
  }
}
```

**Key Points**:
- No TLS configuration
- No ACL configuration
- No UI configuration
- Debug logging enabled
- Single server bootstrap

### Nomad Client ([`nomad_client.hcl.j2`](templates/nomad_client.hcl.j2:1))

Minimal client configuration:
```hcl
data_dir   = "/var/lib/nomad"
bind_addr  = "<vm-ip>"

log_level            = "DEBUG"
log_include_location = true
log_file             = "/var/log/nomad.log"
enable_debug         = true

telemetry {
  publish_allocation_metrics = true
  publish_node_metrics       = true
  prometheus_metrics         = true
}

client {
  enabled           = true
  network_interface = "<interface>"
  
  server_join {
    retry_join = ["<server-ip>"]
  }
}
```

**Key Points**:
- No TLS configuration
- No ACL configuration
- No plugin configuration (uses defaults)
- Docker driver enabled by default

## Prerequisites

- **Multipass**: Install from [multipass.run](https://multipass.run)
- **Terraform**: v1.0 or later
- **Ansible**: v2.9 or later
- **SSH Key**: `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub` must exist
- **GitHub Access**: SSH key added to GitHub account (for cloning repos)

## Deployment Instructions

### 1. Customize Variables

Edit both playbooks to set your information:

**[`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:4-7)**:
```yaml
vars:
  github_email: "your-email@users.noreply.github.com"
  ssh_key_private_filename: "id_rsa"
  ssh_key_public_filename: "id_rsa.pub"
```

**[`playbook_nomad_client.yaml`](playbook_nomad_client.yaml:4-7)**:
```yaml
vars:
  github_email: "your-email@users.noreply.github.com"
  ssh_key_private_filename: "id_rsa"
  ssh_key_public_filename: "id_rsa.pub"
```

### 2. Deploy

```bash
cd lab/multipass/nomad-dev-cluster
terraform init
terraform apply
```

Deployment takes 3-5 minutes.

### 3. Access VMs

After deployment, Terraform outputs SSH commands:
```
SSH commands:
  Nomad Servers:
    - ssh 192.168.64.X
  Nomad Clients:
    - ssh 192.168.64.Y
```

### 4. Clone Nomad Source

SSH into the server VM:
```bash
ssh 192.168.64.X
cd ~
git clone git@github.com:hashicorp/nomad.git
cd nomad
```

### 5. Build and Run Nomad

**On Server VM**:
```bash
# Build Nomad
make bootstrap
make dev

# Run server
sudo ./bin/nomad agent -config=/home/ubuntu/nomad-server.hcl
```

**On Client VM** (in another terminal):
```bash
ssh 192.168.64.Y
cd ~/nomad
make dev

# Run client
sudo ./bin/nomad agent -config=/home/ubuntu/nomad-client.hcl
```

### 6. Verify Cluster

From your local machine:
```bash
export NOMAD_ADDR=http://192.168.64.X:4646
nomad server members
nomad node status
```

## Development Workflow

### Sync Local Changes to VMs

Use the rsync commands from Terraform output:

```bash
# Sync to server
rsync -r --exclude 'nomad/ui/node_modules/*' \
  ~/Projects/Go/nomad \
  ubuntu@192.168.64.X:/home/ubuntu/

# Sync to client
rsync -r --exclude 'nomad/ui/node_modules/*' \
  ~/Projects/Go/nomad \
  ubuntu@192.168.64.Y:/home/ubuntu/
```

### Rebuild and Restart

**After syncing changes**:
```bash
# On server VM
ssh 192.168.64.X
cd ~/nomad
make dev
sudo pkill nomad
sudo ./bin/nomad agent -config=/home/ubuntu/nomad-server.hcl
```

### Use with Debugger (Delve)

**On server VM**:
```bash
cd ~/nomad
go install github.com/go-delve/delve/cmd/dlv@latest

# Run with debugger
sudo ~/go/bin/dlv exec ./bin/nomad -- agent -config=/home/ubuntu/nomad-server.hcl
```

Connect your IDE debugger to the remote Delve session.

### Test Changes

```bash
# Deploy a test job
nomad job run example.nomad

# Check logs
nomad alloc logs <alloc-id>

# Monitor server logs
ssh 192.168.64.X "tail -f /var/log/nomad.log"
```

## Usage Examples

### Deploy a Simple Job

Create `test.nomad`:
```hcl
job "test" {
  datacenters = ["dc1"]
  
  group "web" {
    task "server" {
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
    
    network {
      port "http" {
        static = 8080
      }
    }
  }
}
```

Deploy:
```bash
nomad job run test.nomad
```

### Test Raw Exec Driver

```hcl
task "script" {
  driver = "raw_exec"
  
  config {
    command = "/bin/bash"
    args    = ["-c", "echo Hello from Nomad"]
  }
}
```

### Access Container

```bash
# Find allocation
nomad job status test

# Exec into container
nomad alloc exec <alloc-id> /bin/sh
```

## Troubleshooting

### VMs Not Starting
```bash
multipass list
multipass info nomad-server-*
```

### Nomad Not Running
```bash
ssh 192.168.64.X "ps aux | grep nomad"
ssh 192.168.64.X "tail -100 /var/log/nomad.log"
```

### Build Failures
```bash
ssh 192.168.64.X
cd ~/nomad
make clean
make bootstrap
make dev
```

### SSH Key Issues
```bash
# Verify keys exist locally
ls -la ~/.ssh/id_rsa*

# Verify keys copied to VM
ssh 192.168.64.X "ls -la ~/.ssh/"
```

### GitHub Clone Fails
```bash
# Test SSH connection
ssh -T git@github.com

# Verify SSH key added to GitHub
# https://github.com/settings/keys
```

### Docker Issues
```bash
ssh 192.168.64.Y "docker ps"
ssh 192.168.64.Y "sudo systemctl status docker"
ssh 192.168.64.Y "groups"  # Should include 'docker'
```

## Performance Tips

### Increase VM Resources

Modify [`main.tf`](main.tf:1):
```hcl
module "nomad_server" {
  instance_cpus   = 2
  instance_memory = "4GiB"
}

module "nomad_client" {
  instance_cpus   = 2
  instance_memory = "4GiB"
}
```

### Use Local Nomad Binary

Instead of building in VM, build locally and copy:
```bash
# Build locally
cd ~/Projects/Go/nomad
make dev

# Copy to VMs
scp ./bin/nomad ubuntu@192.168.64.X:/home/ubuntu/nomad/bin/
scp ./bin/nomad ubuntu@192.168.64.Y:/home/ubuntu/nomad/bin/
```

## Cleanup

### Destroy Infrastructure

```bash
terraform destroy
```

### Manual VM Cleanup

```bash
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

## Differences from Production Cluster

This dev cluster intentionally omits:
- ❌ TLS encryption
- ❌ ACL security
- ❌ High availability (3+ servers)
- ❌ Consul integration
- ❌ Vault integration
- ❌ Nomad UI
- ❌ Production-grade logging
- ❌ Monitoring/alerting
- ❌ Backup/recovery

Use [`nomad-cluster`](../nomad-cluster/) for production-like testing.

## Next Steps

- **Contribute to Nomad**: Make changes and submit PRs
- **Test Features**: Validate new functionality
- **Debug Issues**: Use Delve for deep debugging
- **Learn Internals**: Explore Nomad source code
- **Upgrade to Production**: Use [`nomad-cluster`](../nomad-cluster/) for full features

## Related Configurations

- [`nomad-cluster`](../nomad-cluster/): Production-like cluster with TLS/ACLs
- [`nomad-single-machine`](../nomad-single-machine/): All-in-one Nomad instance
- [`nomad-server-cluster`](../nomad-server-cluster/): Server-only cluster