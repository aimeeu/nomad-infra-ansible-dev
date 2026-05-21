# Nomad Single Machine

An all-in-one Nomad deployment running both server and client on a single VM. Perfect for testing, demos, and learning Nomad without the complexity of a multi-node cluster.

## Overview

This configuration deploys a complete Nomad environment on one machine, ideal for:
- Quick testing and experimentation
- Learning Nomad features
- Demo environments
- Development without multi-node complexity
- Testing jobs that don't require multiple clients

## Architecture

- **1 VM**: Single machine running both server and client
- **VM Resources**: 4 CPUs, 4GiB RAM
- **Region**: lcy1 (London)
- **Features**:
  - TLS encryption enabled
  - ACLs enabled
  - Docker and Smuggle CNI
  - DNS resolution via dnsmasq
  - Separate server and client processes
  - Different Nomad versions for server (1.11.1) and client (2.0.0-beta.1)

## What the Terraform Does

[`main.tf`](main.tf:1) provisions a single powerful VM:

1. **Creates Nomad VM** (via `multipass-compute` module):
   - Instance: `nomad-<random-id>`
   - Specs: 4 CPUs, 4GiB RAM
   - Ansible group: `nomad`

2. **Runs Ansible Provisioning** (via `ansible-provision` module):
   - Executes [`playbook_all.yaml`](playbook_all.yaml:1)
   - Configures both server and client

3. **Outputs**:
   - SSH command
   - Port forwarding command for Nomad UI access

## What the Ansible Playbook Does

### [`playbook_nomad.yaml`](playbook_nomad.yaml:1)

Configures a complete Nomad environment on one machine:

1. **Common Role**: Base system setup
   - Installs: build-essential, git, jq, make, net-tools, unzip

2. **dnsmasq Role**: DNS forwarding
   - Forwards `.nomad` queries to CoreDNS (port 1053)
   - Listens on localhost and VM IP
   - Upstream DNS: 1.1.1.1, 8.8.8.8, 8.8.4.4

3. **CNI Role**: Container networking
   - Configures VXLAN for Smuggle plugin

4. **Smuggle Role**: Installs Smuggle CNI plugin
   - VXLAN overlay networking

5. **Docker Role**: Installs Docker
   - Version: containerd.io 2.2.0
   - Adds user to docker group

6. **TLS Role**: Generates certificates
   - Server certificate: DNS `server.lcy1.nomad`
   - Client certificate: DNS `client.lcy1.nomad`
   - Both use VM's IP address

7. **Nomad Role (Server)**: Installs Nomad server
   - Version: 1.11.1
   - Config dir: `/etc/nomad-server.d/`
   - Data dir: `/opt/nomad-server/data/`
   - Service: `nomad-server`
   - Config: [`nomad_server.hcl.j2`](templates/nomad_server.hcl.j2:1)

8. **Nomad Role (Client)**: Installs Nomad client
   - Version: 2.0.0-beta.1
   - Config dir: `/etc/nomad-client.d/`
   - Data dir: `/opt/nomad-client/data/`
   - Service: `nomad-client`
   - Config: [`nomad_client.hcl.j2`](templates/nomad_client.hcl.j2:1)

## Configuration Details

### Two Separate Nomad Processes

This deployment runs **two independent Nomad processes**:
- **nomad-server**: Server-only process (v1.11.1)
- **nomad-client**: Client-only process (v2.0.0-beta.1)

Each has its own:
- Configuration directory
- Data directory
- Systemd service
- TLS certificates

### Why Different Versions?

This setup allows testing version compatibility and upgrade scenarios between server and client.

## Prerequisites

