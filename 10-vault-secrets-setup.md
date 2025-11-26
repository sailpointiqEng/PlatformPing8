# Vault Secrets Setup Guide

## Overview

This document provides step-by-step instructions for creating all required secrets in HashiCorp Vault before executing the Ping Identity Platform 8 deployment. All secrets must exist in Vault before any deployment playbook can run successfully.

## Prerequisites

Before creating secrets in Vault, ensure:

- [ ] HashiCorp Vault is installed and running
- [ ] Vault CLI is installed and configured
- [ ] You have appropriate Vault permissions (write access to `secret/ping/platform8/*`)
- [ ] Vault authentication is configured (user token or AppRole)
- [ ] KV secrets engine is enabled at `secret/` path

## Vault Authentication

### Option 1: User Token Authentication

```bash
# Login to Vault
vault auth -method=userpass username=<your_username>

# Or use existing token
export VAULT_TOKEN=<your_token>
export VAULT_ADDR=https://vault.example.com:8200
```

### Option 2: AppRole Authentication (Recommended for Automation)

```bash
# Login using AppRole (for automation)
vault write auth/approle/login \
  role_id=<role_id> \
  secret_id=<secret_id>
```

## Enable KV Secrets Engine

If KV secrets engine is not already enabled:

```bash
# Enable KV v2 secrets engine
vault secrets enable -path=secret kv-v2

# Verify
vault secrets list
```

## Create Vault Secrets

### Step 1: Create DS Secrets

#### 1.1 DS Admin Password

**Purpose**: Password for DS Directory Manager (root administrator)

**Command**:
```bash
vault kv put secret/ping/platform8/ds/admin_password \
  value="<your_secure_password>"
```

**Password Requirements**:
- Minimum 12 characters
- Mix of uppercase, lowercase, numbers, special characters
- Example: `DS_Admin_Pass123!`

**Verification**:
```bash
vault kv get secret/ping/platform8/ds/admin_password
```

---

#### 1.2 DS Deployment ID

**Purpose**: Unique deployment ID for DS certificate management

**Command**:
```bash
vault kv put secret/ping/platform8/ds/deployment_id \
  value="AX8fkJybs4nP3qfAXN3CMw4BCYspGQ5CBVN1bkVDAOgwVKjG2Wo2ZTs"
```

**Note**: Use the same deployment ID across all environments for consistency, or generate a unique one per environment.

**Verification**:
```bash
vault kv get secret/ping/platform8/ds/deployment_id
```

---

#### 1.3 DS Deployment ID Password

**Purpose**: Password for DS deployment ID (used for certificate operations)

**Command**:
```bash
vault kv put secret/ping/platform8/ds/deployment_id_password \
  value="<your_secure_password>"
```

**Password Requirements**:
- Minimum 12 characters
- Mix of uppercase, lowercase, numbers, special characters
- Example: `DeploymentID_Pass456!`

**Verification**:
```bash
vault kv get secret/ping/platform8/ds/deployment_id_password
```

---

#### 1.4 DS Monitor Password

**Purpose**: Password for DS monitor user (read-only monitoring)

**Command**:
```bash
vault kv put secret/ping/platform8/ds/monitor_password \
  value="<your_secure_password>"
```

**Password Requirements**:
- Minimum 12 characters
- Mix of uppercase, lowercase, numbers, special characters
- Example: `Monitor_Pass789!`

**Verification**:
```bash
vault kv get secret/ping/platform8/ds/monitor_password
```

---

### Step 2: Create AM Secrets

#### 2.1 AM Admin Password

**Purpose**: Password for AM amadmin account (primary AM administrator)

**Command**:
```bash
vault kv put secret/ping/platform8/am/admin_password \
  value="<your_secure_password>"
```

**Password Requirements**:
- Minimum 12 characters
- Mix of uppercase, lowercase, numbers, special characters
- Example: `AM_Admin_Pass123!`

**Note**: This password is also used for:
- AM Config Store admin account
- AM CTS admin account
- AM Identity Store bind account

**Verification**:
```bash
vault kv get secret/ping/platform8/am/admin_password
```

---

#### 2.2 Amster Key Passphrase

**Purpose**: Passphrase for Amster SSH key encryption

**Command**:
```bash
vault kv put secret/ping/platform8/am/amster_key_passphrase \
  value="<your_secure_passphrase>"
```

**Passphrase Requirements**:
- Minimum 12 characters
- Mix of uppercase, lowercase, numbers, special characters
- Example: `Amster_Key_Pass456!`

**Verification**:
```bash
vault kv get secret/ping/platform8/am/amster_key_passphrase
```

