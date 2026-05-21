# Workstation

A development workstation VM configured for HashiCorp stack development with Go, Docker, Git, and build tools.

## Overview

This configuration deploys a development workstation suitable for:
- Remote development environment
- Building HashiCorp tools from source
- Testing and experimentation
- Isolated development environment
- Learning and training

## Architecture

- **1 VM**: Development workstation
- **VM Resources**: Default (1 CPU, 1GiB RAM)
- **Hostname**: event-horizon
- **Features**:
  - Go 1.25.5 installed
  - Docker with user access
  - Git configured for GitHub
  - Build tools (make, gcc, etc.)
  - SSH keys copied from host
  - CNI plugins

## What the Terraform Does

[`main.tf`](main.tf:1) provisions a development VM:

1. **Creates Workstation VM** (via `multipass-compute` module):
   - Instance: `workstation-<random-id>`
   - Default specs: 1 CPU, 1GiB RAM
   - Ansible group: `workstation`

2. **Runs Ansible Provisioning** (via `ansible-provision` module):
   - Executes [`playbook_workstation.yaml`](playbook_workstation.yaml:1)
   - Configures development environment

3. **Outputs**:
   - SSH command
   - Rsync command for syncing code

## What the Ansible Playbook Does

### [`playbook_workstation.yaml`](playbook_workstation.yaml:1)

Configures a complete development environment:

**Variables to Customize** (lines 7-10):
```yaml
github_email: "your-email@users.noreply.github.com"
ssh_key_private_filename: "id_rsa"
ssh_key_public_filename: "id_rsa.pub"
```

**Roles Applied**:

1. **Common Role**: Base system setup
   - Sets hostname to "event-horizon"
   - Installs: jq, net-tools, unzip

2. **Golang Role**: Installs Go 1.25.5
   - GOPATH: `~/go`
   - Required for building HashiCorp tools

3. **CNI Role**: Installs Container Network Interface plugins
   - For container networking

4. **Docker Role**: Installs Docker
   - Adds user to docker group
   - Enables rootless containers

5. **Helper Role**: Development tools and configuration
   - Installs: build-essential, git, make
   - Copies SSH keys from local machine
   - Configures Git:
     - Sets email
     - Configures SSH for GitHub (instead of HTTPS)

**Tasks**:
- Adds GitHub SSH keys to known_hosts

## Prerequisites

