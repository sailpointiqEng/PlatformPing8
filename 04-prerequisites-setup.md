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

**Script**: `scripts/vault-setup.sh`

**Step 2: Configure Ansible Variables**

**File**: `inventory/{env}/group_vars/all.yml`

### Custom Vault Lookup Plugin

**File**: `library/vault_lookup.py`

### Vault Role Implementation

**File**: `roles/vault/tasks/main.yml`

## Inventory Setup

### Environment-Specific Variables

**File**: `inventory/dev/group_vars/all.yml`

**File**: `inventory/test/group_vars/all.yml`

**File**: `inventory/production/group_vars/all.yml`

### Component-Specific Variables

**File**: `inventory/{env}/group_vars/ds.yml`

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

### requirements.yml

**File**: `requirements.yml`

## System Prerequisites

### Operating System Requirements

- **Supported OS**: RHEL 7.9+, CentOS 7.9+, Ubuntu 18.04+, Debian 10+
- **Architecture**: x86_64
- **Memory**: Minimum 16GB RAM
- **Disk Space**: Minimum 20GB free space
- **Network**: All VMs must be reachable via SSH

### Python Requirements

**File**: `requirements-python.txt`

### Installation

**Script**: `scripts/install-prerequisites.sh`

## SSH Configuration

### Passwordless SSH Setup

**Requirement**: Ansible control node must have passwordless SSH access to all target VMs

**Setup Steps**:

1. Generate SSH key on Ansible control node
2. Copy public key to all target VMs
3. Test connectivity

**Script**: `scripts/setup-ssh.sh`

## Network Prerequisites

### DNS Configuration

**Requirement**: All hostnames must resolve correctly

**Validation Commands**: `hostname -f` and `getent hosts <IP_ADDRESS>`

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

### Vault Setup (MUST BE DONE FIRST)

- [ ] HashiCorp Vault is accessible and configured
- [ ] All secrets are stored in Vault (see **10-vault-secrets-setup.md**)
- [ ] AppRole is created and credentials obtained
- [ ] Vault secrets verification passed

**Reference**: See **10-vault-secrets-setup.md** for complete Vault secrets setup instructions

### General Prerequisites

- [ ] Inventory files are configured for target environment
- [ ] All software binaries are in place
- [ ] SSH access is configured (passwordless)
- [ ] DNS resolution works for all hostnames
- [ ] Firewall rules are configured (by network team)
- [ ] Service account (pingIdentity) exists on all VMs (will be created during deployment)
- [ ] Installation directories are created (will be created during deployment)
- [ ] Java is installed (or will be installed on infrastructure VMs)
- [ ] JAVA_HOME is set correctly (will be configured during deployment)

## Validation Commands

### Test Vault Connection

**Playbook**: `playbooks/test-vault-connection.yml`

### Test Inventory

**Command**: `ansible-inventory -i inventory/dev/hosts.yml --list`

### Test Connectivity

**Command**: `ansible all -i inventory/dev/hosts.yml -m ping`

### Validate Prerequisites

**Playbook**: `playbooks/validate-prerequisites.yml`

## Next Steps

After prerequisites are set up:
- Refer to **05-execution-plan.md** for deployment execution
- Refer to **06-component-deployment.md** for component-specific deployment
- Refer to **07-post-deployment.md** for post-deployment configuration

