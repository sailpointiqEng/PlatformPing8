# Ansible Project Structure

## Overview

This document defines the complete Ansible project structure for Ping Identity Platform 8 automation. The structure is organized to support multi-environment deployments, modular roles, and clear separation of concerns.

## Complete Directory Structure

```
ansible/
├── ansible.cfg                          # Ansible configuration
├── requirements.yml                     # Ansible collection requirements
├── inventory/
│   ├── dev/                             # Development environment
│   │   ├── hosts.yml                   # Dev inventory
│   │   └── group_vars/
│   │       ├── all.yml                  # Dev common variables
│   │       ├── infrastructure.yml       # Infrastructure-specific variables
│   │       ├── ds.yml                   # DS-specific variables
│   │       ├── am.yml                   # AM-specific variables
│   │       ├── idm.yml                  # IDM-specific variables
│   │       └── ig.yml                   # IG-specific variables
│   ├── test/                             # Test environment
│   │   ├── hosts.yml                   # Test inventory
│   │   └── group_vars/
│   │       ├── all.yml                  # Test common variables
│   │       ├── infrastructure.yml       # Infrastructure VMs
│   │       ├── ds.yml
│   │       ├── am.yml
│   │       ├── idm.yml
│   │       └── ig.yml
│   └── production/                      # Production environment
│       ├── hosts.yml                   # Production inventory
│       └── group_vars/
│           ├── all.yml                  # Production common variables
│           ├── infrastructure.yml       # Infrastructure VMs
│           ├── ds.yml
│           ├── am.yml
│           ├── idm.yml
│           └── ig.yml
├── roles/
│   ├── common/                          # Shared prerequisites for ALL components
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── java.yml                 # Install OpenJDK 21
│   │   │   ├── tomcat.yml               # Install Tomcat 10
│   │   │   └── prerequisites.yml        # OS pre-reqs, limits.conf, filesystems
│   │   ├── handlers/main.yml
│   │   └── vars/main.yml
│   ├── validation/                        # Pre-deployment checks
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── connectivity.yml
│   │   │   ├── ports.yml
│   │   │   ├── prerequisites.yml
│   │   │   └── software.yml
│   │   └── vars/main.yml
│   ├── vault/
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   └── retrieve_secrets.yml
│   │   ├── library/vault_lookup.py
│   │   └── vars/main.yml
│   ├── ds/
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── detect.yml
│   │   │   ├── install.yml
│   │   │   ├── update.yml
│   │   │   ├── config_store.yml
│   │   │   ├── cts.yml
│   │   │   ├── idrepo.yml
│   │   │   ├── replication.yml
│   │   │   └── verify.yml
│   │   ├── templates/ds-setup-command.j2
│   │   └── vars/main.yml
│   ├── am/
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── detect.yml
│   │   │   ├── install.yml
│   │   │   ├── update.yml
│   │   │   ├── deploy_war.yml
│   │   │   ├── amster.yml
│   │   │   ├── rest_config.yml
│   │   │   └── journeys.yml
│   │   ├── templates/
│   │   │   ├── setenv.j2
│   │   │   └── amster-config.j2
│   │   └── vars/main.yml
│   ├── idm/
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── detect.yml
│   │   │   ├── install.yml
│   │   │   ├── update.yml
│   │   │   ├── config_files.yml
│   │   │   ├── ad_connector.yml
│   │   │   └── verify.yml
│   │   ├── templates/
│   │   │   ├── repo.ds.json.j2
│   │   │   ├── boot.properties.j2
│   │   │   ├── ad-connector.json.j2
│   │   │   └── managed.json.j2
│   │   └── vars/main.yml
│   ├── ig/
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── install.yml
│   │   │   └── routes.yml
│   │   ├── templates/config.json.j2
│   │   └── vars/main.yml
│   └── ui/
│       ├── tasks/main.yml
│       └── templates/
├── playbooks/
│   ├── site.yml
│   ├── validate-prerequisites.yml
│   ├── deploy-infrastructure.yml
│   ├── deploy-ds.yml
│   ├── deploy-am.yml
│   ├── deploy-idm.yml
│   ├── deploy-ig.yml
│   ├── deploy-ui.yml
│   ├── post-deploy.yml
│   └── ad-integration.yml
├── library/vault_lookup.py
├── scripts/validate_deployment.sh
└── README.md
```

## Inventory Structure

### Development Environment (`inventory/dev/hosts.yml`)

