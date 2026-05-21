# Shared Terraform Modules

Reusable Terraform modules for provisioning infrastructure across AWS and Multipass environments. These modules provide consistent, DRY infrastructure patterns used throughout the lab configurations.

## Overview

This directory contains 7 Terraform modules that abstract common infrastructure patterns:

| Module | Purpose | Used By |
|--------|---------|---------|
| [**ansible-provision**](#ansible-provision) | Execute Ansible playbooks | All configs |
| [**aws-ami**](#aws-ami) | Select latest Ubuntu AMI | AWS configs |
| [**aws-compute**](#aws-compute) | Provision EC2 instances | AWS configs |
| [**aws-network**](#aws-network) | Create VPC networking | AWS configs |
| [**multipass-compute**](#multipass-compute) | Provision Multipass VMs | Multipass configs |
| [**tls**](#tls) | Generate TLS certificates | Secure configs |
| [**hashicorp-vault-nomad-wi**](#hashicorp-vault-nomad-wi) | Vault workload identity | Vault integration |

## Module Reference

### ansible-provision

Executes Ansible playbooks after infrastructure provisioning.

**Location**: [`ansible-provision/`](ansible-provision/)

**Purpose**: Automate configuration management by running Ansible playbooks against provisioned infrastructure.

**Inputs**:

| Variable | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `ansible_inventory_path` | string | ✅ | - | Path to Ansible inventory file |
| `ansible_playbook_path` | string | ✅ | - | Path to Ansible playbook to execute |
| `ansible_extra_vars` | list(string) | ❌ | `[]` | Extra variables to pass to Ansible |

**Outputs**: None

**Usage Example**:
```hcl
module "ansible_provision" {
  source     = "../../shared/terraform/ansible-provision"
  depends_on = [module.nomad_server, module.nomad_client]

  ansible_inventory_path = abspath("./inventory.yaml")
  ansible_playbook_path  = abspath("./playbook_all.yaml")
  ansible_extra_vars     = [
    "consul_address=192.168.1.10:8500",
    "vault_address=https://192.168.1.20:8200"
  ]
}
```

**How It Works**:
1. Waits for dependencies (instances) to be ready
2. Executes `ansible-playbook` command
3. Uses specified inventory and playbook
4. Passes extra variables if provided

**Best Practices**:
- Always use `depends_on` to ensure instances are ready
- Use `abspath()` for file paths to avoid relative path issues
- Pass dynamic values (IPs, addresses) via `ansible_extra_vars`

---

### aws-ami

Automatically selects the latest Ubuntu AMI for the current region.

**Location**: [`aws-ami/`](aws-ami/)

**Purpose**: Eliminate hardcoded AMI IDs and always use the latest Ubuntu image.

**Inputs**:

| Variable | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `ami_owner` | string | ❌ | `"099720109477"` | AMI owner ID (Canonical) |
| `ami_name_filter` | string | ❌ | `"ubuntu/images/hvm-ssd/ubuntu-*-amd64-server-*"` | AMI name filter pattern |

**Outputs**:

| Output | Type | Description |
|--------|------|-------------|
| `ami_id` | string | ID of the selected AMI |

**Usage Example**:
```hcl
module "ami" {
  source = "../../shared/terraform/aws-ami"
}

module "nomad_server" {
  source = "../../shared/terraform/aws-compute"
  ami_id = module.ami.ami_id
  # ... other parameters
}
```

**How It Works**:
1. Queries AWS for AMIs matching filters
2. Selects most recent AMI
3. Filters by HVM virtualization type
4. Returns AMI ID for use in compute module

**Customization**:
```hcl
# Use specific Ubuntu version
module "ami" {
  source          = "../../shared/terraform/aws-ami"
  ami_name_filter = "ubuntu/images/hvm-ssd/ubuntu-22.04-*"
}

# Use different Linux distribution
module "ami" {
  source          = "../../shared/terraform/aws-ami"
  ami_owner       = "137112412989"  # Amazon Linux
  ami_name_filter = "amzn2-ami-hvm-*-x86_64-gp2"
}
```

---

### aws-compute

Provisions EC2 instances with optional EBS volumes and Ansible integration.

**Location**: [`aws-compute/`](aws-compute/)

**Purpose**: Create EC2 instances with consistent configuration and automatic Ansible inventory generation.

**Inputs**:

| Variable | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `component_name` | string | ✅ | - | Component name for tagging |
| `stack_name` | string | ✅ | - | Stack name for tagging |
| `stack_owner` | string | ❌ | `"jrasell"` | Owner identifier |
| `instance_count` | number | ❌ | `1` | Number of instances |
| `ami_id` | string | ❌ | `"ami-03628db51da52eeaa"` | AMI to use |
| `instance_type` | string | ❌ | `"m5.xlarge"` | Instance type |
| `subnet_id` | string | ✅ | - | VPC subnet ID |
| `user_data` | string | ❌ | `""` | Cloud-init user data |
| `security_group_ids` | list(string) | ✅ | - | Security group IDs |
| `ssh_key_name` | string | ✅ | - | SSH key pair name |
| `ebs_block_devices` | list(object) | ❌ | `[]` | Additional EBS volumes |
| `ansible_user` | string | ❌ | `"jrasell"` | Ansible SSH user |
| `ansible_group_name` | string | ❌ | `"default"` | Ansible inventory group |

**Outputs**:

| Output | Type | Description |
|--------|------|-------------|
| `instance_public_ips` | list(string) | Public IP addresses of instances |
| `instance_ids` | list(string) | Instance IDs |

**Usage Example**:
```hcl
module "nomad_server" {
  source = "../../shared/terraform/aws-compute"

  ami_id             = module.ami.ami_id
  ansible_group_name = "nomad_server"
  component_name     = "nomad-server"
  instance_count     = 3
  instance_type      = "t3.medium"
  security_group_ids = [module.network.security_group_id]
  ssh_key_name       = module.keys.key_name
  stack_name         = "nomad-cluster"
  stack_owner        = "myteam"
  subnet_id          = module.network.subnet_id
  user_data          = local.ec2_user_data
}
```

**With EBS Volumes**:
```hcl
module "ceph_compute" {
  source = "../../shared/terraform/aws-compute"

  component_name = "ceph"
  ebs_block_devices = [
    {
      size   = 50
      type   = "gp3"
      device = "/dev/xvdc"
    },
    {
      size   = 100
      type   = "gp3"
      device = "/dev/xvdd"
    }
  ]
  # ... other parameters
}
```

**Features**:
- Automatic Ansible inventory generation
- Consistent tagging (Name, Stack, Owner, Component)
- Optional EBS volume attachment
- Cloud-init user data support
- Multiple instance support

---

### aws-network

Creates VPC with public subnet, internet gateway, and security group.

**Location**: [`aws-network/`](aws-network/)

**Purpose**: Provide basic networking infrastructure for AWS deployments.

**Inputs**:

| Variable | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `stack_name` | string | ✅ | - | Stack name for tagging |
| `stack_owner` | string | ❌ | `"jrasell"` | Owner identifier |

**Outputs**:

| Output | Type | Description |
|--------|------|-------------|
| `subnet_id` | string | ID of the created subnet |
| `security_group_id` | string | ID of the security group |
| `vpc_id` | string | ID of the VPC |

**Usage Example**:
```hcl
module "network" {
  source     = "../../shared/terraform/aws-network"
  stack_name = "nomad-cluster"
}

module "nomad_server" {
  source             = "../../shared/terraform/aws-compute"
  subnet_id          = module.network.subnet_id
  security_group_ids = [module.network.security_group_id]
  # ... other parameters
}
```

**What It Creates**:
1. **VPC**: 10.0.0.0/16 CIDR block
2. **Public Subnet**: 10.0.1.0/24 CIDR block
3. **Internet Gateway**: For internet access
4. **Route Table**: Routes 0.0.0.0/0 to IGW
5. **Security Group**: Allows all traffic (⚠️ dev only!)

**Security Warning**:
⚠️ The security group allows **all traffic** from **all sources** (0.0.0.0/0). This is for development/testing only!

**Production Hardening**:
```hcl
# After creating the network, modify the security group
resource "aws_security_group_rule" "ssh_only" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["YOUR_IP/32"]
  security_group_id = module.network.security_group_id
}

resource "aws_security_group_rule" "nomad_api" {
  type              = "ingress"
  from_port         = 4646
  to_port           = 4646
  protocol          = "tcp"
  cidr_blocks       = ["YOUR_IP/32"]
  security_group_id = module.network.security_group_id
}
```

---

### multipass-compute

Provisions Multipass VMs with cloud-init and Ansible integration.

**Location**: [`multipass-compute/`](multipass-compute/)

**Purpose**: Create local VMs using Multipass with consistent configuration.

**Inputs**:

| Variable | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `instance_image` | string | ❌ | `"24.04"` | Ubuntu version |
| `instance_name_prefix` | string | ❌ | `"instance"` | Name prefix for VMs |
| `instance_count` | number | ❌ | `1` | Number of VMs |
| `instance_cpus` | number | ❌ | `4` | CPUs per VM |
| `instance_memory` | string | ❌ | `"8GiB"` | Memory per VM |
| `instance_disk` | string | ❌ | `"20GiB"` | Disk size per VM |
| `instance_ssh_key` | string | ✅ | - | SSH public key content |
| `instance_ssh_user` | string | ❌ | `"jrasell"` | SSH user to create |
| `ansible_user` | string | ❌ | `"jrasell"` | Ansible SSH user |
| `ansible_group_name` | string | ❌ | `"default"` | Ansible inventory group |

**Outputs**:

| Output | Type | Description |
|--------|------|-------------|
| `instance_ips` | list(string) | IP addresses of VMs |
| `instance_names` | list(string) | Names of VMs |

**Usage Example**:
```hcl
module "nomad_server" {
  source = "../../shared/terraform/multipass-compute"

  ansible_group_name   = "nomad_server"
  instance_count       = 3
  instance_cpus        = 2
  instance_memory      = "4GiB"
  instance_name_prefix = "nomad-server"
  instance_ssh_key     = file("~/.ssh/id_rsa.pub")
}
```

**Features**:
- Automatic cloud-init generation
- Random suffix for unique names
- Ansible inventory integration
- Configurable resources per VM

**Resource Sizing Guide**:
```hcl
# Minimal (testing)
instance_cpus   = 1
instance_memory = "1GiB"
instance_disk   = "10GiB"

# Standard (development)
instance_cpus   = 2
instance_memory = "4GiB"
instance_disk   = "20GiB"

# Large (performance testing)
instance_cpus   = 4
instance_memory = "8GiB"
instance_disk   = "50GiB"
```

---

### tls

Generates TLS certificates for secure communication.

**Location**: [`tls/`](tls/)

**Purpose**: Create CA and agent certificates for TLS-enabled deployments.

**Inputs**:

| Variable | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `local_path` | string | ❌ | `"./tls"` | Local storage path |
| `ca_create` | bool | ❌ | `false` | Create new CA |
| `ca_subject` | object | ❌ | See below | CA subject info |
| `ca_validity_hours` | number | ❌ | `43800` | CA validity (5 years) |
| `certificate` | object | ❌ | `null` | Certificate details |

**Default CA Subject**:
```hcl
{
  country             = "GB"
  province            = "Kent"
  locality            = "Faversham"
  common_name         = "Nomad Agent CA"
  organization        = "HashiCorp"
  organizational_unit = "Nomad Engineering"
}
```

**Outputs**:

| Output | Type | Description |
|--------|------|-------------|
| `ca_cert_pem` | string | CA certificate PEM |

**Usage Example**:
```hcl
# Create CA
module "ca" {
  source = "../../shared/terraform/tls"

  local_path = "${path.root}/.tls"
  ca_create  = true
}

# Generate server certificate
module "server_cert" {
  source = "../../shared/terraform/tls"

  local_path = "${path.root}/.tls"
  certificate = {
    name           = "nomad-server-0"
    dns_names      = ["server.lcy1.nomad", "localhost"]
    ip_addresses   = ["192.168.1.10", "127.0.0.1"]
    validity_hours = 8760  # 1 year
    ca_cert_pem    = module.ca.ca_cert_pem
    ca_cert_key    = file("${path.root}/.tls/ca-key.pem")
  }
}
```

**Files Generated**:
```
.tls/
├── ca-certificate.pem       # CA certificate
├── ca-certificate.key       # CA private key
├── nomad-server-0.pem       # Agent certificate
└── nomad-server-0-key.pem   # Agent private key
```

**Best Practices**:
- Create CA once, reuse for all agents
- Use DNS names for flexibility
- Include IP addresses for direct access
- Set appropriate validity periods
- Store certificates securely
- Rotate certificates before expiry

---

### hashicorp-vault-nomad-wi

Configures Vault for Nomad workload identity integration.

**Location**: [`hashicorp-vault-nomad-wi/`](hashicorp-vault-nomad-wi/)

**Purpose**: Set up Vault JWT auth backend for Nomad workload identity.

**Usage**: See module documentation for Vault-specific configuration.

**Key Features**:
- JWT auth backend configuration
- Role creation for Nomad workloads
- Policy management
- Token configuration

---

## Development Guide

### Creating a New Module

1. **Create module directory**:
```bash
mkdir -p lab/shared/terraform/my-module
cd lab/shared/terraform/my-module
```

2. **Create required files**:
```bash
touch main.tf variable.tf output.tf README.md
```

3. **Define variables** (`variable.tf`):
```hcl
variable "required_param" {
  description = "A required parameter"
  type        = string
}

variable "optional_param" {
  description = "An optional parameter"
  type        = string
  default     = "default-value"
}
```

4. **Implement resources** (`main.tf`):
```hcl
resource "example_resource" "main" {
  name  = var.required_param
  value = var.optional_param
}
```

5. **Define outputs** (`output.tf`):
```hcl
output "resource_id" {
  description = "ID of the created resource"
  value       = example_resource.main.id
}
```

6. **Document module** (`README.md`):
- Purpose and use cases
- Input variables
- Output values
- Usage examples
- Best practices

### Module Design Principles

1. **Single Responsibility**: Each module should do one thing well
2. **Composability**: Modules should work together seamlessly
3. **Sensible Defaults**: Provide defaults for optional parameters
4. **Clear Outputs**: Export useful values for other modules
5. **Documentation**: Include examples and best practices

### Testing Modules

```bash
# Initialize module
cd lab/shared/terraform/my-module
terraform init

# Validate syntax
terraform validate

# Format code
terraform fmt

# Test in isolation
terraform plan
```

### Versioning Strategy

While these modules aren't published to a registry, follow semantic versioning principles:

- **Major**: Breaking changes to inputs/outputs
- **Minor**: New features, backward compatible
- **Patch**: Bug fixes, no API changes

Document changes in module README.

## Common Patterns

### Pattern 1: Infrastructure + Configuration

```hcl
# Provision infrastructure
module "compute" {
  source = "../../shared/terraform/aws-compute"
  # ... parameters
}

# Configure with Ansible
module "provision" {
  source     = "../../shared/terraform/ansible-provision"
  depends_on = [module.compute]
  
  ansible_inventory_path = abspath("./inventory.yaml")
  ansible_playbook_path  = abspath("./playbook.yaml")
}
```

### Pattern 2: Networking + Compute

```hcl
# Create network
module "network" {
  source     = "../../shared/terraform/aws-network"
  stack_name = "my-stack"
}

# Create instances in network
module "instances" {
  source             = "../../shared/terraform/aws-compute"
  subnet_id          = module.network.subnet_id
  security_group_ids = [module.network.security_group_id]
  # ... other parameters
}
```

### Pattern 3: TLS + Secure Services

```hcl
# Generate CA
module "ca" {
  source    = "../../shared/terraform/tls"
  ca_create = true
}

# Generate certificates for each instance
module "server_certs" {
  source = "../../shared/terraform/tls"
  count  = 3
  
  certificate = {
    name           = "server-${count.index}"
    dns_names      = ["server-${count.index}.example.com"]
    ip_addresses   = [module.servers.instance_ips[count.index]]
    validity_hours = 8760
    ca_cert_pem    = module.ca.ca_cert_pem
    ca_cert_key    = file(".tls/ca-key.pem")
  }
}
```

## Troubleshooting

### Module Not Found

```bash
# Error: Module not found
Error: Module not found: ../../shared/terraform/my-module

# Solution: Check path is correct
ls -la ../../shared/terraform/my-module

# Initialize Terraform
terraform init
```

### Variable Type Mismatch

```bash
# Error: Invalid value for input variable
Error: Invalid value for input variable "instance_count": number required

# Solution: Check variable type
instance_count = 3  # Not "3"
```

### Circular Dependency

```bash
# Error: Cycle in module dependencies
Error: Cycle: module.a, module.b

# Solution: Use explicit depends_on
module "b" {
  depends_on = [module.a]
}
```

## Best Practices

### 1. Use Modules Consistently

✅ **Good**:
```hcl
module "servers" {
  source = "../../shared/terraform/aws-compute"
  # ... parameters
}
```

❌ **Bad**:
```hcl
resource "aws_instance" "server" {
  # Duplicating module logic
}
```

### 2. Pass Values Between Modules

✅ **Good**:
```hcl
module "network" {
  source = "../../shared/terraform/aws-network"
}

module "compute" {
  source    = "../../shared/terraform/aws-compute"
  subnet_id = module.network.subnet_id  # Use output
}
```

### 3. Use Variables for Customization

✅ **Good**:
```hcl
module "servers" {
  source         = "../../shared/terraform/aws-compute"
  instance_count = var.server_count
  instance_type  = var.server_type
}
```

### 4. Document Module Usage

Always include usage examples in module READMEs.

### 5. Version Control

Commit module changes with clear messages:
```bash
git commit -m "aws-compute: Add support for EBS volumes"
```

## Contributing

When modifying modules:

1. **Test thoroughly**: Verify changes don't break existing configs
2. **Update documentation**: Keep READMEs current
3. **Maintain compatibility**: Avoid breaking changes when possible
4. **Add examples**: Show how to use new features
5. **Consider impact**: Changes affect all configurations using the module

## Related Documentation

- [Terraform Module Documentation](https://www.terraform.io/docs/language/modules/)
- [AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Multipass Provider Documentation](https://registry.terraform.io/providers/larstobi/multipass/latest/docs)
- [TLS Provider Documentation](https://registry.terraform.io/providers/hashicorp/tls/latest/docs)