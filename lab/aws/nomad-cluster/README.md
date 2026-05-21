# AWS Nomad Cluster

Production-like HashiCorp Nomad cluster deployed on AWS EC2 with 3 servers and 2 clients. Includes TLS encryption, ACLs, and full automation via Terraform and Ansible.

## Overview

This configuration deploys a complete, production-like Nomad cluster on AWS suitable for:
- **Production Testing**: Validate features in cloud environment
- **High Availability**: 3-server Raft consensus cluster
- **Performance Testing**: Cloud-scale workload testing
- **Integration Testing**: Test with AWS services
- **Team Collaboration**: Shared development environment

## Architecture

### Infrastructure

- **3 Nomad Servers**: Raft consensus cluster for HA
- **2 Nomad Clients**: Worker nodes for running jobs
- **Region**: eu-west-2 (London) - customizable
- **Instance Type**: t3.medium (default) - customizable
- **AMI**: Latest Ubuntu (auto-selected)
- **Networking**: VPC with public subnet, internet gateway
- **Security**: Security group (all traffic allowed - dev only!)
- **SSH**: Dynamic key pair generation

### Features

- ✅ TLS encryption for all communication
- ✅ ACLs enabled with pre-generated bootstrap token
- ✅ Go 1.24.1 installed for building from source
- ✅ Docker driver on clients
- ✅ CNI networking support
- ✅ Systemd service management
- ✅ Automated provisioning pipeline

## What the Terraform Does

[`main.tf`](main.tf:1) orchestrates the complete infrastructure:

### 1. Local Variables (lines 1-17)
```hcl
locals {
  stack_region  = "eu-west-2"        # AWS region
  stack_name    = "nomad-cluster"    # Stack identifier
  stack_owner   = "jrasell"          # Owner tag
  ec2_user_data = <<EOH              # Cloud-init config
    # Creates 'jrasell' user with sudo access
    # Adds SSH public key
  EOH
}
```

**Customization Points**:
- Change `stack_region` to deploy in different AWS region
- Update `stack_owner` for your name
- Modify `ec2_user_data` to add your SSH key

### 2. Dynamic SSH Keys (lines 23-29)
Uses `mitchellh/dynamic-keys/aws` module:
- Generates new SSH key pair
- Stores in `./keys/` directory
- Registers with AWS
- Unique per deployment

### 3. AMI Selection (lines 31-33)
Uses shared `aws-ami` module:
- Automatically finds latest Ubuntu AMI
- Filters by owner (Canonical)
- HVM virtualization type

### 4. Network Setup (lines 35-38)
Uses shared `aws-network` module:
- Creates VPC
- Public subnet
- Internet gateway
- Route tables
- Security group (allows all traffic)

### 5. Nomad Servers (lines 40-53)
Provisions 3 server instances:
- Component name: `nomad-server`
- Instance count: 3
- Ansible group: `nomad_server`
- Uses generated SSH key
- Applies cloud-init user data

### 6. Nomad Clients (lines 55-68)
Provisions 2 client instances:
- Component name: `nomad-client`
- Instance count: 2
- Ansible group: `nomad_client`
- Same networking as servers

### 7. Ansible Provisioning (lines 70-76)
Automatically runs after instances are ready:
- Executes [`playbook_all.yaml`](playbook_all.yaml:1)
- Configures all servers and clients
- Generates TLS certificates
- Starts Nomad services

### 8. Outputs (lines 78-100)
Provides:
- SSH commands for all instances
- Rsync commands for code deployment

## What the Ansible Playbooks Do

### [`playbook_all.yaml`](playbook_all.yaml:1)
Master playbook that orchestrates configuration:
- Imports [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:1)
- Imports [`playbook_nomad_client.yaml`](playbook_nomad_client.yaml:1)

### [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:1)
Configures Nomad servers:

1. **Common Role** (lines 3-9):
   - Sets hostname
   - Installs: jq, net-tools, unzip

