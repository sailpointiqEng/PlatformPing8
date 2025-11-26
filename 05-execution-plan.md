# Execution Plan

## Overview

This document provides step-by-step execution procedures for deploying Ping Identity Platform 8 using Ansible. It covers playbook execution order, validation workflows, and deployment procedures for all environments.

## Execution Workflow

### High-Level Flow

```
0. Vault Secrets Setup (MUST BE DONE FIRST)
   ↓
1. Environment Preparation (run first)
   ↓
2. Infrastructure Deployment (run second)
   ↓
3. Pre-Deployment Validation (must pass)
   ↓
4. Component Deployment (DS → AM → IDM → IG → UI)
   ↓
5. Post-Deployment Configuration
   ↓
6. AD Integration (if enabled)
   ↓
7. Verification
```

### Deployment Order Summary

**Vault Secrets Setup → Environment Prep → Infrastructure → Validation → DS → AM → IDM → IG/UI**

## Step-by-Step Execution

### Step 0: Vault Secrets Setup (MUST BE DONE FIRST)

**Purpose**: Create all required secrets in HashiCorp Vault before deployment

**Documentation**: Refer to **10-vault-secrets-setup.md** for complete instructions

**Prerequisites**:
- HashiCorp Vault is installed and running
- Vault CLI is installed and configured
- Vault authentication is configured
- KV secrets engine is enabled

**Required Actions**:
1. Create all DS secrets in Vault
2. Create all AM secrets in Vault
3. Create all IDM secrets in Vault
4. Create all AD secrets in Vault (if AD integration enabled)
5. Create all common secrets in Vault

**Verification**:
```bash
# Run verification script
./scripts/verify-vault-secrets.sh

# Or run Ansible validation
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/validate-vault-secrets.yml
```

**Expected Result**: All secrets exist in Vault and are accessible

**If secrets are missing**: Deployment will fail. Create all required secrets before proceeding.

**Reference**: See **10-vault-secrets-setup.md** for detailed instructions and scripts

---

### Step 1: Environment Preparation

**Purpose**: Prepare the environment with all prerequisites

**Playbook**: `playbooks/prepare-environment.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/prepare-environment.yml
```

**What it does**:
- Create service account (pingIdentity)
- Create installation directories (`/opt/ping`, `/opt/ping/am`, `/opt/ping/idm`, `/opt/ping/ds`, `/opt/ping/ig`)
- Install required OS packages (curl, unzip)
- Configure hostname + DNS (network team will take care)
- Configure NTP/time synchronization (network team will take care)
- Create truststore directory (certs added later)
- Validate OS version, CPU, RAM, disk space

**Expected Result**: Environment is prepared

### Step 2: Infrastructure Deployment

**Purpose**: Deploy Java JDK and Tomcat on infrastructure VMs

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
- Install Java JDK (17 or 21 based on Ping docs)
- Set JAVA_HOME and PATH
- Install Tomcat (10.x)
- Deploy baseline Tomcat configs (server.xml, setenv.sh)
- Open system firewall ports (8080/8443 etc.) - Network team will take care
- Verify Tomcat starts successfully

**Expected Result**: Infrastructure VMs are ready

### Step 3: Pre-Deployment Validation

**Purpose**: Validate all prerequisites before deployment (MUST PASS)

**Playbook**: `playbooks/validate-prerequisites.yml`

**Command**:
```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/validate-prerequisites.yml
```

**What it checks**:
- Validate Vault secrets exist (all required secrets are in Vault)
- Validate connectivity to all nodes (ping/SSH)
- Validate required ports are free and not in use
- Validate Java version is correct
- Validate Tomcat installation health
- Validate truststore directory exists
- Validate permissions/ownership for installation directories
- Validate enough disk size
- Validate Vault connectivity (for pulling secrets)

**Expected Result**: All checks pass

**If validation fails**: Fix issues and re-run validation. Deployment will not proceed until validation passes.

### Step 4: Component Deployment

#### 4.1 Deploy Directory Services (DS)

**Purpose**: Deploy DS instances (Config Store → CTS → IDRepo → replication)

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

#### 4.2 Deploy Access Management (AM)

**Purpose**: Deploy AM (WAR + Amster configuration) → two servers

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

**Expected Result**: AM is deployed and accessible on two servers

#### 4.3 Deploy Identity Management (IDM)

**Purpose**: Deploy IDM (application + config + AD connector) → two servers

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
- Configures AD connector (if enabled)
- Updates existing installation (if already installed)

**Expected Result**: IDM is running and connected to DS on two servers

#### 4.4 Deploy Identity Gateway (IG) - Optional

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

#### 4.5 Deploy Platform UI - Optional

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

### Step 5: Post-Deployment Configuration

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

### Step 6: AD Integration

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

### Step 7: Verification

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
0. Validates Vault secrets exist (MUST BE DONE FIRST)
1. Environment preparation
2. Infrastructure deployment
3. Pre-deployment validation (MUST PASS)
4. Deploys all components (DS → AM → IDM → IG → UI)
5. Runs post-deployment configuration
6. Configures AD integration (if enabled)
7. Verification

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
- **Validation**: 10 minutes
- **Infrastructure**: 15 minutes
- **DS Deployment**: 30 minutes (with replication)
- **AM Deployment**: 30 minutes
- **IDM Deployment**: 20 minutes
- **Post-Deployment**: 15 minutes
- **Total**: ~120 minutes

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

