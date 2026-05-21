# nomad-infra-ansible-dev

> Forked from [jrasell/dev-mess](https://github.com/jrasell/dev-mess)

A comprehensive development infrastructure repository focused on HashiCorp Nomad cluster deployment, testing, and development workflows. This project provides multiple deployment patterns across different environments (AWS, Multipass VMs, and local).

## Table of Contents

* [Overview](#overview)
* [Project Structure](#project-structure)
  * [lab/](#lab---infrastructure-lab-environments)
  * [code/](#code---development-code)
  * [tools/](#tools---utility-tools)
  * [workstation/](#workstation---workstation-configuration)
* [Key Technologies](#key-technologies)
* [Deployment Patterns](#deployment-patterns)
* [Getting Started](#getting-started)
* [Use Cases](#use-cases)

## Overview

This repository contains infrastructure tooling and workstation configurations for developing, testing, and running HashiCorp Nomad clusters. It supports multiple deployment strategies:

- **AWS**: Cloud-based production-like environments
- **Multipass**: Local VM-based testing environments
- **Local**: Direct host-based development setups

All configurations use Infrastructure as Code (Terraform) and Configuration Management (Ansible) for reproducible, automated deployments.

## Project Structure

### lab/ - Infrastructure Lab Environments

The core of the project, organized by deployment platform.

#### AWS Deployments (`lab/aws/`)

Cloud-based environments for production-like testing:

- **nomad-cluster**: Full Nomad cluster (3 servers, 2 clients)
- **nomad-dev-cluster**: Development cluster with Vault integration
- **ceph-standalone**: Ceph distributed storage testing
- **libvirt-standalone**: Libvirt/KVM virtualization setup
- **virt**: Combined virtualization and Ceph environment
- **workstation**: AWS-based development workstation

Each deployment uses:
- Terraform for infrastructure provisioning
- Ansible for configuration management
- Dynamic SSH key generation
- Automated provisioning pipeline

#### Multipass Deployments (`lab/multipass/`)

Local VM-based environments using Canonical Multipass:

- **nomad-cluster**: Full cluster with Consul integration (3 servers, 2 clients)
- **nomad-dev-cluster**: Development cluster with simplified configuration
- **nomad-server-cluster**: Server-only cluster (3 nodes)
- **nomad-single-machine**: All-in-one Nomad instance
- **consul-single-machine**: Standalone Consul server
- **vault-single-machine**: Standalone Vault server with init/unseal scripts
- **workstation**: Local development VM

Features:
- Cloud-init for VM initialization
- Reusable Terraform modules
- `Justfile` for task automation
- TLS certificate generation

#### Local Deployments (`lab/local/`)

Direct host-based Nomad configurations for rapid development:

- **nomad-all-in-one-dev**: Single agent (server + client) with persistence
- **nomad-one-server-one-client**: Minimal two-agent setup
- **nomad-server-cluster**: Three-server cluster
- **nomad-multi-region-dev**: Multi-region setup (us-east-1, europe-west-1, asia-south-1)

All include:
- Pre-configured ACL bootstrap tokens (for development only)
- HCL configuration files
- README with startup instructions

#### Shared Resources (`lab/shared/`)

Reusable components across all deployment types:

**Ansible Roles** (`ansible/roles/`):
- `consul`: Consul agent installation and configuration
- `vault`: Vault server setup
- `ceph`: Ceph storage cluster (cephadm, NVMe-oF)
- `tls`: TLS certificate generation
- `dnsmasq`: DNS configuration for service discovery

**Nomad Jobs** (`jobs/`):
- **Monitoring**: Prometheus, Grafana, Loki, Promtail, InfluxDB, cAdvisor, Node Exporter
- **Databases**: PostgreSQL, CockroachDB
- **Event Streaming**: Kafka with Kafka UI
- **Workflow Orchestration**: Temporal, Ray (head/worker)
- **Authentication**: Authentik
- **Autoscaling**: Nomad Autoscaler
- **Testing**: nginx, nomad-nodesim

**Terraform Modules** (`terraform/`):
- `aws-ami`: AMI selection for AWS
- `aws-compute`: EC2 instance provisioning
- `aws-network`: VPC and networking setup
- `multipass-compute`: Multipass VM provisioning
- `ansible-provision`: Ansible playbook execution
- `tls`: Certificate authority and certificate generation

### code/ - Development Code

Various code for exploration, benchmarking, and testing:

- **go/**: Go language code
  - `generic_bench_test.go`: Generic benchmark tests
  - `go.mod`: Go module definition

### tools/ - Utility Tools

Custom-built tools for development workflows:

- **ghr/**: GitHub Review CLI tool
  - Lists open PRs pending your review across multiple repositories
  - Classifies PRs as "required" or "optional" based on review status
  - Filters out PRs already reviewed by others with no new commits
  - Written in Go using GitHub API v85
  - Usage: `ghr -r owner/repo1 -r owner/repo2 -t $GITHUB_TOKEN`

- **gmd/**: Additional utility (Go module)

### workstation/ - Workstation Configuration

Personal workstation setup configurations for macOS and Linux machines.

## Key Technologies

### Infrastructure as Code
- **Terraform**: Infrastructure provisioning across AWS and Multipass
- **Ansible**: Configuration management with role-based architecture
- **Cloud-init**: VM initialization for Multipass instances

### HashiCorp Stack
- **Nomad**: Primary orchestration platform (v1.10.0+)
- **Consul**: Service discovery and configuration
- **Vault**: Secrets management with workload identity support

### Monitoring & Observability
- **Prometheus**: Metrics collection and alerting
- **Grafana**: Visualization and dashboards
- **Loki**: Log aggregation
- **InfluxDB**: Time-series database
- **cAdvisor**: Container metrics

### Storage & Networking
- **Ceph**: Distributed storage system
- **NVMe-oF**: NVMe over Fabrics
- **Consul Connect**: Service mesh
- **dnsmasq**: DNS forwarding for service discovery

## Deployment Patterns

### 1. Terraform → Ansible Pipeline
Infrastructure is provisioned via Terraform, then configured via Ansible in an automated pipeline:

```
Terraform (provision) → Ansible (configure) → Ready to use
```

### 2. TLS Everywhere
All deployments include automatic TLS certificate generation for secure communication between Nomad agents, Consul, and Vault.

### 3. Modular Design
Shared Terraform modules and Ansible roles enable:
- Code reusability across environments
- Consistent configuration
- Easy customization per deployment

### 4. Multi-Environment Support
Same configurations can be deployed to:
- AWS (production-like)
- Multipass (local VMs)
- Local host (rapid development)

## Getting Started

### Prerequisites

- **Terraform** (v1.0+)
- **Ansible** (v2.9+)
- **AWS CLI** (for AWS deployments)
- **Multipass** (for local VM deployments)
- **Nomad** (for local deployments)

### Quick Start - Multipass Nomad Cluster

```bash
cd lab/multipass/nomad-cluster
terraform init
terraform apply
```

This will:
1. Create 3 Nomad server VMs and 2 client VMs
2. Generate TLS certificates
3. Configure Nomad with Ansible
4. Output SSH commands and Nomad API endpoints

### Quick Start - Local Development

```bash
cd lab/local/nomad-all-in-one-dev
nomad agent -config=europe-west-1_nomad.hcl
```

In another terminal:
```bash
nomad acl bootstrap _nomad_root_token
```

### AWS Deployment

```bash
cd lab/aws/nomad-cluster
terraform init
terraform apply
```

Requires AWS credentials configured via environment variables or AWS CLI.

## Use Cases

- **Nomad Development**: Test Nomad features and configurations
- **Learning HashiCorp Stack**: Hands-on experience with Nomad, Consul, and Vault integration
- **Benchmarking**: Performance testing with various cluster configurations
- **Multi-Region Testing**: Experiment with multi-region orchestration patterns
- **Storage Integration**: Test persistent storage with Ceph and NVMe-oF
- **Service Mesh**: Explore Consul Connect and service mesh patterns
- **CI/CD Testing**: Validate deployment pipelines and job specifications

## Contributing

This is a personal development repository. Feel free to fork and adapt for your own use.

## License

See original repository for license information.

## Acknowledgments

Forked from [jrasell/dev-mess](https://github.com/jrasell/dev-mess) - thanks for the excellent foundation!
