# Prerequisites Setup

## Overview

This document covers all prerequisites, dependencies, and preparation tasks required before executing the Ping Identity Platform 8 deployment. This includes HashiCorp Vault configuration, inventory setup, software requirements, and all preparation tasks.

## HashiCorp Vault Configuration

### Vault Secret Structure

Organize secrets in Vault with the following structure:

```
secret/ping/platform8/
├── ds/
│   ├── admin_password                  # DS Directory Manager password
│   ├── deployment_id                   # DS deployment ID
│   ├── deployment_id_password          # DS deployment ID password
│   └── monitor_password                # DS monitor user password
├── am/
│   ├── admin_password                  # AM amadmin password
│   └── amster_key_passphrase           # Amster key passphrase
├── idm/
│   ├── admin_password                  # IDM admin password
│   └── ad_connector_password           # AD connector bind password
├── ad/
│   ├── bind_dn                         # AD bind DN
│   ├── bind_password                   # AD bind password
│   └── base_dn                         # AD base DN
└── common/
    ├── truststore_password             # Java truststore password
    └── keystore_password               # Keystore password
```

### Vault AppRole Setup

**Step 1: Create AppRole in Vault**

```bash
# Enable AppRole auth method
vault auth enable approle

# Create policy for Ansible
vault policy write ansible-ping-platform8 - <<EOF
path "secret/ping/platform8/*" {
  capabilities = ["read"]
}
EOF

# Create AppRole
vault write auth/approle/role/ansible-ping-platform8 \
    token_policies="ansible-ping-platform8" \
    token_ttl=1h \
    token_max_ttl=4h \
    bind_secret_id=true

# Get Role ID
vault read auth/approle/role/ansible-ping-platform8/role-id

# Generate Secret ID
vault write -f auth/approle/role/ansible-ping-platform8/secret-id
```

**Step 2: Configure Ansible Variables**

**File**: `inventory/{env}/group_vars/all.yml`

```yaml
vault_enabled: true
vault_addr: "https://vault.example.com:8200"
vault_approle_role_id: "{{ vault_role_id | default(omit) }}"
vault_approle_secret_id: "{{ vault_secret_id | default(omit) }}"
vault_secret_path: "secret/ping/platform8"
```

### Custom Vault Lookup Plugin

**File**: `library/vault_lookup.py`

```python
from ansible.plugins.lookup import LookupBase
from ansible.errors import AnsibleError
import hvac
import os

class LookupModule(LookupBase):
    def run(self, terms, variables=None, **kwargs):
        vault_addr = variables.get('vault_addr', os.environ.get('VAULT_ADDR'))
        vault_role_id = variables.get('vault_approle_role_id')
        vault_secret_id = variables.get('vault_approle_secret_id')
        vault_secret_path = variables.get('vault_secret_path', 'secret/ping/platform8')
        
        if not vault_addr:
            raise AnsibleError("vault_addr must be set")
        
        # Authenticate with Vault using AppRole
        client = hvac.Client(url=vault_addr)
        
        if vault_role_id and vault_secret_id:
            response = client.auth.approle.login(
                role_id=vault_role_id,
                secret_id=vault_secret_id
            )
            client.token = response['auth']['client_token']
        
        # Retrieve secrets
        results = []
        for term in terms:
            secret_path = f"{vault_secret_path}/{term}"
            try:
                secret = client.secrets.kv.v2.read_secret_version(path=secret_path)
                if 'data' in secret['data']:
                    # KV v2 format
                    value = secret['data']['data'].get(term.split('/')[-1])
                else:
                    # KV v1 format
                    value = secret['data'].get(term.split('/')[-1])
                
                if value:
                    results.append(value)
                else:
                    raise AnsibleError(f"Secret {term} not found in {secret_path}")
            except Exception as e:
                raise AnsibleError(f"Failed to retrieve secret {term}: {str(e)}")
        
        return results
```

### Vault Role Implementation

**File**: `roles/vault/tasks/main.yml`

```yaml
---
- name: Authenticate with Vault using AppRole
  set_fact:
    vault_token: "{{ lookup('vault_lookup', 'vault/token') }}"
  when: vault_enabled | default(true)
  no_log: true

- name: Set Vault token for session
  set_fact:
    ansible_vault_token: "{{ vault_token }}"
  when: vault_enabled | default(true)
  no_log: true
```

## Inventory Setup

### Environment-Specific Variables