2. **HashiCorp Release Role** (lines 11-14):
   - Downloads Nomad 1.10.0
   - Verifies checksum
   - Installs binary to `/usr/local/bin/`

3. **Golang Role** (lines 16-19):
   - Installs Go 1.24.1
   - Sets GOPATH to `/home/<user>/go`
   - For building Nomad from source

4. **TLS Role** (lines 21-25):
   - Generates CA certificate
   - Creates server-specific certificates
   - DNS: `server.lhr1.nomad`
   - Stores in `.tls/` directory

5. **Helper Role** (lines 27-54):
   - Installs build tools: build-essential, git, make
   - Creates Nomad config from template: [`nomad_server.hcl.j2`](templates/nomad_server.hcl.j2:1)
   - Copies systemd service file
   - Copies TLS certificates to `/etc/nomad.d/.tls/`
   - Generates bootstrap token file locally
   - Creates `.envrc` with environment variables
   - Starts Nomad service via systemd

**Pre-generated Bootstrap Token**: `b6e63b6a-527c-14db-9474-063cb1dcc026`
- ⚠️ For development only! Change for production!
- Stored in `generated_nomad_root_bootstrap_token`

### [`playbook_nomad_client.yaml`](playbook_nomad_client.yaml:1)
Configures Nomad clients:

1. **Common Role** (lines 3-9):
   - Base system setup

2. **HashiCorp Release Role** (lines 11-14):
   - Installs Nomad 1.10.0

3. **Golang Role** (lines 16-19):
   - Installs Go 1.24.1

4. **CNI Role** (line 21):
   - Installs Container Network Interface plugins
   - For container networking

5. **Docker Role** (lines 23-26):
   - Installs Docker
   - Adds user to docker group
   - Enables Docker driver

6. **TLS Role** (lines 28-32):
   - Generates client certificates
   - DNS: `client.lhr1.nomad`

7. **Helper Role** (lines 34-54):
   - Installs build tools
   - Loads bridge kernel module
   - Creates Nomad config from template: [`nomad_client.hcl.j2`](templates/nomad_client.hcl.j2:1)
   - Copies systemd service and TLS certificates
   - Starts Nomad service

## Prerequisites

### Required Tools

```bash
# AWS CLI
brew install awscli
aws --version

# Terraform
brew install terraform
terraform --version

# Ansible
brew install ansible
ansible --version
```

### AWS Credentials

Configure AWS access:

```bash
# Option 1: AWS CLI
aws configure
# Enter: Access Key ID, Secret Access Key, Region (eu-west-2), Output (json)

# Option 2: Environment variables
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="eu-west-2"

# Verify credentials
aws sts get-caller-identity
```

### AWS Permissions

Your IAM user/role needs:
- EC2: Full access (instances, security groups, key pairs)
- VPC: Full access (subnets, internet gateways, route tables)

Minimum policy:
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

## Deployment Instructions

### Step 1: Customize Configuration

Edit [`main.tf`](main.tf:1) to customize:

```hcl
locals {
  stack_region  = "eu-west-2"        # Change region if needed
  stack_name    = "nomad-cluster"    # Change stack name
  stack_owner   = "yourname"         # Change owner
  ec2_user_data = <<EOH
#cloud-config
---
users:
  - default
  - name: yourname                   # Change username
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa YOUR_PUBLIC_KEY_HERE # Add your SSH key
EOH
}
```

### Step 2: Initialize Terraform

```bash
cd lab/aws/nomad-cluster
terraform init
```

Expected output:
```
Initializing modules...
Initializing the backend...
Initializing provider plugins...
Terraform has been successfully initialized!
```

### Step 3: Review Plan

```bash
terraform plan
```

Review the resources to be created:
- 5 EC2 instances (3 servers + 2 clients)
- 1 VPC
- 1 Subnet
- 1 Internet Gateway
- 1 Route Table
- 1 Security Group
- 1 Key Pair