---

### Step 3: Create IDM Secrets

#### 3.1 IDM Admin Password

**Purpose**: Password for IDM openidm-admin account

**Command**:
```bash
vault kv put secret/ping/platform8/idm/admin_password \
  value="<your_secure_password>"
```

**Password Requirements**:
- Minimum 12 characters
- Mix of uppercase, lowercase, numbers, special characters
- Example: `IDM_Admin_Pass123!`

**Verification**:
```bash
vault kv get secret/ping/platform8/idm/admin_password
```

---

#### 3.2 IDM AD Connector Password

**Purpose**: Password for AD connector bind account (same as AD bind password)

**Command**:
```bash
vault kv put secret/ping/platform8/idm/ad_connector_password \
  value="<your_secure_password>"
```

**Note**: This should be the same value as `secret/ping/platform8/ad/bind_password`

**Verification**:
```bash
vault kv get secret/ping/platform8/idm/ad_connector_password
```

---

### Step 4: Create AD Secrets (If AD Integration Enabled)

#### 4.1 AD Bind DN

**Purpose**: Distinguished Name of AD service account for IDM connector

**Command**:
```bash
vault kv put secret/ping/platform8/ad/bind_dn \
  value="CN=IDM Service Account,OU=Service Accounts,DC=example,DC=com"
```

**Note**: Replace with your actual AD service account DN

**Verification**:
```bash
vault kv get secret/ping/platform8/ad/bind_dn
```

---

#### 4.2 AD Bind Password

**Purpose**: Password for AD service account

**Command**:
```bash
vault kv put secret/ping/platform8/ad/bind_password \
  value="<your_secure_password>"
```

**Password Requirements**:
- Must meet AD password policy requirements
- Minimum 12 characters (or as per AD policy)
- Mix of uppercase, lowercase, numbers, special characters
- Example: `AD_Bind_Pass456!`

**Note**: This account must exist in AD before deployment

**Verification**:
```bash
vault kv get secret/ping/platform8/ad/bind_password
```

---

#### 4.3 AD Base DN

**Purpose**: Base DN for user/group search in AD

**Command**:
```bash
vault kv put secret/ping/platform8/ad/base_dn \
  value="DC=example,DC=com"
```

**Note**: Replace with your actual AD domain DN

**Verification**:
```bash
vault kv get secret/ping/platform8/ad/base_dn
```

---

### Step 5: Create Common Secrets

#### 5.1 Truststore Password

**Purpose**: Password for Java truststore (default is "changeit")

**Command**:
```bash
vault kv put secret/ping/platform8/common/truststore_password \
  value="changeit"
```

**Note**: You can use the default "changeit" or set a custom password

**Verification**:
```bash
vault kv get secret/ping/platform8/common/truststore_password
```

---

#### 5.2 Keystore Password (Optional)

**Purpose**: Password for custom keystores (if using custom keystores)

**Command**:
```bash
vault kv put secret/ping/platform8/common/keystore_password \
  value="<your_secure_password>"
```

**Note**: Only required if using custom keystores

**Verification**:
```bash
vault kv get secret/ping/platform8/common/keystore_password
```

---

## Batch Creation Script

For convenience, you can create all secrets at once using a script:

**File**: `scripts/create-vault-secrets.sh`

```bash
#!/bin/bash

# Set Vault address
export VAULT_ADDR="https://vault.example.com:8200"

# Authenticate (adjust method as needed)
# vault auth -method=userpass username=<your_username>

# DS Secrets
vault kv put secret/ping/platform8/ds/admin_password value="DS_Admin_Pass123!"
vault kv put secret/ping/platform8/ds/deployment_id value="AX8fkJybs4nP3qfAXN3CMw4BCYspGQ5CBVN1bkVDAOgwVKjG2Wo2ZTs"
vault kv put secret/ping/platform8/ds/deployment_id_password value="DeploymentID_Pass456!"
vault kv put secret/ping/platform8/ds/monitor_password value="Monitor_Pass789!"

# AM Secrets
vault kv put secret/ping/platform8/am/admin_password value="AM_Admin_Pass123!"
vault kv put secret/ping/platform8/am/amster_key_passphrase value="Amster_Key_Pass456!"

# IDM Secrets
vault kv put secret/ping/platform8/idm/admin_password value="IDM_Admin_Pass123!"
vault kv put secret/ping/platform8/idm/ad_connector_password value="AD_Bind_Pass456!"

# AD Secrets (if AD integration enabled)
vault kv put secret/ping/platform8/ad/bind_dn value="CN=IDM Service Account,OU=Service Accounts,DC=example,DC=com"
vault kv put secret/ping/platform8/ad/bind_password value="AD_Bind_Pass456!"
vault kv put secret/ping/platform8/ad/base_dn value="DC=example,DC=com"

# Common Secrets
vault kv put secret/ping/platform8/common/truststore_password value="changeit"
vault kv put secret/ping/platform8/common/keystore_password value="Keystore_Pass123!"

echo "All secrets created successfully!"
```