```yaml
all:
  children:
    infrastructure:
      hosts:
        infra-tomcat-1:
          ansible_host: infra-tomcat-1.dev.example.com
          infrastructure_role: tomcat
          tomcat_version: "10.1.46"
        infra-tomcat-2:
          ansible_host: infra-tomcat-2.dev.example.com
          infrastructure_role: tomcat
          tomcat_version: "10.1.46"
        infra-jdk-1:
          ansible_host: infra-jdk-1.dev.example.com
          infrastructure_role: jdk
          jdk_version: "21"
    ds:
      hosts:
        ds-config-1:
          ansible_host: ds-config-1.dev.example.com
          ds_instance: config
          ds_replica_id: 1
          ds_replication_enabled: true
        ds-config-2:
          ansible_host: ds-config-2.dev.example.com
          ds_instance: config
          ds_replica_id: 2
          ds_replication_enabled: true
        ds-cts-1:
          ansible_host: ds-cts-1.dev.example.com
          ds_instance: cts
          ds_replica_id: 1
          ds_replication_enabled: true
        ds-cts-2:
          ansible_host: ds-cts-2.dev.example.com
          ds_instance: cts
          ds_replica_id: 2
          ds_replication_enabled: true
        ds-idrepo-1:
          ansible_host: ds-idrepo-1.dev.example.com
          ds_instance: idrepo
          ds_replica_id: 1
          ds_replication_enabled: true
        ds-idrepo-2:
          ansible_host: ds-idrepo-2.dev.example.com
          ds_instance: idrepo
          ds_replica_id: 2
          ds_replication_enabled: true
    am:
      hosts:
        am-1:
          ansible_host: am-1.dev.example.com
          am_replica_id: 1
          tomcat_host: infra-tomcat-1
        am-2:
          ansible_host: am-2.dev.example.com
          am_replica_id: 2
          tomcat_host: infra-tomcat-2
    idm:
      hosts:
        idm-1:
          ansible_host: idm-1.dev.example.com
          idm_replica_id: 1
        idm-2:
          ansible_host: idm-2.dev.example.com
          idm_replica_id: 2
```

### Test Environment (`inventory/test/hosts.yml`)

```yaml
all:
  children:
    infrastructure:
      hosts:
        infra-tomcat-1:
          ansible_host: infra-tomcat-1.test.example.com
          infrastructure_role: tomcat
          tomcat_version: "10.1.46"
        infra-tomcat-2:
          ansible_host: infra-tomcat-2.test.example.com
          infrastructure_role: tomcat
          tomcat_version: "10.1.46"
        infra-jdk-1:
          ansible_host: infra-jdk-1.test.example.com
          infrastructure_role: jdk
          jdk_version: "21"
    ds:
      hosts:
        ds-config-1:
          ansible_host: ds-config-1.test.example.com
          ds_instance: config
          ds_replica_id: 1
          ds_replication_enabled: true
        ds-config-2:
          ansible_host: ds-config-2.test.example.com
          ds_instance: config
          ds_replica_id: 2
          ds_replication_enabled: true
        ds-cts-1:
          ansible_host: ds-cts-1.test.example.com
          ds_instance: cts
          ds_replica_id: 1
          ds_replication_enabled: true
        ds-cts-2:
          ansible_host: ds-cts-2.test.example.com
          ds_instance: cts
          ds_replica_id: 2
          ds_replication_enabled: true
        ds-idrepo-1:
          ansible_host: ds-idrepo-1.test.example.com
          ds_instance: idrepo
          ds_replica_id: 1
          ds_replication_enabled: true
        ds-idrepo-2:
          ansible_host: ds-idrepo-2.test.example.com
          ds_instance: idrepo
          ds_replica_id: 2
          ds_replication_enabled: true
    am:
      hosts:
        am-1:
          ansible_host: am-1.test.example.com
          am_replica_id: 1
          tomcat_host: infra-tomcat-1
        am-2:
          ansible_host: am-2.test.example.com
          am_replica_id: 2
          tomcat_host: infra-tomcat-2
    idm:
      hosts:
        idm-1:
          ansible_host: idm-1.test.example.com
          idm_replica_id: 1
        idm-2:
          ansible_host: idm-2.test.example.com
          idm_replica_id: 2
```

### Production Environment (`inventory/production/hosts.yml`)