- **Multipass**: Install from [multipass.run](https://multipass.run)
- **Terraform**: v1.0 or later
- **Ansible**: v2.9 or later
- **SSH Key**: `~/.ssh/id_rsa.pub` must exist

## Deployment Instructions

### 1. Deploy

```bash
cd lab/multipass/nomad-single-machine
terraform init
terraform apply
```

Deployment takes 5-7 minutes.

### 2. Access the VM

```bash
# SSH into VM
ssh 192.168.64.X
```

### 3. Verify Nomad

```bash
# Check server
sudo systemctl status nomad-server

# Check client
sudo systemctl status nomad-client

# Verify cluster
nomad server members
nomad node status
```

### 4. Bootstrap ACLs

```bash
ssh 192.168.64.X
nomad acl bootstrap
# Save the management token
```

### 5. Access Nomad UI

**Option 1: Port Forwarding** (recommended)
```bash
# Use the command from Terraform output
ssh -L 4646:192.168.64.X:4646 192.168.64.X
```

Open browser to `https://localhost:4646`

**Option 2: Direct Access**
```bash
# Export CA cert
terraform output -raw nomad_ca_cert_pem > nomad-ca.pem

# Set environment
export NOMAD_ADDR=https://192.168.64.X:4646
export NOMAD_CACERT=nomad-ca.pem
export NOMAD_TOKEN=<management-token>

# Access via CLI
nomad status
```

## Usage Examples

### Deploy a Job

Create `example.nomad`:
```hcl
job "example" {
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
```

Deploy:
```bash
nomad job run example.nomad
nomad job status example
```

### Test Service Discovery

```bash
# Deploy CoreDNS job (if available)
nomad job run ../../shared/jobs/coredns.nomad.hcl

# Test DNS resolution
dig @192.168.64.X -p 1053 example.service.nomad
```

### Use Docker Driver

```hcl
task "app" {
  driver = "docker"
  
  config {
    image = "alpine:latest"
    command = "sh"
    args = ["-c", "echo Hello && sleep 3600"]
  }
}
```

### Monitor Logs

```bash
# Server logs
ssh 192.168.64.X "sudo journalctl -u nomad-server -f"

# Client logs
ssh 192.168.64.X "sudo journalctl -u nomad-client -f"
```

## Troubleshooting

### VM Not Starting
```bash
multipass list
multipass info nomad-*
```

### Nomad Server Not Running
```bash
ssh 192.168.64.X "sudo systemctl status nomad-server"
ssh 192.168.64.X "sudo journalctl -u nomad-server -n 50"
```

### Nomad Client Not Running
```bash
ssh 192.168.64.X "sudo systemctl status nomad-client"
ssh 192.168.64.X "sudo journalctl -u nomad-client -n 50"
```

### Client Not Connecting to Server
```bash
# Check client can reach server
ssh 192.168.64.X "curl -k https://localhost:4646/v1/status/leader"

# Check TLS certificates
ssh 192.168.64.X "ls -la /etc/nomad-server.d/.tls/"
ssh 192.168.64.X "ls -la /etc/nomad-client.d/.tls/"
```

### Docker Issues
```bash
ssh 192.168.64.X "docker ps"
ssh 192.168.64.X "sudo systemctl status docker"
```

### DNS Resolution Issues
```bash
ssh 192.168.64.X "dig @localhost web.service.nomad"
ssh 192.168.64.X "sudo systemctl status dnsmasq"
```

## Advanced Configuration

### Restart Services

```bash
# Restart server
ssh 192.168.64.X "sudo systemctl restart nomad-server"

# Restart client
ssh 192.168.64.X "sudo systemctl restart nomad-client"
```

### View Configurations

```bash
# Server config
ssh 192.168.64.X "cat /etc/nomad-server.d/nomad.hcl"

# Client config
ssh 192.168.64.X "cat /etc/nomad-client.d/nomad.hcl"
```

### Upgrade Nomad

Edit [`playbook_nomad.yaml`](playbook_nomad.yaml:58) to change versions, then:
```bash
terraform apply
```

## Cleanup

```bash
terraform destroy
```

## Configuration Files

- [`main.tf`](main.tf:1): Terraform infrastructure definition
- [`ansible.cfg`](ansible.cfg:1): Ansible configuration
- [`inventory.yaml`](inventory.yaml:1): Generated Ansible inventory
- [`playbook_all.yaml`](playbook_all.yaml:1): Master playbook
- [`playbook_nomad.yaml`](playbook_nomad.yaml:1): Nomad configuration
- [`templates/nomad_server.hcl.j2`](templates/nomad_server.hcl.j2:1): Server config template
- [`templates/nomad_client.hcl.j2`](templates/nomad_client.hcl.j2:1): Client config template

## Advantages

- ✅ Simple single-VM deployment
- ✅ Full Nomad features (server + client)
- ✅ TLS and ACLs enabled
- ✅ Docker and CNI networking
- ✅ Easy to destroy and recreate
- ✅ Lower resource requirements than multi-node

## Limitations

- ❌ No high availability
- ❌ Cannot test multi-client scenarios
- ❌ Single point of failure
- ❌ Limited scalability testing

## Next Steps

- **Scale Up**: Use [`nomad-cluster`](../nomad-cluster/) for multi-node testing
- **Add Consul**: Integrate with [`consul-single-machine`](../consul-single-machine/)
- **Add Vault**: Integrate with [`vault-single-machine`](../vault-single-machine/)
- **Deploy Jobs**: Use job specs from `../../shared/jobs/`

## Related Configurations

- [`nomad-cluster`](../nomad-cluster/): Full multi-node cluster
- [`nomad-dev-cluster`](../nomad-dev-cluster/): Development cluster
- [`nomad-server-cluster`](../nomad-server-cluster/): Server-only cluster