**Usage**:
```bash
chmod +x scripts/create-vault-secrets.sh
./scripts/create-vault-secrets.sh
```

**Important**: Update all password values in the script before running!

---

## Verification

### Verify All Secrets Exist

**Script**: `scripts/verify-vault-secrets.sh`

```bash
#!/bin/bash

# Set Vault address
export VAULT_ADDR="https://vault.example.com:8200"

# List of required secrets
secrets=(
  "secret/ping/platform8/ds/admin_password"
  "secret/ping/platform8/ds/deployment_id"
  "secret/ping/platform8/ds/deployment_id_password"
  "secret/ping/platform8/ds/monitor_password"
  "secret/ping/platform8/am/admin_password"
  "secret/ping/platform8/am/amster_key_passphrase"
  "secret/ping/platform8/idm/admin_password"
  "secret/ping/platform8/idm/ad_connector_password"
  "secret/ping/platform8/ad/bind_dn"
  "secret/ping/platform8/ad/bind_password"
  "secret/ping/platform8/ad/base_dn"
  "secret/ping/platform8/common/truststore_password"
)

echo "Verifying Vault secrets..."
failed=0

for secret in "${secrets[@]}"; do
  if vault kv get "$secret" > /dev/null 2>&1; then
    echo "✓ $secret exists"
  else
    echo "✗ $secret MISSING"
    failed=1
  fi
done

if [ $failed -eq 0 ]; then
  echo "All secrets verified successfully!"
  exit 0
else
  echo "Some secrets are missing. Please create them before deployment."
  exit 1
fi
```

**Usage**:
```bash
chmod +x scripts/verify-vault-secrets.sh
./scripts/verify-vault-secrets.sh
```

---

## Integration with Deployment Process

### Pre-Deployment Validation

The deployment process includes a validation step to verify all Vault secrets exist before deployment begins.

**Playbook**: `playbooks/validate-vault-secrets.yml`

**File**: `playbooks/validate-vault-secrets.yml`

```yaml
---
- name: Validate Vault Secrets
  hosts: localhost
  gather_facts: no
  vars:
    vault_secrets:
      - path: "secret/ping/platform8/ds/admin_password"
        description: "DS Admin Password"
      - path: "secret/ping/platform8/ds/deployment_id"
        description: "DS Deployment ID"
      - path: "secret/ping/platform8/ds/deployment_id_password"
        description: "DS Deployment ID Password"
      - path: "secret/ping/platform8/ds/monitor_password"
        description: "DS Monitor Password"
      - path: "secret/ping/platform8/am/admin_password"
        description: "AM Admin Password"
      - path: "secret/ping/platform8/am/amster_key_passphrase"
        description: "Amster Key Passphrase"
      - path: "secret/ping/platform8/idm/admin_password"
        description: "IDM Admin Password"
      - path: "secret/ping/platform8/idm/ad_connector_password"
        description: "IDM AD Connector Password"
      - path: "secret/ping/platform8/ad/bind_dn"
        description: "AD Bind DN"
        required: "{{ ad_integration_enabled | default(false) }}"
      - path: "secret/ping/platform8/ad/bind_password"
        description: "AD Bind Password"
        required: "{{ ad_integration_enabled | default(false) }}"
      - path: "secret/ping/platform8/ad/base_dn"
        description: "AD Base DN"
        required: "{{ ad_integration_enabled | default(false) }}"
      - path: "secret/ping/platform8/common/truststore_password"
        description: "Truststore Password"

  tasks:
    - name: Verify Vault connectivity
      uri:
        url: "{{ vault_addr }}/v1/sys/health"
        method: GET
      register: vault_health
      failed_when: vault_health.status != 200

    - name: Verify Vault secrets exist
      uri:
        url: "{{ vault_addr }}/v1/{{ item.path }}"
        method: GET
        headers:
          X-Vault-Token: "{{ vault_token }}"
        status_code: [200, 404]
      register: secret_check
      loop: "{{ vault_secrets }}"
      when: item.required | default(true)

    - name: Report missing secrets
      fail:
        msg: "Secret {{ item.item.path }} is missing: {{ item.item.description }}"
      loop: "{{ secret_check.results }}"
      when:
        - item.required | default(true)
        - item.status != 200
```

