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
│   │       ├── infrastructure.yml       # Infrastructure VMs (Tomcat/JDK)
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
│   ├── common/                          # Common prerequisites
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── java.yml                 # Java 21 installation
│   │   │   ├── tomcat.yml               # Tomcat 10.x installation
│   │   │   └── prerequisites.yml        # System prerequisites
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   └── vars/main.yml
│   ├── infrastructure/                 # Infrastructure VMs (Tomcat/JDK)
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── install_tomcat.yml       # Install Tomcat on infrastructure VMs
│   │   │   ├── install_jdk.yml          # Install JDK on infrastructure VMs
│   │   │   └── verify.yml               # Verify infrastructure installation
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   │   └── tomcat.service.j2       # Tomcat systemd service template
│   │   └── vars/main.yml
│   ├── validation/                      # Pre-deployment validation
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── connectivity.yml         # Ping and connectivity checks
│   │   │   ├── ports.yml                 # Port availability checks
│   │   │   ├── prerequisites.yml        # Prerequisites validation
│   │   │   └── software.yml             # Software binary checks
│   │   └── vars/main.yml
│   ├── vault/                           # HashiCorp Vault integration
│   │   ├── tasks/
│   │   │   ├── main.yml                 # Vault authentication
│   │   │   └── retrieve_secrets.yml    # Secret retrieval
│   │   ├── library/
│   │   │   └── vault_lookup.py          # Custom Vault lookup plugin
│   │   └── vars/main.yml
│   ├── ds/                              # Directory Services role
│   │   ├── tasks/
│   │   │   ├── main.yml                 # Main orchestration
│   │   │   ├── detect.yml               # Detect existing installation
│   │   │   ├── install.yml              # Fresh installation
│   │   │   ├── update.yml                # Incremental update
│   │   │   ├── config_store.yml         # Config Store instance
│   │   │   ├── cts.yml                  # CTS instance
│   │   │   ├── idrepo.yml                # IDRepo instance
│   │   │   ├── replication.yml          # Replication setup (optional)
│   │   │   └── verify.yml                # Health checks
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   │   └── ds-setup-command.j2      # DS setup command template
│   │   └── vars/main.yml
│   ├── am/                              # Access Management role
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── detect.yml
│   │   │   ├── install.yml
│   │   │   ├── update.yml
│   │   │   ├── deploy_war.yml            # Deploy AM WAR to Tomcat
│   │   │   ├── amster.yml               # Amster configuration
│   │   │   ├── rest_config.yml          # REST API configuration
│   │   │   └── journeys.yml             # Authentication tree import
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   │   ├── setenv.j2                # Tomcat setenv.sh
│   │   │   └── amster-config.j2
│   │   └── vars/main.yml
│   ├── idm/                             # Identity Management role
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── detect.yml
│   │   │   ├── install.yml
│   │   │   ├── update.yml
│   │   │   ├── config_files.yml          # IDM config file deployment
│   │   │   ├── ad_connector.yml          # AD connector setup
│   │   │   └── verify.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   │   ├── repo.ds.json.j2          # DS repository config
│   │   │   ├── boot.properties.j2
│   │   │   ├── ad-connector.json.j2     # AD connector config
│   │   │   └── managed.json.j2
│   │   └── vars/main.yml
│   ├── ig/                              # Identity Gateway role
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── install.yml
│   │   │   └── routes.yml               # Route configuration
│   │   ├── templates/
│   │   │   └── config.json.j2
│   │   └── vars/main.yml
│   └── ui/                              # Platform UI role
│       ├── tasks/main.yml
│       └── templates/
├── playbooks/
│   ├── site.yml                         # Main orchestration playbook
│   ├── validate-prerequisites.yml      # Pre-deployment validation playbook
│   ├── deploy-infrastructure.yml        # Infrastructure VMs (Tomcat/JDK)
│   ├── deploy-ds.yml                     # DS deployment playbook
│   ├── deploy-am.yml                    # AM deployment playbook
│   ├── deploy-idm.yml                   # IDM deployment playbook
│   ├── deploy-ig.yml                    # IG deployment playbook
│   ├── deploy-ui.yml                    # UI deployment playbook
│   ├── post-deploy.yml                  # Post-deployment configuration
│   └── ad-integration.yml               # AD connector setup
├── library/
│   └── vault_lookup.py                  # Custom Vault lookup plugin
├── scripts/
│   └── validate_deployment.sh          # Deployment validation script
└── README.md                            # Documentation
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
          ds_replication_enabled: false
        ds-cts-1:
          ansible_host: ds-cts-1.dev.example.com
          ds_instance: cts
          ds_replica_id: 1
        ds-idrepo-1:
          ansible_host: ds-idrepo-1.dev.example.com
          ds_instance: idrepo
          ds_replica_id: 1
    am:
      hosts:
        am-1:
          ansible_host: am-1.dev.example.com
          am_replica_id: 1
          tomcat_host: infra-tomcat-1
    idm:
      hosts:
        idm-1:
          ansible_host: idm-1.dev.example.com
          idm_replica_id: 1
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

**Purpose**: Handles common prerequisites for all components

**Tasks**:
- Java installation
- Tomcat installation
- System prerequisites

**Location**: `roles/common/`

### Infrastructure Role

**Purpose**: Manages infrastructure VMs (Tomcat and JDK)

**Tasks**:
- Install Tomcat on infrastructure VMs
- Install JDK on infrastructure VMs
- Verify infrastructure installation

**Location**: `roles/infrastructure/`

### Validation Role

**Purpose**: Pre-deployment validation (MUST PASS before deployment)

**Tasks**:
- Connectivity checks
- Port availability checks
- Prerequisites validation
- Software binary checks

**Location**: `roles/validation/`

### Vault Role

**Purpose**: HashiCorp Vault integration

**Tasks**:
- Vault authentication (AppRole)
- Secret retrieval
- Custom lookup plugin

**Location**: `roles/vault/`

### DS Role

**Purpose**: Directory Services deployment

**Tasks**:
- Detect existing installation
- Fresh installation
- Incremental updates
- Replication setup
- Health checks

**Location**: `roles/ds/`

### AM Role

**Purpose**: Access Management deployment

**Tasks**:
- Detect existing installation
- Deploy AM WAR
- Amster configuration
- REST API configuration
- Authentication tree import

**Location**: `roles/am/`

### IDM Role

**Purpose**: Identity Management deployment

**Tasks**:
- Detect existing installation
- Deploy IDM
- Configure IDM files
- AD connector setup

**Location**: `roles/idm/`

### IG Role

**Purpose**: Identity Gateway deployment

**Tasks**:
- Install IG
- Configure routes

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
- `infrastructure.yml`: Infrastructure-specific variables
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

## Best Practices

1. **Modularity**: Each component has its own role
2. **Reusability**: Common tasks in common role
3. **Environment Separation**: Separate inventory per environment
4. **Variable Hierarchy**: Group vars → Role vars → Host vars
5. **Idempotency**: All tasks are idempotent
6. **Documentation**: Each role has README.md

## Next Steps

- Refer to **04-prerequisites-setup.md** for Vault and inventory configuration
- Refer to **05-execution-plan.md** for deployment execution
- Refer to **06-component-deployment.md** for component-specific details

