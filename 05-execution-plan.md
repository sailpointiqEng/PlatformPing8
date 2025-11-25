# Execution Plan

## Overview

This document provides step-by-step execution procedures for deploying Ping Identity Platform 8 using Ansible. It covers playbook execution order, validation workflows, and deployment procedures for all environments.

## Execution Workflow

### High-Level Flow

```
1. Pre-Deployment Validation (MANDATORY)
   ↓
2. Infrastructure Deployment
   ↓
3. Component Deployment (DS → AM → IDM → IG → UI)
   ↓
4. Post-Deployment Configuration
   ↓
5. AD Integration (if enabled)
   ↓
6. Verification
```

## Step-by-Step Execution

### Step 1: Pre-Deployment Validation

**Purpose**: Validate all prerequisites before deployment

**Playbook**: `playbooks/validate-prerequisites.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/validate-prerequisites.yml
```

**What it checks**:
- Connectivity to all VMs
- Port availability
- Java version and JAVA_HOME
- Service account (pingIdentity)
- Installation directories
- NTP synchronization
- DNS resolution
- Software binaries

**Expected Result**: All checks pass

**If validation fails**: Fix issues and re-run validation. Deployment will not proceed until validation passes.

### Step 2: Infrastructure Deployment

**Purpose**: Deploy Tomcat and JDK on infrastructure VMs

**Playbook**: `playbooks/deploy-infrastructure.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/deploy-infrastructure.yml \
  --limit infrastructure
```

**What it does**:
- Installs Tomcat on infrastructure VMs
- Installs JDK on infrastructure VMs
- Configures systemd services
- Verifies installation

**Expected Result**: Infrastructure VMs are ready

### Step 3: Component Deployment

#### 3.1 Deploy Directory Services (DS)

**Purpose**: Deploy DS instances (Config Store, CTS, IDRepo)

**Playbook**: `playbooks/deploy-ds.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/deploy-ds.yml \
  --limit ds
```

**What it does**:
- Detects existing DS installations
- Installs DS Config Store (if not installed)
- Installs DS CTS (if not installed)
- Installs DS IDRepo (if not installed)
- Configures replication (if enabled and auto_replication=true)
- Updates existing installations (if already installed)

**Expected Result**: All DS instances are running

#### 3.2 Deploy Access Management (AM)

**Purpose**: Deploy AM on Tomcat infrastructure VMs

**Playbook**: `playbooks/deploy-am.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/deploy-am.yml \
  --limit am
```

**What it does**:
- Detects existing AM installation
- Deploys AM WAR to Tomcat
- Configures AM via Amster
- Imports authentication trees
- Updates existing installation (if already installed)

**Expected Result**: AM is deployed and accessible

#### 3.3 Deploy Identity Management (IDM)

**Purpose**: Deploy IDM

**Playbook**: `playbooks/deploy-idm.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/deploy-idm.yml \
  --limit idm
```

**What it does**:
- Detects existing IDM installation
- Extracts and deploys IDM
- Configures IDM files
- Connects to DS IDRepo
- Updates existing installation (if already installed)

**Expected Result**: IDM is running and connected to DS

#### 3.4 Deploy Identity Gateway (IG) - Optional

**Purpose**: Deploy IG if needed

**Playbook**: `playbooks/deploy-ig.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/deploy-ig.yml \
  --limit ig
```

#### 3.5 Deploy Platform UI - Optional

**Purpose**: Deploy UI if needed

**Playbook**: `playbooks/deploy-ui.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/deploy-ui.yml \
  --limit am
```

### Step 4: Post-Deployment Configuration

**Purpose**: Configure AM via REST API and set up integrations

**Playbook**: `playbooks/post-deploy.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/post-deploy.yml
```

**What it does**:
- Configures AM realms
- Configures OAuth2 clients
- Configures IDM integration service
- Maps self-service trees
- Configures validation services
- Sets up CORS
- Configures base URLs

**Expected Result**: AM is fully configured

### Step 5: AD Integration

**Purpose**: Configure AD connector in IDM

**Playbook**: `playbooks/ad-integration.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  -e "ad_integration_enabled=true" \
  playbooks/ad-integration.yml
```

