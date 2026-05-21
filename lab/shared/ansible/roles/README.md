# Shared Ansible Roles

Reusable Ansible roles for configuring HashiCorp stack infrastructure and supporting services. These roles provide consistent, idempotent configuration management across all lab environments.

## Overview

This directory contains 14 Ansible roles organized by functionality:

| Role | Purpose | Key Features |
|------|---------|--------------|
| [**build**](#build) | Build Nomad from source | Development toolchain, Git sync, Make automation |
| [**ceph**](#ceph) | Deploy Ceph storage cluster | Cephadm bootstrap, NVMe-oF support |
| [**cni**](#cni) | Install CNI plugins | Network plugin management, Custom configs |
| [**common**](#common) | Base system configuration | Package installation, User management, MOTD |
| [**consul**](#consul) | Install/configure Consul | Service discovery, Health checks |
| [**dnsmasq**](#dnsmasq) | Configure DNS forwarding | Consul DNS integration |
| [**hashicorp_release**](#hashicorp_release) | Download HashiCorp binaries | Version management, Checksum verification |
| [**helper**](#helper) | Utility tasks | File operations, Package management, Systemd |
| [**nomad**](#nomad) | Install/configure Nomad | Orchestration, Driver plugins, TLS support |
| [**prometheus_node_exporter**](#prometheus_node_exporter) | Install node exporter | System metrics collection |
| [**smuggle**](#smuggle) | File transfer utility | Local-to-remote file copying |
| [**tls**](#tls) | Generate TLS certificates | CA creation, Self-signed certs |
| [**vault**](#vault) | Install/configure Vault | Secrets management |
| [**virt**](#virt) | Install virtualization tools | QEMU, libvirt, KVM |

## Quick Start

### Using a Role in a Playbook

```yaml
---
- name: Configure Nomad cluster
  hosts: nomad_servers
  become: true
  roles:
    - role: common
      vars:
        common_apt_packages:
          - jq
          - curl
          - vim
    
    - role: consul
      vars:
        consul_binary_version: "1.22.6"
        consul_config_template: "templates/consul-server.hcl.j2"
    
    - role: nomad
      vars:
        nomad_binary_version: "1.11.1"
        nomad_config_template: "templates/nomad-server.hcl.j2"
```

### Role Dependencies

Some roles depend on others:

```yaml
# Nomad and Consul roles use hashicorp_release internally
- role: nomad
  # Automatically includes hashicorp_release role

# Build role can sync code using helper role
- role: build
  vars:
    build_local_code_path: "/path/to/nomad/source"
```

## Role Reference

### build

Builds Nomad from source code for development and testing.

**Location**: [`build/`](build/)

**Purpose**: Set up development environment and compile Nomad binaries from source.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `build_apt_packages` | list | See below | Development packages to install |
| `build_nomad_code_path` | string | `/opt/nomad-code` | Path to Nomad source code |
| `build_dev_make_commands` | list | `["lint-deps", "bootstrap", "dev"]` | Make targets to execute |
| `build_local_code_path` | string | `""` | Local source path to sync |

**Default Packages**:
- `build-essential` - Compiler toolchain
- `curl` - HTTP client
- `git` - Version control
- `make` - Build automation
- `unzip` - Archive extraction

**Usage Example**:
```yaml
- role: build
  vars:
    build_local_code_path: "{{ lookup('env', 'HOME') }}/nomad"
    build_nomad_code_path: "/opt/nomad-code"
    build_dev_make_commands:
      - "lint-deps"
      - "bootstrap"
      - "dev"
```

**What It Does**:
1. Installs development packages
2. Marks Git directory as safe
3. Syncs local code to remote (if specified)
4. Checks for existing binary
5. Runs Make commands to build Nomad

**Use Cases**:
- Testing local Nomad changes
- Building custom Nomad versions
- Development workflow automation

---

### ceph

Deploys and configures Ceph storage clusters with NVMe-oF support.

**Location**: [`ceph/`](ceph/)

**Purpose**: Provide distributed storage for container workloads.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `ceph_version` | string | `18.2.2` | Ceph version to install |
| `ceph_cephadm_install_method` | string | `"none"` | Installation method |
| `ceph_cephadm_bootstrap` | bool | `false` | Bootstrap new cluster |
| `ceph_cephadm_bootstrap_hosts` | list | `[]` | Initial cluster hosts |
| `ceph_cephadm_bootstrap_mons` | list | `[]` | Monitor nodes |
| `ceph_cephadm_bootstrap_mgrs` | list | `[]` | Manager nodes |
| `ceph_cephadm_bootstrap_osd_hosts` | list | `[]` | OSD hosts |
| `ceph_cephadm_bootstrap_osd_paths` | list | `[]` | OSD device paths |
| `ceph_nvmeof_bootstrap` | bool | `false` | Enable NVMe-oF gateway |
| `ceph_nvmeof_bootstrap_pool_name` | string | `"default_0"` | NVMe-oF pool name |

**Usage Example**:
```yaml
- role: ceph
  vars:
    ceph_version: "18.2.2"
    ceph_cephadm_install_method: "download"
    ceph_cephadm_bootstrap: true
    ceph_cephadm_bootstrap_hosts:
      - "ceph-node-1"
      - "ceph-node-2"
      - "ceph-node-3"
    ceph_cephadm_bootstrap_mons:
      - "ceph-node-1"
    ceph_cephadm_bootstrap_osd_hosts:
      - "ceph-node-1"
      - "ceph-node-2"
      - "ceph-node-3"
    ceph_cephadm_bootstrap_osd_paths:
      - "/dev/sdb"
      - "/dev/sdc"
```

**Features**:
- Automated cluster bootstrapping
- NVMe-oF gateway configuration
- Multi-node OSD deployment
- Container-based deployment via cephadm

---

### cni

Installs and configures Container Network Interface (CNI) plugins.

**Location**: [`cni/`](cni/)

**Purpose**: Provide network connectivity for containerized workloads.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `cni_plugins_path` | string | `/opt/cni/bin` | Plugin binary directory |
| `cni_plugins_config_path` | string | `/opt/cni/config` | Configuration directory |
| `cni_plugins_version` | string | `1.9.0` | CNI plugins version |
| `cni_configs` | list | `[]` | CNI configuration files |

**Usage Example**:
```yaml
- role: cni
  vars:
    cni_plugins_version: "1.9.0"
    cni_configs:
      - name: "10-bridge.conf"
        content: |
          {
            "cniVersion": "1.0.0",
            "name": "bridge",
            "type": "bridge",
            "bridge": "cni0",
            "isGateway": true,
            "ipMasq": true,
            "ipam": {
              "type": "host-local",
              "subnet": "10.22.0.0/16",
              "routes": [
                { "dst": "0.0.0.0/0" }
              ]
            }
          }
```

**Supported Architectures**:
- `x86_64` → `amd64`
- `aarch64` → `arm64`

**Included Plugins**:
- bridge, host-device, ipvlan, macvlan
- loopback, ptp, vlan
- dhcp, host-local, static
- bandwidth, firewall, portmap, tuning

---

### common

Provides base system configuration for all hosts.

**Location**: [`common/`](common/)

**Purpose**: Standardize system setup across all nodes.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `common_apt_packages` | list | See below | Packages to install |
| `common_hostname` | string | `""` | Set hostname |
| `common_users` | list | `[]` | Users to create |

**Default Packages**:
- `jq` - JSON processor
- `net-tools` - Network utilities
- `ntp` - Time synchronization
- `unzip` - Archive extraction

**Usage Example**:
```yaml
- role: common
  vars:
    common_hostname: "nomad-server-01"
    common_apt_packages:
      - jq
      - curl
      - vim
      - htop
    common_users:
      - name: "deploy"
        groups: ["sudo"]
        shell: "/bin/bash"
```

**Features**:
- Package installation
- Hostname configuration
- User management
- Custom MOTD (Debian/Ubuntu)

---

### consul

Installs and configures HashiCorp Consul for service discovery.

**Location**: [`consul/`](consul/)

**Purpose**: Provide service mesh and service discovery capabilities.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `consul_user` | string | `consul` | Service user |
| `consul_group` | string | `consul` | Service group |
| `consul_binary_version` | string | `1.22.6` | Consul version |
| `consul_binary_checksum` | string | SHA256 hash | Binary checksum |
| `consul_config_dir` | string | `/etc/consul.d` | Configuration directory |
| `consul_config_template` | string | `""` | Config template path |
| `consul_data_dir` | string | `/opt/consul/data` | Data directory |

**Usage Example**:
```yaml
- role: consul
  vars:
    consul_binary_version: "1.22.6"
    consul_config_template: "{{ playbook_dir }}/templates/consul-server.hcl.j2"
```

**What It Does**:
1. Creates consul user and group
2. Downloads and installs Consul binary
3. Creates systemd service
4. Sets up configuration directories
5. Writes configuration from template
6. Enables and starts service

**Handlers**:
- `reload_systemd` - Reload systemd daemon
- `enable_consul` - Enable Consul service
- `restart_consul` - Restart Consul service

---

### dnsmasq

Configures dnsmasq for DNS forwarding to Consul.

**Location**: [`dnsmasq/`](dnsmasq/)

**Purpose**: Forward `.consul` domain queries to Consul DNS.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `dnsmasq_listen_address` | string | `127.0.0.1` | Listen address |
| `dnsmasq_listen_port` | int | `53` | Listen port |
| `dnsmasq_servers` | list | `[]` | Upstream DNS servers |

**Usage Example**:
```yaml
- role: dnsmasq
  vars:
    dnsmasq_listen_address: "0.0.0.0"
    dnsmasq_listen_port: 53
    dnsmasq_servers:
      - server: "/consul/127.0.0.1#8600"
      - server: "8.8.8.8"
```

**Configuration**:
- Forwards `.consul` queries to Consul (port 8600)
- Handles other queries via upstream DNS
- Supports Ubuntu-specific configuration

---

### hashicorp_release

Downloads and installs HashiCorp product binaries.

**Location**: [`hashicorp_release/`](hashicorp_release/)

**Purpose**: Centralized binary management for HashiCorp products.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `hashicorp_release_product_name` | string | `""` | Product name (nomad, consul, vault) |
| `hashicorp_release_product_version` | string | `""` | Version to install |
| `hashicorp_release_product_checksum` | string | `""` | SHA256 checksum |
| `hashicorp_release_product_install_dir` | string | `/usr/local/bin` | Installation directory |

**Architecture Mapping**:
- `amd64`, `x86_64` → `amd64`
- `aarch64` → `arm64`
- `armv7l` → `arm`
- `32-bit` → `386`

**Usage Example**:
```yaml
- role: hashicorp_release
  vars:
    hashicorp_release_product_name: "nomad"
    hashicorp_release_product_version: "1.11.1"
    hashicorp_release_product_checksum: "sha256:a120ba1be96d536ef7196911b57bfbe78fe08c53935dfd16eae0206eba09d729"
```

**Process**:
1. Constructs download URL
2. Downloads ZIP archive
3. Verifies checksum
4. Extracts binary
5. Installs to specified directory
6. Sets executable permissions

**Note**: This role is typically used internally by other roles (nomad, consul, vault) rather than directly in playbooks.

---

### helper

Provides utility tasks for common operations.

**Location**: [`helper/`](helper/)

**Purpose**: Reusable helper tasks for file operations, package management, and system configuration.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `helper_apt_packages` | list | `[]` | APT packages to install |
| `helper_yum_packages` | list | `[]` | YUM packages to install |
| `helper_sync_local_paths` | list | `[]` | Paths to sync |
| `helper_print_host_facts` | bool | `false` | Debug host facts |
| `helper_file_copy_local` | list | `[]` | Files to copy |
| `helper_file_write_template` | list | `[]` | Templates to render |
| `helper_file_write_content` | list | `[]` | Content to write |
| `helper_systemd_start_service_name` | string | `""` | Service to start |

**Usage Example**:
```yaml
- role: helper
  vars:
    helper_apt_packages:
      - curl
      - wget
    helper_file_copy_local:
      - src: "/local/path/config.json"
        dest: "/remote/path/config.json"
        mode: "0644"
    helper_file_write_content:
      - path: "/etc/app/config.txt"
        content: "key=value\n"
        mode: "0600"
    helper_systemd_start_service_name: "myapp"
```

**Features**:
- Cross-platform package management (APT/YUM)
- File synchronization
- Template rendering
- Content writing
- Systemd service management
- Host facts debugging

---

### nomad

Installs and configures HashiCorp Nomad orchestrator.

**Location**: [`nomad/`](nomad/)

**Purpose**: Deploy and configure Nomad for workload orchestration.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `nomad_user` | string | `root` | Service user |
| `nomad_group` | string | `root` | Service group |
| `nomad_binary_version` | string | `1.11.1` | Nomad version |
| `nomad_binary_checksum` | string | SHA256 hash | Binary checksum |
| `nomad_config_dir` | string | `/etc/nomad.d` | Configuration directory |
| `nomad_config_template` | string | `""` | Config template path |
| `nomad_data_dir` | string | `/opt/nomad/data` | Data directory |
| `nomad_plugin_dir` | string | `/opt/nomad/plugins` | Plugin directory |
| `nomad_tls_enabled` | bool | `false` | Enable TLS |
| `nomad_tls_ca_cert` | string | `""` | CA certificate path |
| `nomad_tls_cert` | string | `""` | Certificate path |
| `nomad_tls_cert_key` | string | `""` | Certificate key path |
| `nomad_license` | string | `""` | Enterprise license |
| `nomad_service_name` | string | `nomad` | Systemd service name |
| `nomad_driver_plugins` | list | `[]` | Driver plugins to install |

**Driver Plugin Configuration**:
```yaml
nomad_driver_plugins:
  - name: "nomad-driver-podman"
    version: "0.6.4"
    checksum: "sha256:abc123..."
    config:
      volumes:
        enabled: true
      gc:
        container: true
```

**Usage Example**:
```yaml
- role: nomad
  vars:
    nomad_binary_version: "1.11.1"
    nomad_config_template: "{{ playbook_dir }}/templates/nomad-client.hcl.j2"
    nomad_tls_enabled: true
    nomad_tls_ca_cert: "{{ playbook_dir }}/.tls/ca.pem"
    nomad_tls_cert: "{{ playbook_dir }}/.tls/client.pem"
    nomad_tls_cert_key: "{{ playbook_dir }}/.tls/client-key.pem"
    nomad_driver_plugins:
      - name: "nomad-driver-podman"
        version: "0.6.4"
```

**Features**:
- Version management with checksums
- TLS configuration
- Enterprise license support
- Driver plugin installation
- Multiple service instances (via `nomad_service_name`)
- Automatic service restart on config changes

**Handlers**:
- `reload_systemd` - Reload systemd daemon
- `enable_{{ nomad_service_name }}` - Enable service
- `restart_{{ nomad_service_name }}` - Restart service

---

### prometheus_node_exporter

Installs Prometheus Node Exporter for system metrics.

**Location**: [`prometheus_node_exporter/`](prometheus_node_exporter/)

**Purpose**: Collect and expose system-level metrics for monitoring.

**Usage Example**:
```yaml
- role: prometheus_node_exporter
```

**Features**:
- Systemd service configuration
- Automatic startup
- Metrics exposed on port 9100

---

### smuggle

Transfers files from local machine to remote hosts.

**Location**: [`smuggle/`](smuggle/)

**Purpose**: Simple file transfer utility for playbooks.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `smuggle_local_paths` | list | `[]` | Files to transfer |

**Usage Example**:
```yaml
- role: smuggle
  vars:
    smuggle_local_paths:
      - src: "/local/path/file.txt"
        dest: "/remote/path/file.txt"
        mode: "0644"
```

---

### tls

Generates TLS certificates for secure communication.

**Location**: [`tls/`](tls/)

**Purpose**: Create CA and self-signed certificates for development/testing.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `tls_path` | string | `{{ playbook_dir }}/.tls` | Certificate storage path |
| `tls_ca_generate` | bool | `true` | Generate CA certificate |
| `tls_ca_pem_filename` | string | `ca.pem` | CA certificate filename |
| `tls_ca_key_filename` | string | `ca-key.pem` | CA key filename |
| `tls_self_signed_generate` | list | `[]` | Certificates to generate |

**Certificate Configuration**:
```yaml
tls_self_signed_generate:
  - name: "nomad-server-0"
    common_name: "server.global.nomad"
    dns_names:
      - "server.global.nomad"
      - "localhost"
    ip_addresses:
      - "192.168.1.10"
      - "127.0.0.1"
    validity_hours: 8760  # 1 year
```

**Usage Example**:
```yaml
- role: tls
  vars:
    tls_path: "{{ playbook_dir }}/.tls"
    tls_ca_generate: true
    tls_self_signed_generate:
      - name: "nomad-server"
        common_name: "server.global.nomad"
        dns_names:
          - "server.global.nomad"
          - "localhost"
        ip_addresses:
          - "{{ ansible_default_ipv4.address }}"
          - "127.0.0.1"
```

**Generated Files**:
```
.tls/
├── ca.pem                    # CA certificate
├── ca-key.pem                # CA private key
├── nomad-server.pem          # Server certificate
└── nomad-server-key.pem      # Server private key
```

---

### vault

Installs and configures HashiCorp Vault for secrets management.

**Location**: [`vault/`](vault/)

**Purpose**: Deploy Vault for secure secret storage and access.

**Key Variables**:

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `vault_user` | string | `vault` | Service user |
| `vault_group` | string | `vault` | Service group |
| `vault_binary_version` | string | `1.21.4` | Vault version |
| `vault_binary_checksum` | string | SHA256 hash | Binary checksum |
| `vault_config_dir` | string | `/etc/vault.d` | Configuration directory |
| `vault_config_template` | string | `""` | Config template path |
| `vault_data_dir` | string | `/opt/vault/data` | Data directory |

**Usage Example**:
```yaml
- role: vault
  vars:
    vault_binary_version: "1.21.4"
    vault_config_template: "{{ playbook_dir }}/templates/vault-server.hcl.j2"
```

**Features**:
- User/group creation
- Binary installation
- Systemd service configuration
- Configuration management

---

### virt

Installs virtualization tools (QEMU, libvirt, KVM).

**Location**: [`virt/`](virt/)

**Purpose**: Enable virtualization capabilities on hosts.

**Usage Example**:
```yaml
- role: virt
```

**Installed Components**:
- QEMU
- libvirt
- KVM
- virt-manager (optional)

---

## Development Guide

### Creating a New Role

1. **Generate role structure**:
```bash
cd lab/shared/ansible/roles
ansible-galaxy init my_role
```

2. **Define variables** (`defaults/main.yaml`):
```yaml
my_role_version: "1.0.0"
my_role_config_dir: "/etc/my-app"
my_role_enabled: true
```

3. **Implement tasks** (`tasks/main.yaml`):
```yaml
- name: "install_package"
  become: true
  ansible.builtin.apt:
    name: "my-package"
    state: present

- name: "create_config"
  become: true
  ansible.builtin.template:
    src: "config.j2"
    dest: "{{ my_role_config_dir }}/config.yaml"
  notify: "restart_service"
```

4. **Add handlers** (`handlers/main.yaml`):
```yaml
- name: "restart_service"
  become: true
  ansible.builtin.systemd:
    name: "my-service"
    state: restarted
```

5. **Create templates** (`templates/config.j2`):
```jinja
# {{ ansible_managed }}
version: {{ my_role_version }}
enabled: {{ my_role_enabled }}
```

### Role Design Principles

1. **Idempotency**: Roles should be safe to run multiple times
2. **Defaults**: Provide sensible defaults for all variables
3. **Flexibility**: Allow customization via variables
4. **Documentation**: Document all variables and usage
5. **Handlers**: Use handlers for service restarts
6. **Become**: Use `become: true` for privileged operations

### Testing Roles

```bash
# Syntax check
ansible-playbook --syntax-check playbook.yaml

# Dry run
ansible-playbook --check playbook.yaml

# Run with verbosity
ansible-playbook -vvv playbook.yaml

# Run specific role
ansible-playbook playbook.yaml --tags "my_role"
```

## Common Patterns

### Pattern 1: HashiCorp Stack Deployment

```yaml
- name: Deploy HashiCorp stack
  hosts: cluster
  become: true
  roles:
    - role: common
      vars:
        common_apt_packages: [jq, curl, unzip]
    
    - role: consul
      vars:
        consul_binary_version: "1.22.6"
        consul_config_template: "templates/consul.hcl.j2"
    
    - role: nomad
      vars:
        nomad_binary_version: "1.11.1"
        nomad_config_template: "templates/nomad.hcl.j2"
    
    - role: vault
      vars:
        vault_binary_version: "1.21.4"
        vault_config_template: "templates/vault.hcl.j2"
```

### Pattern 2: Secure Cluster with TLS

```yaml
- name: Generate TLS certificates
  hosts: localhost
  roles:
    - role: tls
      vars:
        tls_ca_generate: true
        tls_self_signed_generate:
          - name: "server-{{ item }}"
            common_name: "server.global.nomad"
            dns_names: ["server.global.nomad"]
            ip_addresses: ["{{ hostvars[item].ansible_host }}"]
      loop: "{{ groups['servers'] }}"

- name: Deploy with TLS
  hosts: servers
  become: true
  roles:
    - role: nomad
      vars:
        nomad_tls_enabled: true
        nomad_tls_ca_cert: ".tls/ca.pem"
        nomad_tls_cert: ".tls/server-{{ inventory_hostname }}.pem"
        nomad_tls_cert_key: ".tls/server-{{ inventory_hostname }}-key.pem"
```

### Pattern 3: Development Build

```yaml
- name: Build and deploy Nomad from source
  hosts: dev_servers
  become: true
  roles:
    - role: build
      vars:
        build_local_code_path: "{{ lookup('env', 'HOME') }}/nomad"
        build_nomad_code_path: "/opt/nomad-code"
    
    - role: nomad
      vars:
        nomad_binary_version: "dev"
        nomad_config_template: "templates/nomad-dev.hcl.j2"
```

### Pattern 4: Storage Cluster

```yaml
- name: Deploy Ceph storage
  hosts: storage
  become: true
  roles:
    - role: common
    
    - role: ceph
      vars:
        ceph_cephadm_bootstrap: true
        ceph_cephadm_bootstrap_hosts: "{{ groups['storage'] }}"
        ceph_cephadm_bootstrap_osd_hosts: "{{ groups['storage'] }}"
        ceph_cephadm_bootstrap_osd_paths:
          - "/dev/sdb"
          - "/dev/sdc"
```

## Troubleshooting

### Role Not Found

```bash
# Error: Role 'my_role' not found
ERROR! the role 'my_role' was not found

# Solution: Check role path
ls -la lab/shared/ansible/roles/my_role

# Verify ansible.cfg
cat ansible.cfg | grep roles_path
```

### Variable Not Defined

```bash
# Error: Undefined variable
fatal: [host]: FAILED! => {"msg": "The task includes an option with an undefined variable"}

# Solution: Check defaults/main.yaml
cat roles/my_role/defaults/main.yaml

# Or provide variable in playbook
vars:
  my_variable: "value"
```

### Handler Not Triggered

```bash
# Handlers only run if task reports 'changed'
# Check task status
TASK [my_role : update_config] ***
changed: [host]

# Verify handler name matches
notify: "restart_service"  # Must match handler name exactly
```

### Permission Denied

```bash
# Error: Permission denied
fatal: [host]: FAILED! => {"msg": "Permission denied"}

# Solution: Add become
- name: "privileged_task"
  become: true
  ansible.builtin.file:
    path: "/etc/app"
    state: directory
```

## Best Practices

### 1. Use Role Defaults

✅ **Good**:
```yaml
# defaults/main.yaml
my_role_version: "1.0.0"
my_role_enabled: true
```

❌ **Bad**:
```yaml
# tasks/main.yaml
- name: "install"
  apt:
    name: "package-1.0.0"  # Hardcoded version
```

### 2. Parameterize Templates

✅ **Good**:
```jinja
# templates/config.j2
version: {{ my_role_version }}
enabled: {{ my_role_enabled }}
```

❌ **Bad**:
```jinja
# templates/config.j2
version: 1.0.0
enabled: true
```

### 3. Use Handlers for Restarts

✅ **Good**:
```yaml
- name: "update_config"
  template:
    src: "config.j2"
    dest: "/etc/app/config"
  notify: "restart_service"
```

❌ **Bad**:
```yaml
- name: "update_config"
  template:
    src: "config.j2"
    dest: "/etc/app/config"

- name: "restart_service"
  systemd:
    name: "app"
    state: restarted
```

### 4. Check Before Install

✅ **Good**:
```yaml
- name: "check_version"
  command: "/usr/local/bin/app version"
  register: installed_version
  ignore_errors: true
  changed_when: false

- name: "install_binary"
  when: installed_version.rc != 0 or desired_version not in installed_version.stdout
```

### 5. Document Variables

✅ **Good**:
```yaml
# defaults/main.yaml

# The version of the application to install
my_role_version: "1.0.0"

# Whether to enable the service on boot
my_role_enabled: true

# Configuration directory path
my_role_config_dir: "/etc/my-app"
```

## Version Management

### Updating HashiCorp Products

When updating Nomad, Consul, or Vault versions:

1. **Update version and checksum**:
```yaml
# roles/nomad/defaults/main.yaml
nomad_binary_version: "1.11.2"  # New version
nomad_binary_checksum: "sha256:new_checksum_here"
```

2. **Get checksum from releases page**:
```bash
# Visit https://releases.hashicorp.com/nomad/1.11.2/
# Copy SHA256 checksum for your platform
```

3. **Test in development first**:
```bash
ansible-playbook -i inventory/dev playbook.yaml --check
```

4. **Document changes**:
```markdown
## Changelog
- Updated Nomad to 1.11.2
- Updated Consul to 1.22.7
```

## Contributing

When modifying roles:

1. **Test thoroughly**: Verify changes don't break existing playbooks
2. **Update defaults**: Keep default values current
3. **Document changes**: Update role README and variable descriptions
4. **Maintain compatibility**: Avoid breaking changes when possible
5. **Add examples**: Show how to use new features
6. **Consider impact**: Changes affect all configurations using the role

## Related Documentation

- [Ansible Role Documentation](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [HashiCorp Nomad](https://www.nomadproject.io/docs)
- [HashiCorp Consul](https://www.consul.io/docs)
- [HashiCorp Vault](https://www.vaultproject.io/docs)