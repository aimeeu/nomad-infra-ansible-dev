# Vault Single Machine

A standalone HashiCorp Vault server for secrets management, ideal for integration with Nomad workload identity and testing Vault features.

## Overview

This configuration deploys a single Vault server suitable for:
- Secrets management for Nomad workloads
- Testing Vault integration with Nomad
- Learning Vault features
- Development environments
- Workload identity testing

## Architecture

- **1 Vault Server**: Single-node Vault server
- **VM Resources**: 1 CPU, 1GiB RAM
- **Features**:
  - Vault UI enabled
  - File storage backend
  - Development mode (not for production!)
  - HTTP (not HTTPS) for simplicity

## What the Terraform Does

[`main.tf`](main.tf:1) provisions the Vault server:

1. **Creates Vault Server VM** (via `multipass-compute` module):
   - Instance: `vault-server-<random-id>`
   - Specs: 1 CPU, 1GiB RAM
   - Ansible group: `vault_server`

2. **Runs Ansible Provisioning** (via `ansible-provision` module):
   - Executes [`playbook_all.yaml`](playbook_all.yaml:1)
   - Configures Vault server

3. **Outputs**:
   - Initialization script command
   - SSH command
   - Vault UI URL
   - Vault server address

## What the Ansible Playbook Does

### [`playbook_vault_server.yaml`](playbook_vault_server.yaml:1)

Configures the Vault server:

1. **Common Role**: Base system setup
   - Sets hostname
   - Installs: build-essential, git, jq, make, net-tools, unzip

2. **Vault Role**: Installs and configures Vault
   - Version: 2.0.0
   - Binary checksum verification
   - Config directory: `/etc/vault.d/`
   - Data directory: `/opt/vault/data/`
   - Config template: [`vault_server.hcl.j2`](templates/vault_server.hcl.j2:1)

## Initialization Script

The [`init_unseal.sh`](scripts/init_unseal.sh:1) script automates Vault initialization:

**What it does**:
1. Checks if Vault is already initialized
2. If not initialized:
   - Runs `vault operator init`
   - Uses 1 key share with threshold of 1 (dev mode)
   - Saves output to `generated_vault_init.json`
3. Unseals Vault automatically

**Output file contains**:
- Root token
- Unseal key
- Recovery keys (if applicable)

## Prerequisites