**What it does**:
- Configures AD connector in IDM
- Sets up synchronization mappings
- Configures reconciliation jobs
- Sets up data flow: AD → IDM → DS

**Expected Result**: AD integration is functional

### Step 6: Verification

**Purpose**: Verify all components and integrations

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/verify-deployment.yml
```

**What it checks**:
- All services are running
- AM → DS connectivity
- IDM → DS connectivity
- AM → IDM OAuth2 flow
- AD → IDM → DS data flow

## Complete Deployment (All-in-One)

### Main Orchestration Playbook

**Playbook**: `playbooks/site.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml
```

**What it does**:
1. Retrieves credentials from Vault
2. Runs pre-deployment validation (MUST PASS)
3. Deploys infrastructure
4. Deploys all components
5. Runs post-deployment configuration
6. Configures AD integration (if enabled)

## Environment-Specific Execution

### Development Environment

```bash
# Set environment
ENV=dev

# Full deployment
ansible-playbook -i inventory/${ENV}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml
```

### Test Environment

```bash
# Set environment
ENV=test

# Full deployment
ansible-playbook -i inventory/${ENV}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml
```

### Production Environment

```bash
# Set environment
ENV=production

# Full deployment (with extra caution)
ansible-playbook -i inventory/${ENV}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --check  # Dry run first
```

## Tagged Execution

### Execute Only Validation

```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --tags validation
```

### Execute Only Infrastructure

```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --tags infrastructure
```

### Execute Only DS Deployment

```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --tags ds
```

### Execute Only AM Deployment

```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --tags am
```

### Execute Only Post-Deployment

```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --tags post-deploy
```

## Dry Run (Check Mode)

**Purpose**: Preview changes without applying them

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --check \
  --diff
```

## Update/Re-deployment

### Update Existing Installation

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  -e "deployment_mode=update" \
  playbooks/site.yml
```

**What it does**:
- Detects existing installations
- Backs up existing configurations
- Updates only what's necessary
- Preserves existing configs, CTS, identity store, extensions

## Rolling Updates

### Update One Replica at a Time

**Command**:
```bash
# Update first replica
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --limit ds-config-1

# Update second replica
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --limit ds-config-2
```

## Troubleshooting Execution

### Verbose Output

```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  -vvv  # Very verbose
```

### Start from Specific Task

```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --start-at-task "Deploy DS Config Store"
```

### Limit to Specific Hosts

```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/site.yml \
  --limit ds-config-1,ds-cts-1
```

## Execution Best Practices

1. **Always validate first**: Run validation before deployment
2. **Use check mode**: Test with `--check` before production
3. **Start with dev**: Test in dev environment first
4. **Rolling updates**: Update one replica at a time in production
5. **Backup before update**: Ensure backups are in place
6. **Monitor logs**: Watch logs during execution
7. **Verify after deployment**: Always run verification

## Execution Timeline

### Development Environment
- **Validation**: 5 minutes
- **Infrastructure**: 10 minutes
- **DS Deployment**: 15 minutes
- **AM Deployment**: 20 minutes
- **IDM Deployment**: 15 minutes
- **Post-Deployment**: 10 minutes
- **Total**: ~75 minutes

### Test Environment
- **Validation**: 10 minutes
- **Infrastructure**: 15 minutes
- **DS Deployment**: 30 minutes (with replication)
- **AM Deployment**: 30 minutes
- **IDM Deployment**: 20 minutes
- **Post-Deployment**: 15 minutes
- **Total**: ~120 minutes

### Production Environment
- **Validation**: 15 minutes
- **Infrastructure**: 20 minutes
- **DS Deployment**: 60 minutes (rolling updates)
- **AM Deployment**: 45 minutes (rolling updates)
- **IDM Deployment**: 30 minutes (rolling updates)
- **Post-Deployment**: 20 minutes
- **Total**: ~190 minutes

## Next Steps

After execution:
- Refer to **06-component-deployment.md** for component-specific details
- Refer to **07-post-deployment.md** for post-deployment procedures
- Refer to **08-reference-guide.md** for troubleshooting