```yaml
all:
  children:
    infrastructure:
      hosts:
        infra-tomcat-1:
          ansible_host: infra-tomcat-1.prod.example.com
          infrastructure_role: tomcat
          tomcat_version: "10.1.46"
        infra-tomcat-2:
          ansible_host: infra-tomcat-2.prod.example.com
          infrastructure_role: tomcat
          tomcat_version: "10.1.46"
        infra-jdk-1:
          ansible_host: infra-jdk-1.prod.example.com
          infrastructure_role: jdk
          jdk_version: "21"
        infra-jdk-2:
          ansible_host: infra-jdk-2.prod.example.com
          infrastructure_role: jdk
          jdk_version: "21"
    ds:
      hosts:
        ds-config-1:
          ansible_host: ds-config-1.prod.example.com
          ds_instance: config
          ds_replica_id: 1
          ds_replication_enabled: true
        ds-config-2:
          ansible_host: ds-config-2.prod.example.com
          ds_instance: config
          ds_replica_id: 2
          ds_replication_enabled: true
        ds-cts-1:
          ansible_host: ds-cts-1.prod.example.com
          ds_instance: cts
          ds_replica_id: 1
          ds_replication_enabled: true
        ds-cts-2:
          ansible_host: ds-cts-2.prod.example.com
          ds_instance: cts
          ds_replica_id: 2
          ds_replication_enabled: true
        ds-idrepo-1:
          ansible_host: ds-idrepo-1.prod.example.com
          ds_instance: idrepo
          ds_replica_id: 1
          ds_replication_enabled: true
        ds-idrepo-2:
          ansible_host: ds-idrepo-2.prod.example.com
          ds_instance: idrepo
          ds_replica_id: 2
          ds_replication_enabled: true
    am:
      hosts:
        am-1:
          ansible_host: am-1.prod.example.com
          am_replica_id: 1
          tomcat_host: infra-tomcat-1
        am-2:
          ansible_host: am-2.prod.example.com
          am_replica_id: 2
          tomcat_host: infra-tomcat-2
    idm:
      hosts:
        idm-1:
          ansible_host: idm-1.prod.example.com
          idm_replica_id: 1
        idm-2:
          ansible_host: idm-2.prod.example.com
          idm_replica_id: 2
    ig:
      hosts:
        ig-1:
          ansible_host: ig-1.prod.example.com
        ig-2:
          ansible_host: ig-2.prod.example.com
```

## Role Organization

### Common Role

**Purpose**: Shared prerequisites for ALL components

**Tasks**:
- Install OpenJDK 21 (or 17)
- Install Tomcat 10
- OS prerequisites (packages, limits.conf, filesystems)
- Create service account (pingIdentity)
- Create installation directories

**Location**: `roles/common/`

**Pattern**: All components use this role for common prerequisites

### Validation Role

**Purpose**: Pre-deployment checks (100% clean and separate)

**Tasks**:
- Connectivity checks
- Port availability checks
- Prerequisites validation
- Software binary checks

**Location**: `roles/validation/`

**Usage**: Can be run standalone with `ansible-playbook validate-prerequisites.yml`

### Vault Role

**Purpose**: HashiCorp Vault integration

**Tasks**:
- Vault authentication (AppRole)
- Secret retrieval
- Custom lookup plugin

**Location**: `roles/vault/`

### DS Role

**Purpose**: Directory Services deployment

**Tasks** (follows standard pattern):
- `detect.yml` - Detect existing installation
- `install.yml` - Fresh installation
- `update.yml` - Incremental update
- `verify.yml` - Health checks
- Instance-specific: `config_store.yml`, `cts.yml`, `idrepo.yml`
- `replication.yml` - Replication setup (optional)

**Location**: `roles/ds/`

### AM Role

**Purpose**: Access Management deployment

**Tasks** (follows standard pattern):
- `detect.yml` - Detect existing installation
- `install.yml` - Fresh installation
- `update.yml` - Incremental update
- `verify.yml` - Health checks
- Component-specific: `deploy_war.yml`, `amster.yml`, `rest_config.yml`, `journeys.yml`

**Location**: `roles/am/`

### IDM Role

**Purpose**: Identity Management deployment

**Tasks** (follows standard pattern):
- `detect.yml` - Detect existing installation
- `install.yml` - Fresh installation
- `update.yml` - Incremental update
- `verify.yml` - Health checks
- Component-specific: `config_files.yml`, `ad_connector.yml`

**Location**: `roles/idm/`

### IG Role

**Purpose**: Identity Gateway deployment

**Tasks** (follows standard pattern):
- `detect.yml` - Detect existing installation
- `install.yml` - Fresh installation
- `update.yml` - Incremental update
- `verify.yml` - Health checks
- `routes.yml` - Route configuration