### Step 4: Deploy Infrastructure

```bash
terraform apply
```

Type `yes` when prompted.

**Deployment time**: 5-8 minutes

**What happens**:
1. Creates AWS resources (2-3 min)
2. Waits for instances to boot (1-2 min)
3. Runs Ansible playbooks (2-3 min)
4. Starts Nomad services

### Step 5: Verify Deployment

After successful deployment, Terraform outputs:

```
SSH commands:
  Nomad Servers:
    - ssh 18.130.X.X
    - ssh 18.130.Y.Y
    - ssh 18.130.Z.Z
  Nomad Client:
    - ssh 18.130.A.A
    - ssh 18.130.B.B

Rsync commands:
  Nomad Servers:
    - rsync -r --exclude 'nomad/ui/node_modules/*' /Users/jrasell/Projects/Go/nomad jrasell@18.130.X.X:/home/jrasell/
  ...
```

### Step 6: Access Nomad Cluster

**SSH into a server**:
```bash
ssh 18.130.X.X
```

**Check Nomad status**:
```bash
# Server members
nomad server members

# Expected output:
# Name                    Address       Port  Status  Leader  Raft Version
# nomad-server-0.lhr1     10.0.1.X:4648  4648  alive   true    3
# nomad-server-1.lhr1     10.0.1.Y:4648  4648  alive   false   3
# nomad-server-2.lhr1     10.0.1.Z:4648  4648  alive   false   3

# Client nodes
nomad node status

# Expected output:
# ID        DC    Name            Class   Drain  Eligibility  Status
# abc123    lhr1  nomad-client-0  <none>  false  eligible     ready
# def456    lhr1  nomad-client-1  <none>  false  eligible     ready
```

### Step 7: Configure Local Nomad CLI

**Option 1: Use generated .envrc file**:
```bash
# Source the environment file
source .envrc

# Verify connection
nomad status
```

**Option 2: Manual configuration**:
```bash
# Set environment variables
export NOMAD_ADDR="https://18.130.X.X:4646"
export NOMAD_TOKEN="b6e63b6a-527c-14db-9474-063cb1dcc026"
export NOMAD_CACERT="./.tls/ca-certificate.pem"
export NOMAD_CLIENT_CERT="./.tls/nomad-server-0-certificate.pem"
export NOMAD_CLIENT_KEY="./.tls/nomad-server-0-certificate.key"

# Verify
nomad server members
nomad node status
```

### Step 8: Access Nomad UI

**Option 1: SSH Port Forwarding** (recommended):
```bash
# Forward port 4646
ssh -L 4646:localhost:4646 18.130.X.X

# Open browser to https://localhost:4646
# Accept self-signed certificate warning
# Enter token: b6e63b6a-527c-14db-9474-063cb1dcc026
```

**Option 2: Direct Access** (requires security group modification):
```bash
# Add your IP to security group
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port 4646 \
  --cidr $(curl -s ifconfig.me)/32

# Access https://18.130.X.X:4646
```

## Usage Examples

### Deploy a Job

Create `example.nomad`:
```hcl
job "example" {
  datacenters = ["lhr1"]
  
  group "web" {
    count = 3
    
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
nomad alloc status <alloc-id>
```

### Test High Availability

```bash
# Stop the leader
ssh <leader-ip> "sudo systemctl stop nomad"

# Watch leader election
nomad server members
# New leader should be elected within seconds

# Restart stopped server
ssh <stopped-ip> "sudo systemctl start nomad"
```

### Deploy Monitoring Stack

```bash
# Deploy Prometheus
nomad job run ../../shared/jobs/monitoring/prometheus-server.nomad.hcl

# Deploy Grafana
nomad job run ../../shared/jobs/monitoring/grafana-server.nomad.hcl

# Check status
nomad job status prometheus-server
nomad job status grafana-server
```

### Sync Local Nomad Source