- **Multipass**: Install from [multipass.run](https://multipass.run)
- **Terraform**: v1.0 or later
- **Ansible**: v2.9 or later
- **SSH Key**: `~/.ssh/id_rsa.pub` must exist
- **Vault CLI**: Install locally for initialization

## Deployment Instructions

### 1. Deploy Vault

```bash
cd lab/multipass/vault-single-machine
terraform init
terraform apply
```

### 2. Initialize and Unseal Vault

After deployment, Terraform outputs the initialization command:

```bash
# Run the initialization script
./scripts/init_unseal.sh --addr 192.168.64.X

# This creates generated_vault_init.json with:
# - root_token
# - unseal_keys_b64
```

### 3. Set Vault Token

```bash
# Export root token for CLI access
export VAULT_TOKEN="$(jq -r .root_token generated_vault_init.json)"
export VAULT_ADDR="http://192.168.64.X:8200"
```

### 4. Verify Vault

```bash
# Check status
vault status

# List auth methods
vault auth list

# List secrets engines
vault secrets list
```

### 5. Access Vault UI

Open browser to: `http://192.168.64.X:8200`

Login with the root token from `generated_vault_init.json`.

## Integration with Nomad

### 1. Enable JWT Auth for Nomad

```bash
# Enable JWT auth backend
vault auth enable -path=nomad-lcy1 jwt

# Configure JWT auth
vault write auth/nomad-lcy1/config \
  bound_issuer="https://nomad.example.com" \
  jwks_url="https://nomad-server-ip:4646/.well-known/jwks.json" \
  default_role="nomad-workloads"

# Create a role for Nomad workloads
vault write auth/nomad-lcy1/role/nomad-workloads \
  role_type="jwt" \
  bound_audiences="vault.io" \
  user_claim="/nomad_job_id" \
  user_claim_json_pointer=true \
  claim_mappings="/nomad_namespace"="nomad_namespace" \
  claim_mappings="/nomad_job_id"="nomad_job_id" \
  claim_mappings="/nomad_task"="nomad_task" \
  token_type="service" \
  token_policies="nomad-workloads" \
  token_period="30m" \
  token_explicit_max_ttl="1h"
```

### 2. Create Policies for Nomad Workloads

```bash
# Create a policy for app secrets
vault policy write app-policy - <<EOF
path "secret/data/app/*" {
  capabilities = ["read"]
}
EOF
```

### 3. Store Secrets

```bash
# Enable KV v2 secrets engine (if not already enabled)
vault secrets enable -path=secret kv-v2

# Store application secrets
vault kv put secret/app/database \
  username="dbuser" \
  password="secretpassword"
```

### 4. Configure Nomad to Use Vault

In your Nomad cluster configuration, add:

```hcl
vault {
  enabled               = true
  address               = "http://192.168.64.X:8200"
  jwt_auth_backend_path = "nomad-lcy1"
}
```

### 5. Use Vault in Nomad Jobs

```hcl
job "app" {
  group "web" {
    task "server" {
      vault {
        policies = ["app-policy"]
      }
      
      template {
        data = <<EOH
{{ with secret "secret/data/app/database" }}
DB_USER={{ .Data.data.username }}
DB_PASS={{ .Data.data.password }}
{{ end }}
EOH
        destination = "secrets/db.env"
        env         = true
      }
      
      driver = "docker"
      config {
        image = "myapp:latest"
      }
    }
  }
}
```

## Usage Examples

### Store and Retrieve Secrets

```bash
# Store a secret
vault kv put secret/myapp/config \
  api_key="abc123" \
  db_password="secret"

# Retrieve a secret
vault kv get secret/myapp/config

# Get specific field
vault kv get -field=api_key secret/myapp/config
```

### Create Dynamic Database Credentials

```bash
# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="readonly" \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/mydb" \
  username="vault" \
  password="vaultpass"

# Create a role
vault write database/roles/readonly \
  db_name=postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Generate credentials
vault read database/creds/readonly
```

### Enable PKI for TLS Certificates

```bash
# Enable PKI secrets engine
vault secrets enable pki

# Configure PKI
vault secrets tune -max-lease-ttl=87600h pki

# Generate root CA
vault write pki/root/generate/internal \
  common_name="example.com" \
  ttl=87600h

# Configure CA and CRL URLs
vault write pki/config/urls \
  issuing_certificates="http://192.168.64.X:8200/v1/pki/ca" \
  crl_distribution_points="http://192.168.64.X:8200/v1/pki/crl"

# Create a role
vault write pki/roles/example-dot-com \
  allowed_domains="example.com" \
  allow_subdomains=true \
  max_ttl="720h"

# Issue a certificate
vault write pki/issue/example-dot-com \
  common_name="app.example.com" \
  ttl="24h"
```

## Troubleshooting

### VM Not Starting
```bash
multipass list
multipass info vault-server-*
```

### Vault Not Running
```bash
ssh 192.168.64.X "sudo systemctl status vault"
ssh 192.168.64.X "sudo journalctl -u vault -f"
```

### Vault Sealed
```bash
# Check status
vault status

# Unseal manually
vault operator unseal $(jq -r .unseal_keys_b64[0] generated_vault_init.json)
```

### Cannot Access UI
```bash
# Verify Vault is listening
ssh 192.168.64.X "curl http://localhost:8200/v1/sys/health"

# Check firewall
ssh 192.168.64.X "sudo ufw status"
```

### Lost Root Token
```bash
# If you lost the token, check the generated file
cat generated_vault_init.json | jq -r .root_token
```

## Security Considerations

⚠️ **This configuration is for development/testing only!**

**Not suitable for production because**:
- HTTP only (no TLS)
- Single unseal key
- File storage backend (not HA)
- Root token stored in plain text
- No audit logging configured

**For production, you should**:
- Enable TLS
- Use multiple unseal keys with threshold
- Use a proper storage backend (Consul, Raft, etc.)
- Implement proper key management
- Enable audit logging
- Use auto-unseal with cloud KMS
- Implement proper access policies

## Cleanup

```bash
terraform destroy
```

## Configuration Files

- [`main.tf`](main.tf:1): Terraform infrastructure definition
- [`ansible.cfg`](ansible.cfg:1): Ansible configuration
- [`inventory.yaml`](inventory.yaml:1): Generated Ansible inventory
- [`playbook_all.yaml`](playbook_all.yaml:1): Master playbook
- [`playbook_vault_server.yaml`](playbook_vault_server.yaml:1): Vault configuration
- [`templates/vault_server.hcl.j2`](templates/vault_server.hcl.j2:1): Vault config template
- [`scripts/init_unseal.sh`](scripts/init_unseal.sh:1): Initialization script

## Next Steps

- **Integrate with Nomad**: Use with [`nomad-cluster`](../nomad-cluster/) for workload identity
- **Store Secrets**: Add application secrets for your workloads
- **Enable Auth Methods**: Configure LDAP, OIDC, or other auth methods
- **Dynamic Secrets**: Set up database credential generation
- **PKI**: Use Vault as a certificate authority

## Related Configurations

- [`nomad-cluster`](../nomad-cluster/): Nomad cluster with Vault integration
- [`consul-single-machine`](../consul-single-machine/): Consul for service discovery
- [`nomad-dev-cluster`](../nomad-dev-cluster/): Development Nomad cluster