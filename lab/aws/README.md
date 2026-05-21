# AWS Lab Environments

Cloud-based infrastructure deployments on Amazon Web Services for HashiCorp Nomad testing, development, and specialized workloads like virtualization and storage.

## Overview

This directory contains Terraform + Ansible configurations for deploying HashiCorp stack components and supporting infrastructure on AWS EC2. Each subdirectory provides a production-like environment for different use cases, from Nomad clusters to specialized virtualization and storage setups.

## Available Configurations

### Nomad Deployments

| Configuration | Description | Instances | Use Case |
|--------------|-------------|-----------|----------|
| [**nomad-cluster**](#nomad-cluster) | Production-like cluster | 3 servers + 2 clients | Full-featured testing, HA validation |
| [**nomad-dev-cluster**](#nomad-dev-cluster) | Development cluster | 1 server + 1 client | Source code development, rapid iteration |

### Specialized Infrastructure

| Configuration | Description | Instances | Use Case |
|--------------|-------------|-----------|----------|
| [**ceph-standalone**](#ceph-standalone) | Ceph storage cluster | 1 node + 2 EBS volumes | Distributed storage testing |
| [**libvirt-standalone**](#libvirt-standalone) | KVM/libvirt host | 1 metal + 1 router | Nested virtualization, VM testing |
| [**virt**](#virt) | Combined virt + storage | 1 metal + 1 ceph | Full virtualization stack |
| [**workstation**](#workstation) | Development instance | 1 instance | Remote development, builds |

## Architecture Overview

### Common Components

All AWS configurations use shared Terraform modules:

1. **Dynamic SSH Keys** (`mitchellh/dynamic-keys/aws`)
   - Automatically generates and manages SSH key pairs
   - Stores keys in `./keys/` directory
   - Unique per deployment

2. **AMI Selection** (`aws-ami` module)
   - Automatically selects latest Ubuntu AMI
   - Filters by owner and name pattern
   - HVM virtualization type

3. **Networking** (`aws-network` module)
   - Creates VPC with public subnet
   - Configures security groups
   - Internet gateway for public access

4. **Compute** (`aws-compute` module)
   - Provisions EC2 instances
   - Configures cloud-init user data
   - Manages EBS volumes (where needed)
   - Creates Ansible inventory

5. **Ansible Provisioning** (`ansible-provision` module)
   - Executes playbooks automatically
   - Configures instances post-launch

### Default Configuration

- **Region**: eu-west-2 (London)
- **Stack Owner**: jrasell (customizable)
- **User**: jrasell (created via cloud-init)
- **SSH Access**: Public key authentication
- **Instance Type**: t3.medium (default, varies by config)

## Prerequisites

### Required Tools

```bash
# AWS CLI
brew install awscli
aws configure

# Terraform
brew install terraform

# Ansible
brew install ansible

# Verify installations
aws --version
terraform --version
ansible --version
```

### AWS Credentials

Configure AWS credentials:

```bash
# Option 1: AWS CLI configuration
aws configure
# Enter: Access Key ID, Secret Access Key, Region (eu-west-2), Output format (json)

# Option 2: Environment variables
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="eu-west-2"

# Option 3: AWS credentials file
cat > ~/.aws/credentials <<EOF
[default]
aws_access_key_id = your-access-key
aws_secret_access_key = your-secret-key
EOF
```

### AWS Permissions

Your AWS user/role needs permissions for:
- EC2 (instances, security groups, key pairs, EBS volumes)
- VPC (subnets, internet gateways, route tables)
- IAM (if using instance profiles)

Minimum IAM policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "vpc:*"
      ],
      "Resource": "*"
    }
  ]
}
```

## Configuration Details

### nomad-cluster

**Purpose**: Production-like Nomad cluster for comprehensive testing

**Infrastructure** ([`main.tf`](nomad-cluster/main.tf:1)):
- 3 Nomad servers (HA cluster)
- 2 Nomad clients (worker nodes)
- Dynamic SSH key generation
- Latest Ubuntu AMI
- VPC with public subnet
- Security group allowing all traffic (dev only!)

**Customization Points**:
```hcl
locals {
  stack_region  = "eu-west-2"        # Change region
  stack_name    = "nomad-cluster"    # Change stack name
  stack_owner   = "jrasell"          # Change owner tag
  ec2_user_data = <<EOH              # Customize user/SSH keys
    # cloud-init configuration
  EOH
}
```

**Instance Counts**:
- Servers: 3 (line 46)
- Clients: 2 (line 61)

**Deployment**:
```bash
cd lab/aws/nomad-cluster
terraform init
terraform apply
```

**Outputs**:
- SSH commands for all instances
- Rsync commands for code deployment

### nomad-dev-cluster

**Purpose**: Minimal Nomad cluster for development

**Infrastructure** ([`main.tf`](nomad-dev-cluster/main.tf:1)):
- 1 Nomad server
- 1 Nomad client
- Same networking as nomad-cluster
- Smaller footprint for cost savings

**Key Difference**: Single instance of each type (no HA)

**Deployment**:
```bash
cd lab/aws/nomad-dev-cluster
terraform init
terraform apply
```

### ceph-standalone

**Purpose**: Standalone Ceph storage cluster for distributed storage testing

**Infrastructure** ([`main.tf`](ceph-standalone/main.tf:1)):
- 1 EC2 instance for Ceph
- 2 EBS volumes (50GB each, gp3)
  - `/dev/xvdc` - OSD 1
  - `/dev/xvdd` - OSD 2
- Ceph UI accessible on port 8443

**Use Cases**:
- Testing Ceph storage
- NVMe-oF experiments
- Persistent storage for containers
- Block/object storage testing

**Deployment**:
```bash
cd lab/aws/ceph-standalone
terraform init
terraform apply

# Access Ceph UI
# Output shows: https://<public-ip>:8443
```

**Configuration**:
Ansible playbook configures:
- Cephadm installation
- Cluster initialization
- OSD creation from EBS volumes
- Dashboard setup

### libvirt-standalone

**Purpose**: KVM/libvirt host for nested virtualization

**Infrastructure** ([`main.tf`](libvirt-standalone/main.tf:1)):
- 1 c5n.metal instance (bare metal for nested virt)
- 1 t2.nano router instance
- Custom AMI (ami-0474244c88b835731)

**Use Cases**:
- Nested virtualization testing
- Running VMs inside EC2
- Testing VM-based workloads
- Libvirt/KVM development

**Why Metal Instance?**:
- Supports nested virtualization
- Full CPU access
- Better performance for VMs

**Deployment**:
```bash
cd lab/aws/libvirt-standalone
terraform init
terraform apply
```

**Cost Warning**: c5n.metal instances are expensive (~$3.89/hour)

### virt

**Purpose**: Combined virtualization and storage infrastructure

**Infrastructure** ([`main.tf`](virt/main.tf:1)):
- 1 Ceph instance with 2 EBS volumes
- 1 c5n.metal instance for virtualization
- Integrated storage and compute

**Use Cases**:
- Full virtualization stack testing
- VM storage on Ceph
- Persistent volumes for VMs
- Complete infrastructure simulation

**Deployment**:
```bash
cd lab/aws/virt
terraform init
terraform apply
```

**Outputs**:
- Ceph UI URL
- SSH commands for both instances

### workstation

**Purpose**: Cloud-based development workstation

**Infrastructure** ([`main.tf`](workstation/main.tf:1)):
- 1 EC2 instance (t3.medium default)
- Latest Ubuntu AMI
- Development tools via Ansible

**Use Cases**:
- Remote development
- Building HashiCorp tools from source
- Testing in cloud environment
- Team development environment

**Deployment**:
```bash
cd lab/aws/workstation
terraform init
terraform apply
```

## Common Workflows

### 1. Deploy Nomad Cluster for Testing

```bash
# Deploy cluster
cd lab/aws/nomad-cluster
terraform init
terraform apply

# Note the server IPs from output
# SSH into a server
ssh <server-ip>

# Bootstrap ACLs
nomad acl bootstrap

# Configure local CLI
export NOMAD_ADDR=http://<server-ip>:4646
export NOMAD_TOKEN=<bootstrap-token>

# Verify cluster
nomad server members
nomad node status
```

### 2. Development Workflow

```bash
# Deploy dev cluster
cd lab/aws/nomad-dev-cluster
terraform apply

# Sync local code
rsync -r --exclude 'nomad/ui/node_modules/*' \
  ~/Projects/nomad \
  ubuntu@<server-ip>:/home/ubuntu/

# SSH and build
ssh <server-ip>
cd ~/nomad
make dev
```

### 3. Storage Testing with Ceph

```bash
# Deploy Ceph
cd lab/aws/ceph-standalone
terraform apply

# Access Ceph UI
open https://<ceph-ip>:8443

# SSH and check status
ssh <ceph-ip>
sudo ceph status
sudo ceph osd tree
```

### 4. Nested Virtualization

```bash
# Deploy libvirt host
cd lab/aws/libvirt-standalone
terraform apply

# SSH into metal instance
ssh <libvirt-ip>

# Create VMs
virsh list --all
virt-install ...
```

## Cost Management

### Estimated Costs (eu-west-2, per hour)

| Configuration | Instance Types | Approx. Cost/Hour |
|--------------|----------------|-------------------|
| nomad-cluster | 5x t3.medium | ~$0.42 |
| nomad-dev-cluster | 2x t3.medium | ~$0.17 |
| ceph-standalone | 1x t3.medium + EBS | ~$0.10 |
| libvirt-standalone | 1x c5n.metal + 1x t2.nano | ~$3.90 |
| virt | 1x c5n.metal + 1x t3.medium + EBS | ~$4.00 |
| workstation | 1x t3.medium | ~$0.08 |

**Cost Saving Tips**:
1. **Destroy when not in use**: `terraform destroy`
2. **Use smaller instances**: Edit `instance_type` in main.tf
3. **Stop instances**: `aws ec2 stop-instances --instance-ids <id>`
4. **Use spot instances**: Add spot configuration to aws-compute module
5. **Set up billing alerts**: AWS Console → Billing → Budgets

### Auto-Shutdown Script

Create a Lambda function to stop instances after hours:

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2', region_name='eu-west-2')
    
    # Find instances with specific tag
    instances = ec2.describe_instances(
        Filters=[
            {'Name': 'tag:Owner', 'Values': ['jrasell']},
            {'Name': 'instance-state-name', 'Values': ['running']}
        ]
    )
    
    instance_ids = []
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            instance_ids.append(instance['InstanceId'])
    
    if instance_ids:
        ec2.stop_instances(InstanceIds=instance_ids)
        print(f'Stopped instances: {instance_ids}')
```

## Customization

### Change Region

Edit `locals` block in `main.tf`:
```hcl
locals {
  stack_region = "us-east-1"  # Change from eu-west-2
}
```

### Change Instance Types

Edit module call:
```hcl
module "nomad_server" {
  source = "../../shared/terraform/aws-compute"
  
  instance_type = "t3.large"  # Add this line
  # ... other parameters
}
```

### Add Your SSH Key

Edit `ec2_user_data` in `main.tf`:
```hcl
locals {
  ec2_user_data = <<EOH
#cloud-config
---
users:
  - default
  - name: yourname
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa YOUR_PUBLIC_KEY_HERE
EOH
}
```

### Modify Instance Counts

Edit module parameters:
```hcl
module "nomad_server" {
  instance_count = 5  # Change from 3
}

module "nomad_client" {
  instance_count = 10  # Change from 2
}
```

## Troubleshooting

### Terraform Errors

**"Error: creating EC2 Instance: UnauthorizedOperation"**
```bash
# Check AWS credentials
aws sts get-caller-identity

# Verify IAM permissions
aws iam get-user-policy --user-name <your-user> --policy-name <policy-name>
```

**"Error: creating VPC: VpcLimitExceeded"**
```bash
# List existing VPCs
aws ec2 describe-vpcs

# Delete unused VPCs
aws ec2 delete-vpc --vpc-id <vpc-id>
```

**"Error: AMI not found"**
```bash
# Check available AMIs
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-*-amd64-server-*" \
  --query 'Images[*].[ImageId,Name,CreationDate]' \
  --output table
```

### SSH Connection Issues

**"Connection refused"**
```bash
# Check instance is running
aws ec2 describe-instances --instance-ids <instance-id>

# Check security group allows SSH
aws ec2 describe-security-groups --group-ids <sg-id>

# Verify SSH key
ssh -i ./keys/<key-name>.pem ubuntu@<public-ip>
```

**"Permission denied (publickey)"**
```bash
# Use correct key
ssh -i ./keys/<stack-name>.pem ubuntu@<public-ip>

# Check key permissions
chmod 600 ./keys/<stack-name>.pem
```

### Ansible Failures

**"UNREACHABLE! => SSH Error"**
```bash
# Wait for instance to fully boot
sleep 60

# Test SSH manually
ssh ubuntu@<public-ip>

# Re-run Ansible
cd <config-dir>
ansible-playbook -i inventory.yaml playbook_all.yaml
```

**"Failed to connect to the host via ssh"**
```bash
# Check inventory file
cat inventory.yaml

# Verify Ansible can reach hosts
ansible all -i inventory.yaml -m ping
```

### Instance Access

**Can't access Nomad UI**
```bash
# Check security group
aws ec2 describe-security-groups --group-ids <sg-id>

# Add rule if needed
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port 4646 \
  --cidr 0.0.0.0/0
```

## Cleanup

### Destroy Single Configuration

```bash
cd <configuration-directory>
terraform destroy
```

### Destroy All AWS Resources

```bash
# Destroy each configuration
for dir in nomad-cluster nomad-dev-cluster ceph-standalone libvirt-standalone virt workstation; do
  cd lab/aws/$dir
  terraform destroy -auto-approve
  cd -
done
```

### Manual Cleanup (if Terraform fails)

```bash
# List all instances with your tag
aws ec2 describe-instances \
  --filters "Name=tag:Owner,Values=jrasell" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Terminate instances
aws ec2 terminate-instances --instance-ids <id1> <id2> <id3>

# Delete VPCs
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=*nomad*"
aws ec2 delete-vpc --vpc-id <vpc-id>

# Delete key pairs
aws ec2 describe-key-pairs
aws ec2 delete-key-pair --key-name <key-name>
```

## Security Considerations

⚠️ **These configurations are for development/testing only!**

**Security Issues**:
- Security groups allow all traffic (0.0.0.0/0)
- Instances have public IPs
- No encryption at rest configured
- Default passwords may be used
- SSH keys stored locally

**For Production**:
1. Restrict security groups to specific IPs
2. Use private subnets with NAT gateway
3. Enable EBS encryption
4. Use AWS Secrets Manager for credentials
5. Implement proper IAM roles
6. Enable CloudTrail logging
7. Use AWS Systems Manager Session Manager instead of SSH
8. Implement network ACLs
9. Enable VPC Flow Logs
10. Use AWS KMS for key management

## Best Practices

### Resource Tagging

All resources are tagged with:
- `Name`: Component identifier
- `Owner`: Stack owner
- `Stack`: Stack name

Add more tags in shared modules:
```hcl
tags = {
  Environment = "development"
  Project     = "nomad-testing"
  CostCenter  = "engineering"
}
```

### State Management

**Use Remote State** (recommended):
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "lab/aws/nomad-cluster/terraform.tfstate"
    region = "eu-west-2"
  }
}
```

### Module Versioning

Pin module versions:
```hcl
module "keys" {
  source  = "mitchellh/dynamic-keys/aws"
  version = "= 2.0.0"  # Exact version
}
```

## Advanced Topics

### Multi-Region Deployment

Deploy to multiple regions:

```bash
# Region 1: London
cd lab/aws/nomad-cluster
sed -i 's/eu-west-2/eu-west-1/g' main.tf
terraform apply