- **Multipass**: Install from [multipass.run](https://multipass.run)
- **Terraform**: v1.0 or later
- **Ansible**: v2.9 or later
- **SSH Key**: `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub` must exist
- **GitHub Access**: SSH key added to GitHub account

## Deployment Instructions

### 1. Customize Variables

Edit [`playbook_workstation.yaml`](playbook_workstation.yaml:7-10):
```yaml
vars:
  github_email: "your-email@users.noreply.github.com"
  ssh_key_private_filename: "id_rsa"
  ssh_key_public_filename: "id_rsa.pub"
```

### 2. Deploy

```bash
cd lab/multipass/workstation
terraform init
terraform apply
```

### 3. Access Workstation

```bash
# SSH into workstation
ssh 192.168.64.X
```

### 4. Verify Setup

```bash
# Check Go installation
go version

# Check Docker
docker ps

# Check Git configuration
git config --list

# Test GitHub access
ssh -T git@github.com
```

## Usage Examples

### Clone and Build Nomad

```bash
# SSH into workstation
ssh 192.168.64.X

# Clone Nomad
cd ~
git clone git@github.com:hashicorp/nomad.git
cd nomad

# Build Nomad
make bootstrap
make dev

# Run tests
make test
```

### Clone and Build Consul

```bash
git clone git@github.com:hashicorp/consul.git
cd consul
make dev
```

### Clone and Build Vault

```bash
git clone git@github.com:hashicorp/vault.git
cd vault
make bootstrap
make dev
```

### Sync Local Code to Workstation

Use the rsync command from Terraform output:

```bash
# Sync Nomad source
rsync -r --exclude 'nomad/ui/node_modules/*' \
  ~/Projects/Go/nomad \
  ubuntu@192.168.64.X:/home/ubuntu/

# Sync any project
rsync -avz --exclude 'node_modules' \
  ~/Projects/myproject \
  ubuntu@192.168.64.X:/home/ubuntu/
```

### Run Docker Containers

```bash
# Pull and run a container
docker run -d -p 8080:80 nginx:alpine

# Build a custom image
cd ~/myproject
docker build -t myapp:latest .
docker run -p 3000:3000 myapp:latest
```

### Use as Remote Development Environment

**VS Code Remote SSH**:
1. Install "Remote - SSH" extension
2. Add SSH host: `192.168.64.X`
3. Connect and open folder
4. Develop directly on the workstation

**JetBrains IDEs**:
1. Configure remote interpreter
2. Set deployment to workstation
3. Use remote Go SDK

## Development Workflows

### Iterative Development

```bash
# On local machine: make changes
vim ~/Projects/nomad/command/agent.go

# Sync to workstation
rsync -avz ~/Projects/nomad/ ubuntu@192.168.64.X:/home/ubuntu/nomad/

# On workstation: build and test
ssh 192.168.64.X
cd ~/nomad
make dev
./bin/nomad version
```

### Testing with Docker

```bash
# Build test container
docker build -t test-env .

# Run tests in container
docker run --rm test-env make test

# Interactive testing
docker run -it --rm test-env /bin/bash
```

### Multi-Project Development

```bash
# Work on multiple projects
~/nomad/     # Nomad source
~/consul/    # Consul source
~/vault/     # Vault source
~/myapp/     # Your application
```

## Customization

### Increase VM Resources

Modify [`main.tf`](main.tf:1):
```hcl
module "workstation" {
  source = "../../shared/terraform/multipass-compute"
  
  instance_cpus   = 4
  instance_memory = "8GiB"
  instance_disk   = "50GB"
  
  ansible_group_name = "workstation"
  instance_ssh_key   = file("~/.ssh/id_rsa.pub")
}
```

Then:
```bash
terraform apply
```

### Install Additional Tools

Edit [`playbook_workstation.yaml`](playbook_workstation.yaml:1) to add more packages:

```yaml
- role: common
  common_apt_packages:
    - "jq"
    - "net-tools"
    - "unzip"
    - "vim"
    - "tmux"
    - "htop"
    - "postgresql-client"
```

### Add More Languages

```yaml
- role: nodejs
  nodejs_version: "20.x"

- role: python
  python_version: "3.11"
```

## Troubleshooting

### VM Not Starting
```bash
multipass list
multipass info workstation-*
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

# Check Git configuration
ssh 192.168.64.X "git config --list"
```

### Docker Permission Denied
```bash
# Verify user in docker group
ssh 192.168.64.X "groups"

# If not, log out and back in
ssh 192.168.64.X "exit"
ssh 192.168.64.X
```

### Go Build Fails
```bash
# Verify Go installation
ssh 192.168.64.X "go version"
ssh 192.168.64.X "go env"

# Check GOPATH
ssh 192.168.64.X "echo \$GOPATH"
```

## Cleanup

```bash
terraform destroy
```

## Configuration Files

- [`main.tf`](main.tf:1): Terraform infrastructure definition
- [`ansible.cfg`](ansible.cfg:1): Ansible configuration
- [`inventory.yaml`](inventory.yaml:1): Generated Ansible inventory
- [`playbook_workstation.yaml`](playbook_workstation.yaml:1): Workstation configuration

## Advantages

- ✅ Isolated development environment
- ✅ Consistent setup across team members
- ✅ Easy to destroy and recreate
- ✅ No pollution of host machine
- ✅ Can run resource-intensive builds
- ✅ Test in Linux environment (if host is macOS/Windows)

## Use Cases

- **HashiCorp Development**: Build Nomad, Consul, Vault from source
- **Remote Work**: Develop on a remote machine
- **Team Onboarding**: Consistent dev environment for new team members
- **Testing**: Isolated environment for experiments
- **Learning**: Safe environment for learning new tools

## Next Steps

- **Connect to Nomad**: Use with [`nomad-cluster`](../nomad-cluster/) for testing
- **Add More Tools**: Customize with additional development tools
- **Increase Resources**: Scale up for larger projects
- **Snapshot**: Create Multipass snapshot for quick recovery

## Related Configurations

- [`nomad-dev-cluster`](../nomad-dev-cluster/): Development Nomad cluster
- [`nomad-cluster`](../nomad-cluster/): Full Nomad cluster for testing
- [`consul-single-machine`](../consul-single-machine/): Consul server
- [`vault-single-machine`](../vault-single-machine/): Vault server