```bash
# Use rsync command from Terraform output
rsync -r --exclude 'nomad/ui/node_modules/*' \
  ~/Projects/nomad \
  jrasell@18.130.X.X:/home/jrasell/

# SSH and build
ssh 18.130.X.X
cd ~/nomad
make dev
```

## Configuration Files

- [`main.tf`](main.tf:1): Terraform infrastructure definition
- [`ansible.cfg`](ansible.cfg:1): Ansible configuration (roles path, SSH settings)
- [`inventory.yaml`](inventory.yaml:1): Generated by Terraform (Ansible inventory)
- [`playbook_all.yaml`](playbook_all.yaml:1): Master Ansible playbook
- [`playbook_nomad_server.yaml`](playbook_nomad_server.yaml:1): Server configuration
- [`playbook_nomad_client.yaml`](playbook_nomad_client.yaml:1): Client configuration
- [`templates/nomad_server.hcl.j2`](templates/nomad_server.hcl.j2:1): Server config template
- [`templates/nomad_client.hcl.j2`](templates/nomad_client.hcl.j2:1): Client config template
- [`files/nomad.service`](files/nomad.service:1): Systemd service file

## Customization

### Change Instance Type

Edit [`main.tf`](main.tf:40):
```hcl
module "nomad_server" {
  source = "../../shared/terraform/aws-compute"
  
  instance_type = "t3.large"  # Add this line
  # ... other parameters
}
```

### Change Instance Counts

```hcl
module "nomad_server" {
  instance_count = 5  # Change from 3
}

module "nomad_client" {
  instance_count = 10  # Change from 2
}
```

### Change Region

```hcl
locals {
  stack_region = "us-east-1"  # Change from eu-west-2
}
```

### Upgrade Nomad Version

Edit both playbooks:
```yaml
- role: hashicorp_release
  hashicorp_release_product_name: "nomad"
  hashicorp_release_product_version: "1.11.0"  # Change version
  hashicorp_release_product_checksum: "sha256:NEW_CHECKSUM_HERE"
```

Then:
```bash
terraform apply
```

## Cost Management

### Estimated Costs (eu-west-2)

- **5x t3.medium**: ~$0.42/hour (~$302/month if running 24/7)
- **EBS volumes**: ~$0.10/GB/month (default 8GB per instance)
- **Data transfer**: First 1GB free, then $0.09/GB

**Monthly estimate**: ~$350 if running continuously

### Cost Saving Strategies

1. **Stop instances when not in use**:
```bash
# Get instance IDs
aws ec2 describe-instances \
  --filters "Name=tag:Stack,Values=nomad-cluster" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' \
  --output table

# Stop instances
aws ec2 stop-instances --instance-ids i-xxx i-yyy i-zzz

# Start when needed
aws ec2 start-instances --instance-ids i-xxx i-yyy i-zzz
```

2. **Use smaller instances**:
```hcl
instance_type = "t3.small"  # ~$0.02/hour vs $0.08/hour
```

3. **Reduce instance counts**:
```hcl
module "nomad_server" {
  instance_count = 1  # Single server for dev
}
```

4. **Destroy when not needed**:
```bash
terraform destroy
```

## Troubleshooting

### Terraform Errors

**"Error: creating EC2 Instance: UnauthorizedOperation"**
```bash
# Check AWS credentials
aws sts get-caller-identity

# Verify IAM permissions
aws iam get-user-policy --user-name <your-user> --policy-name <policy>
```

**"Error: creating VPC: VpcLimitExceeded"**
```bash
# List VPCs
aws ec2 describe-vpcs --region eu-west-2

# Delete unused VPCs
aws ec2 delete-vpc --vpc-id <vpc-id>
```

### SSH Connection Issues

**"Connection refused"**
```bash
# Check instance is running
aws ec2 describe-instances --instance-ids <instance-id>

# Check security group
aws ec2 describe-security-groups --group-ids <sg-id>

# Try with key file
ssh -i ./keys/nomad-cluster.pem ubuntu@<public-ip>
```