# Region 2: Frankfurt
sed -i 's/eu-west-1/eu-central-1/g' main.tf
terraform apply
```

### VPC Peering

Connect clusters across regions:

```hcl
resource "aws_vpc_peering_connection" "peer" {
  vpc_id        = module.network.vpc_id
  peer_vpc_id   = "<other-vpc-id>"
  peer_region   = "eu-central-1"
  auto_accept   = false
}
```

### Monitoring Integration

Add CloudWatch monitoring:

```hcl
resource "aws_cloudwatch_metric_alarm" "cpu" {
  alarm_name          = "nomad-server-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "120"
  statistic           = "Average"
  threshold           = "80"
  
  dimensions = {
    InstanceId = module.nomad_server.instance_ids[0]
  }
}
```

## Related Documentation

- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Nomad on AWS](https://www.nomadproject.io/docs/install/production/deployment-guide#cloud-auto-join-on-aws)
- [Ceph Documentation](https://docs.ceph.com/)
- [Libvirt Documentation](https://libvirt.org/docs.html)

## Support

For issues specific to:
- **AWS**: Check [AWS Support](https://aws.amazon.com/support/)
- **Terraform**: Check [Terraform Registry](https://registry.terraform.io/)
- **Nomad**: Check [HashiCorp Forums](https://discuss.hashicorp.com/c/nomad)
- **Ceph**: Check [Ceph Mailing Lists](https://ceph.io/en/community/)