**File**: `inventory/dev/group_vars/all.yml`

```yaml
# Environment
environment: dev

# Installation user
install_user: pingIdentity

# Base installation directory
base_install_dir: /opt/ping

# Software versions
ds_version: "8.0.0"
am_version: "8.0.1"
amster_version: "8.0.1"
idm_version: "8.0.0"
ig_version: "2025.3.0"
ui_version: "8.0.1.0523"

# Tomcat configuration
tomcat_dir: /opt/tomcat
tomcat_version: "10.1.46"
tomcat_http_port: 8081

# Java configuration
jdk_version: "21"
java_home: /usr/lib/jvm/java-21-openjdk

# Vault configuration
vault_enabled: true
vault_addr: "https://vault-dev.example.com:8200"
```

**File**: `inventory/test/group_vars/all.yml`

```yaml
# Same as dev but with test-specific values
environment: test
vault_addr: "https://vault-test.example.com:8200"
```

**File**: `inventory/production/group_vars/all.yml`

```yaml
# Same as dev but with production-specific values
environment: production
vault_addr: "https://vault-prod.example.com:8200"
```

### Component-Specific Variables

**File**: `inventory/{env}/group_vars/ds.yml`

```yaml
# DS Configuration
ds_dir: "{{ base_install_dir }}/ds"
ds_zip_file: "{{ software_dir }}/ds/DS-{{ ds_version }}.zip"

# DS Config Store
ds_config: "{{ ds_dir }}/config"
ds_amconfig_server: amconfig1.{{ environment }}.example.com
ds_amconfig_server_ldap_port: 3389
ds_amconfig_server_ldaps_port: 3636
ds_amconfig_server_http_port: 38081
ds_amconfig_server_https_port: 38443
ds_amconfig_server_admin_connector_port: 34444

# DS CTS
ds_cts: "{{ ds_dir }}/cts"
ds_cts_server: cts1.{{ environment }}.example.com
ds_cts_server_ldap_port: 1389
ds_cts_server_ldaps_port: 1636
ds_cts_server_http_port: 18081
ds_cts_server_https_port: 18443
ds_cts_server_admin_connector_port: 14444

# DS IDRepo
ds_idrepo: "{{ ds_dir }}/idrepo"
ds_idrepo_server: idrepo1.{{ environment }}.example.com
ds_idrepo_server_ldap_port: 2389
ds_idrepo_server_ldaps_port: 2636
ds_idrepo_server_http_port: 28081
ds_idrepo_server_https_port: 28443
ds_idrepo_server_admin_connector_port: 24444
ds_idrepo_dn: "ou=identities"

# DS Admin
ds_admin_dn: "cn=Directory Manager"
ds_deployment_id: "AX8fkJybs4nP3qfAXN3CMw4BCYspGQ5CBVN1bkVDAOgwVKjG2Wo2ZTs"
```

**File**: `inventory/{env}/group_vars/am.yml`

```yaml
# AM Configuration
am_context: am
am_hostname: am.{{ environment }}.example.com
am_url: "http://{{ am_hostname }}:{{ tomcat_http_port }}/{{ am_context }}"
am_dir: "{{ base_install_dir }}/{{ am_context }}"
am_cfg_dir: "/home/{{ install_user }}/.openamcfg"
am_war: "{{ software_dir }}/am/AM-{{ am_version }}.war"
amster_zip: "{{ software_dir }}/am/Amster-{{ amster_version }}.zip"
amster_dir: "{{ software_dir }}/am/amster"

# AM Truststore
am_truststore_folder: "{{ base_install_dir }}/{{ am_context }}/security/keystores"
am_truststore: "{{ am_truststore_folder }}/truststore"
truststore_password: "changeit"

# Cookie domain
cookie_domain: "{{ environment }}.example.com"
```

**File**: `inventory/{env}/group_vars/idm.yml`

```yaml
# IDM Configuration
idm_dir: "{{ base_install_dir }}"
idm_extract_dir: "{{ idm_dir }}/openidm"
idm_config_dir: "{{ idm_extract_dir }}/conf"
idm_hostname: openidm.{{ environment }}.example.com
idm_zip: "{{ software_dir }}/idm/IDM-{{ idm_version }}.zip"

# IDM Ports
boot_port_http: 8080
boot_port_https: 8553
boot_port_mutualauth: 9444

# IDM Truststore
idm_truststore: "{{ idm_extract_dir }}/security/truststore"
idm_storepass_file: "{{ idm_extract_dir }}/security/storepass"
idm_cert_file: "{{ am_truststore_folder }}/ds-repo-ca-cert.pem"
```