**"Permission denied (publickey)"**
```bash
# Use correct key
ssh -i ./keys/nomad-cluster.pem jrasell@<public-ip>

# Check key permissions
chmod 600 ./keys/nomad-cluster.pem
```

### Nomad Not Running

```bash
# SSH into instance
ssh <instance-ip>

# Check service status
sudo systemctl status nomad

# View logs
sudo journalctl -u nomad -f

# Restart service
sudo systemctl restart nomad
```

### Cluster Not Forming

```bash
# Check server members
nomad server members

# Check connectivity between servers
ssh <server-1> "ping -c 3 <server-2-private-ip>"

# Check Nomad logs
ssh <server-ip> "sudo journalctl -u nomad | grep -i raft"
```

### TLS Certificate Issues

```bash
# Verify certificates exist
ls -la .tls/

# Check certificate validity
openssl x509 -in .tls/nomad-server-0-certificate.pem -text -noout

# Regenerate if needed
rm -rf .tls/
terraform apply
```

## Security Considerations

⚠️ **This configuration is for development/testing only!**

**Current Security Issues**:
- Security group allows all traffic (0.0.0.0/0)
- Pre-generated ACL bootstrap token
- Self-signed TLS certificates
- Instances have public IPs
- No encryption at rest

**For Production**:
1. Restrict security group to specific IPs/ports
2. Generate unique bootstrap token
3. Use proper CA-signed certificates
4. Use private subnets with NAT gateway
5. Enable EBS encryption
6. Implement proper IAM roles
7. Enable CloudTrail logging
8. Use AWS Secrets Manager
9. Implement network ACLs
10. Enable VPC Flow Logs

## Cleanup

### Destroy Infrastructure

```bash
terraform destroy
```

Type `yes` when prompted.

**What gets deleted**:
- All EC2 instances
- VPC and networking
- Security groups
- SSH key pair
- Local key files

### Manual Cleanup (if Terraform fails)

```bash
# List instances
aws ec2 describe-instances \
  --filters "Name=tag:Stack,Values=nomad-cluster" \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]'

# Terminate instances
aws ec2 terminate-instances --instance-ids <id1> <id2> <id3>

# Delete VPC (after instances terminated)
aws ec2 delete-vpc --vpc-id <vpc-id>

# Delete key pair
aws ec2 delete-key-pair --key-name nomad-cluster
```

## Advanced Topics

### Remote State

Use S3 backend for team collaboration:

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "nomad-cluster/terraform.tfstate"
    region = "eu-west-2"
  }
}
```

### Multi-Region Deployment

Deploy to multiple regions:

```bash
# Region 1
cd lab/aws/nomad-cluster
sed -i 's/eu-west-2/us-east-1/g' main.tf
terraform workspace new us-east-1
terraform apply

# Region 2
terraform workspace new eu-central-1
sed -i 's/us-east-1/eu-central-1/g' main.tf
terraform apply
```

### CloudWatch Monitoring

Add monitoring:

```hcl
resource "aws_cloudwatch_metric_alarm" "nomad_cpu" {
  alarm_name          = "nomad-server-high-cpu"
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

## Next Steps

- **Deploy Workloads**: Use job specs from `../../shared/jobs/`
- **Add Consul**: Deploy service discovery
- **Add Vault**: Integrate secrets management
- **Enable Monitoring**: Deploy Prometheus/Grafana
- **Configure Autoscaling**: Set up Nomad Autoscaler
- **Implement CI/CD**: Automate deployments

## Related Configurations

- [`nomad-dev-cluster`](../nomad-dev-cluster/): Smaller development cluster
- [`workstation`](../workstation/): Development instance
- [`../multipass/nomad-cluster`](../../multipass/nomad-cluster/): Local VM alternative

## Support

- [Nomad Documentation](https://www.nomadproject.io/docs)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [HashiCorp Forums](https://discuss.hashicorp.com/c/nomad)