**Location**: `roles/ig/`

### UI Role

**Purpose**: Platform UI deployment

**Tasks**:
- Deploy UI components

**Location**: `roles/ui/`

## Playbook Organization

### Main Playbooks

1. **site.yml**: Main orchestration playbook
2. **validate-prerequisites.yml**: Pre-deployment validation
3. **deploy-infrastructure.yml**: Infrastructure deployment
4. **deploy-ds.yml**: DS deployment
5. **deploy-am.yml**: AM deployment
6. **deploy-idm.yml**: IDM deployment
7. **deploy-ig.yml**: IG deployment
8. **deploy-ui.yml**: UI deployment
9. **post-deploy.yml**: Post-deployment configuration
10. **ad-integration.yml**: AD connector setup

## Variable Organization

### Group Variables

**Location**: `inventory/{env}/group_vars/`

**Files**:
- `all.yml`: Common variables for all hosts
- `infrastructure.yml`: Infrastructure-specific variables (Tomcat/JDK configuration)
- `ds.yml`: DS-specific variables
- `am.yml`: AM-specific variables
- `idm.yml`: IDM-specific variables
- `ig.yml`: IG-specific variables

### Role Variables

**Location**: `roles/{role}/vars/main.yml`

**Purpose**: Default variables for each role

### Host Variables

**Location**: `inventory/{env}/host_vars/{hostname}.yml`

**Purpose**: Host-specific overrides

## Template Organization

**Location**: `roles/{role}/templates/`

**Purpose**: Jinja2 templates for configuration files

**Examples**:
- `roles/ds/templates/ds-setup-command.j2`
- `roles/am/templates/setenv.j2`
- `roles/idm/templates/repo.ds.json.j2`

## Handler Organization

**Location**: `roles/{role}/handlers/main.yml`

**Purpose**: Service restart handlers

**Examples**:
- Restart Tomcat
- Restart IDM
- Reload systemd

## File Naming Conventions

- **Playbooks**: `{action}-{component}.yml` (e.g., `deploy-ds.yml`)
- **Tasks**: `{action}.yml` (e.g., `install.yml`, `update.yml`)
- **Templates**: `{component}-{purpose}.j2` (e.g., `ds-setup-command.j2`)
- **Variables**: `{component}.yml` (e.g., `ds.yml`, `am.yml`)

## Component Pattern

All components follow the same pattern for consistency:

1. **detect.yml** - Detect existing installation
2. **install.yml** - Fresh installation
3. **update.yml** - Incremental update (preserves existing configs)
4. **verify.yml** - Health checks and verification

This ensures:
- Consistent behavior across all components
- Easy to understand and maintain
- Idempotent operations
- Safe re-deployments

## Best Practices

1. **Modularity**: Each component has its own role
2. **Reusability**: Common tasks in `roles/common/` (Tomcat + JDK are common prerequisites)
3. **Environment Separation**: Separate inventory per environment
4. **Variable Hierarchy**: Group vars → Role vars → Host vars
5. **Idempotency**: All tasks are idempotent
6. **Validation Separation**: Validation role is 100% clean and can be run standalone
7. **Consistent Pattern**: All components follow detect → install → update → verify pattern

## What Was Corrected / Improved

### 1. Removed Unnecessary `infrastructure/` Role

**Reason**:
- Tomcat + JDK are common prerequisites
- Should always live in `roles/common/`
- No need for separate infrastructure role

**Result**: Cleaner structure with Tomcat and JDK installation in `roles/common/tasks/tomcat.yml` and `roles/common/tasks/java.yml`

### 2. Ensured All Components Follow the Same Pattern

**Standard Pattern**:
- `detect.yml` → Detect existing installation
- `install.yml` → Fresh installation
- `update.yml` → Incremental update
- `verify.yml` → Health checks

**Benefits**:
- Consistent behavior across all components
- Easy to understand and maintain
- Predictable deployment flow
- Safe re-deployments

### 3. Validation Role Separated 100% Clean

**Purpose**: Pre-deployment validation can be run standalone

**Usage**:
```bash
ansible-playbook validate-prerequisites.yml
```

**Benefits**:
- Can validate environment before any deployment
- Independent of other playbooks
- Clear separation of concerns
- Easy to troubleshoot

## Next Steps

- Refer to **04-prerequisites-setup.md** for Vault and inventory configuration
- Refer to **05-execution-plan.md** for deployment execution
- Refer to **06-component-deployment.md** for component-specific details