## Software Requirements

### Required Software Binaries

Place the following software files in the `software/` directory structure:

```
software/
├── am/
│   ├── AM-8.0.1.war
│   └── Amster-8.0.1.zip
├── ds/
│   └── DS-8.0.0.zip
├── idm/
│   └── IDM-8.0.0.zip
├── ig/
│   └── PingGateway-2025.3.0.zip
└── ui/
    └── PlatformUI-8.0.1.0523.zip
```

### Software Directory Variable

**File**: `inventory/{env}/group_vars/all.yml`

```yaml
software_dir: "/path/to/software"
```

## Ansible Configuration

### ansible.cfg

**File**: `ansible.cfg`

```ini
[defaults]
inventory = inventory/production/hosts.yml
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 86400

[inventory]
enable_plugins = host_list, script, auto, yaml, ini, toml

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
```

### requirements.yml

**File**: `requirements.yml`

```yaml
---
collections:
  - name: community.general
    version: ">=5.0.0"
  - name: ansible.posix
    version: ">=1.5.0"
  - name: community.crypto
    version: ">=2.0.0"

roles:
  # No external roles required - all roles are custom
```

## System Prerequisites

### Operating System Requirements

- **Supported OS**: RHEL 7.9+, CentOS 7.9+, Ubuntu 18.04+, Debian 10+
- **Architecture**: x86_64
- **Memory**: Minimum 16GB RAM
- **Disk Space**: Minimum 20GB free space
- **Network**: All VMs must be reachable via SSH

### Python Requirements

**File**: `requirements-python.txt`

```
ansible>=2.10.0
hvac>=1.0.0
```

### Installation

```bash
pip install -r requirements-python.txt
ansible-galaxy collection install -r requirements.yml
```

## SSH Configuration

### Passwordless SSH Setup

**Requirement**: Ansible control node must have passwordless SSH access to all target VMs

**Setup Steps**:

1. Generate SSH key on Ansible control node:
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible_key
```

2. Copy public key to all target VMs:
```bash
ssh-copy-id -i ~/.ssh/ansible_key.pub pingIdentity@target-vm
```

3. Test connectivity:
```bash
ansible all -i inventory/dev/hosts.yml -m ping
```

## Network Prerequisites

### DNS Configuration

**Requirement**: All hostnames must resolve correctly

**Validation**:
```bash
# Forward DNS
hostname -f

# Reverse DNS
getent hosts <IP_ADDRESS>
```

### Firewall Rules

**Note**: Network team will configure firewall, but ports must be documented:

**Required Ports**:
- SSH: 22
- DS LDAP: 1389, 2389, 3389
- DS LDAPS: 1636, 2636, 3636
- DS Admin: 14444, 24444, 34444
- AM HTTP: 8081
- AM HTTPS: 8443
- IDM HTTP: 8080
- IDM HTTPS: 8553

## Pre-Deployment Checklist

Before executing deployment, verify:

- [ ] HashiCorp Vault is accessible and configured
- [ ] All secrets are stored in Vault
- [ ] AppRole is created and credentials obtained
- [ ] Inventory files are configured for target environment
- [ ] All software binaries are in place
- [ ] SSH access is configured (passwordless)
- [ ] DNS resolution works for all hostnames
- [ ] Firewall rules are configured (by network team)
- [ ] Service account (pingIdentity) exists on all VMs
- [ ] Installation directories are created
- [ ] Java is installed (or will be installed on infrastructure VMs)
- [ ] JAVA_HOME is set correctly

## Validation Commands

### Test Vault Connection

```bash
ansible-playbook -i inventory/dev/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/test-vault-connection.yml
```

### Test Inventory

```bash
ansible-inventory -i inventory/dev/hosts.yml --list
```

### Test Connectivity

```bash
ansible all -i inventory/dev/hosts.yml -m ping
```

### Validate Prerequisites

```bash
ansible-playbook -i inventory/dev/hosts.yml \
  playbooks/validate-prerequisites.yml
```

## Next Steps

After prerequisites are set up:
- Refer to **05-execution-plan.md** for deployment execution
- Refer to **06-component-deployment.md** for component-specific deployment
- Refer to **07-post-deployment.md** for post-deployment configuration