**Execution**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/validate-vault-secrets.yml
```

### Integration with Main Validation Playbook

The Vault secrets validation is included in the main pre-deployment validation playbook.

**File**: `playbooks/validate-prerequisites.yml`

```yaml
---
- name: Pre-Deployment Validation
  hosts: localhost
  gather_facts: yes
  tasks:
    - name: Include Vault secrets validation
      include_tasks: validate-vault-secrets.yml

    - name: Include connectivity validation
      include_role:
        name: validation
        tasks_from: connectivity.yml

    - name: Include port validation
      include_role:
        name: validation
        tasks_from: ports.yml

    # ... other validation tasks
```

---

## Pre-Installation Checklist

Before executing any deployment playbooks, complete this checklist:

### Vault Setup

- [ ] Vault is installed and running
- [ ] Vault CLI is installed and configured
- [ ] KV secrets engine is enabled at `secret/` path
- [ ] Vault authentication is configured
- [ ] Vault AppRole is created (for automation)
- [ ] AppRole credentials are obtained (role_id, secret_id)

### Secrets Creation

- [ ] All DS secrets are created
  - [ ] `secret/ping/platform8/ds/admin_password`
  - [ ] `secret/ping/platform8/ds/deployment_id`
  - [ ] `secret/ping/platform8/ds/deployment_id_password`
  - [ ] `secret/ping/platform8/ds/monitor_password`

- [ ] All AM secrets are created
  - [ ] `secret/ping/platform8/am/admin_password`
  - [ ] `secret/ping/platform8/am/amster_key_passphrase`

- [ ] All IDM secrets are created
  - [ ] `secret/ping/platform8/idm/admin_password`
  - [ ] `secret/ping/platform8/idm/ad_connector_password`

- [ ] All AD secrets are created (if AD integration enabled)
  - [ ] `secret/ping/platform8/ad/bind_dn`
  - [ ] `secret/ping/platform8/ad/bind_password`
  - [ ] `secret/ping/platform8/ad/base_dn`

- [ ] All common secrets are created
  - [ ] `secret/ping/platform8/common/truststore_password`
  - [ ] `secret/ping/platform8/common/keystore_password` (if using custom keystores)

### Verification

- [ ] Run verification script: `./scripts/verify-vault-secrets.sh`
- [ ] Run Ansible validation: `ansible-playbook playbooks/validate-vault-secrets.yml`
- [ ] All secrets are accessible from Ansible control node
- [ ] Vault connectivity is tested

---

## Security Best Practices

### Password Generation

Use strong, unique passwords for each secret:

```bash
# Generate random password (Linux)
openssl rand -base64 24

# Generate random password (macOS)
openssl rand -base64 24

# Or use password generator tools
pwgen -s 24 1
```

### Password Storage

- Never store passwords in:
  - Git repositories
  - Plain text files
  - Environment variables in code
  - Configuration files

- Always store passwords in:
  - HashiCorp Vault (recommended)
  - Secure password managers
  - Encrypted storage

### Access Control

- Limit Vault access to authorized personnel only
- Use AppRole for automation (not user tokens)
- Implement audit logging
- Regular access reviews
- Rotate passwords regularly

### Password Rotation

1. Update password in Vault
2. Update component configuration
3. Restart affected services
4. Test functionality
5. Document rotation date

---

## Troubleshooting

### Secret Not Found

**Error**: `Secret not found: secret/ping/platform8/ds/admin_password`

**Solution**:
1. Verify secret path is correct
2. Check Vault authentication
3. Verify KV secrets engine is enabled
4. Check Vault permissions

### Authentication Failed

**Error**: `Authentication failed`

**Solution**:
1. Verify Vault token is valid
2. Check AppRole credentials
3. Verify Vault address is correct
4. Check network connectivity

### Permission Denied

**Error**: `Permission denied`

**Solution**:
1. Verify user/AppRole has write permissions
2. Check Vault policies
3. Verify path permissions

---

## Next Steps

After creating all Vault secrets:

1. **Verify Secrets**: Run `./scripts/verify-vault-secrets.sh`
2. **Test Vault Connection**: Run `ansible-playbook playbooks/validate-vault-secrets.yml`
3. **Proceed with Deployment**: Refer to **05-execution-plan.md** for deployment execution
4. **Monitor Secrets**: Regularly audit and rotate passwords

---

## Related Documentation

- **09-service-accounts-and-vault-secrets.md**: Complete list of service accounts and secrets
- **04-prerequisites-setup.md**: Vault configuration and setup
- **05-execution-plan.md**: Deployment execution procedures

