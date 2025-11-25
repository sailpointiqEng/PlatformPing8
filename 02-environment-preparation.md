# Environment Preparation and Validation

## Overview

This document outlines all environment preparation requirements and validation procedures that MUST be completed before deploying Ping Identity Platform 8 components. All validation checks must pass before any deployment can proceed.

## Environment Preparation Requirements

### 1. Java Version

**Requirement**: PingAM/PingIDM/PingDS 8 supports OpenJDK 17 and 21

**Validation Command**: `java -version`

**Expected Output**: OpenJDK version "21.0.x" or "17.0.x"

**Ansible Task**: `roles/validation/tasks/prerequisites.yml`

### 2. Set JAVA_HOME

**Requirement**: JAVA_HOME environment variable must be set correctly

**Validation Command**: `echo $JAVA_HOME`

**Expected Output**: Valid Java installation path (e.g., `/usr/lib/jvm/java-21-openjdk`)

**Ansible Task**: `roles/common/tasks/java.yml`

### 3. Create Service Account

**Requirement**: Create dedicated service account for Ping Identity components

**Account Details**:
- **Username**: `pingIdentity`
- **Group**: `pingIdentity`
- **Home Directory**: `/home/pingIdentity` (or as per organization standards)
- **Permissions**: Required sudo access for installation tasks

**Ansible Task**: `roles/common/tasks/prerequisites.yml`

### 4. Create Installation Directories

**Requirement**: Create installation directories with correct ownership

**Directories**:
- `/opt/ping/am` - AM installation directory
- `/opt/ping/ds` - DS installation directory
- `/opt/ping/idm` - IDM installation directory

**Ownership**: `pingIdentity:pingIdentity`
**Permissions**: Read/write for pingIdentity user

**Ansible Task**: `roles/common/tasks/prerequisites.yml`

### 5. Network/Ports

**Note**: Network team will handle firewall/network configuration, but validation is required

**Required Ports**:

**PingAM**:
- HTTP: 8080
- HTTPS: 8443

**PingDS** (sample ports):
- Admin port: 4444
- LDAP: 1389
- LDAPS: 1636
- HTTP: 18081
- HTTPS: 18443

**PingIDM**:
- HTTP: 8080
- HTTPS: 8443

**Inter-Server Communication**:
- AM → DS (Config Store, CTS, IDRepo)
- IDM → DS (IDRepo)
- Replication ports between DS replicas

**Ansible Task**: `roles/validation/tasks/ports.yml`

### 6. Security/Truststore

**Requirement**: Create dedicated truststore for AM to trust DS

**Steps**:
1. Create truststore directory
2. Copy Java cacerts as base
3. Import DS certificates
4. Configure Tomcat to use truststore via CATALINA_OPTS
5. Ensure pingIdentity user has read permissions

**Ansible Task**: `roles/common/tasks/truststore.yml`

## Pre-Deployment Validation

### Validation Checklist

All of the following MUST pass before deployment:

1. ✅ **Java Version**: OpenJDK 17 or 21 installed
2. ✅ **JAVA_HOME**: Set correctly
3. ✅ **Service Account**: pingIdentity user/group exists
4. ✅ **Installation Directories**: Created with correct ownership
5. ✅ **Ports**: All required ports are available
6. ✅ **NTP**: Time synchronization active
7. ✅ **DNS**: FQDN and reverse DNS resolution working
8. ✅ **Connectivity**: All VMs are reachable
9. ✅ **Truststore**: Created and configured
10. ✅ **Software Binaries**: All required software files present

### Connectivity Validation

**Ansible Task**: `roles/validation/tasks/connectivity.yml`

### NTP Synchronization

**Ansible Task**: `roles/validation/tasks/prerequisites.yml`

### Software Binary Validation

**Ansible Task**: `roles/validation/tasks/software.yml`

## Infrastructure VMs Setup

### Tomcat Infrastructure VMs

**Purpose**: Dedicated VMs for Tomcat installation

**Requirements**:
- Tomcat 10.1.46
- Owned by pingIdentity user
- Systemd service configured
- Ports configured

**Ansible Task**: `roles/infrastructure/tasks/install_tomcat.yml`

### JDK Infrastructure VMs

**Purpose**: Dedicated VMs for JDK installation

**Requirements**:
- OpenJDK 21 (or 17)
- JAVA_HOME configured
- Available system-wide

**Ansible Task**: `roles/infrastructure/tasks/install_jdk.yml`

## Validation Playbook

### Complete Validation Role

**File**: `roles/validation/tasks/main.yml`

## Execution Order

### 1. Environment Preparation (run first)

- Create service account (pingIdentity)
- Create installation directories:
  - `/opt/ping`
  - `/opt/ping/am`
  - `/opt/ping/idm`
  - `/opt/ping/ds`
  - `/opt/ping/ig`
- Install required OS packages (curl, unzip)
- Configure hostname + DNS (network team will take care)
- Configure NTP/time synchronization (network team will take care)
- Create truststore directory (certs added later)
- Validate OS version, CPU, RAM, disk space

**Ansible Task**: `roles/common/tasks/prerequisites.yml`

### 2. Infrastructure Deployment (run second)

- Install Java JDK (17 or 21 based on Ping docs)
- Set JAVA_HOME and PATH
- Install Tomcat (10.x)
- Deploy baseline Tomcat configs (server.xml, setenv.sh)
- Open system firewall ports (8080/8443 etc.) - Network team will take care
- Verify Tomcat starts successfully

**Ansible Task**: `roles/infrastructure/tasks/main.yml`

### 3. Pre-Deployment Validation (must pass)

- Validate connectivity to all nodes (ping/SSH)
- Validate required ports are free and not in use
- Validate Java version is correct
- Validate Tomcat installation health
- Validate truststore directory exists
- Validate permissions/ownership for installation directories
- Validate enough disk size
- Validate Vault connectivity (for pulling secrets)

**Ansible Task**: `roles/validation/tasks/main.yml`

### 4. Component Deployment (run only if validation passes)

1. Deploy DS (Config Store → CTS → IDRepo → replication)
2. Deploy AM (WAR + Amster configuration) → two servers
3. Deploy IDM (application + config + AD connector) → two servers
4. Deploy IG (optional)
5. Deploy UI (optional)

**Ansible Task**: `playbooks/deploy-ds.yml`, `playbooks/deploy-am.yml`, `playbooks/deploy-idm.yml`

## Deployment Order Summary

**Environment Prep → Infrastructure → Validation → DS → AM → IDM → IG/UI**

## Failure Handling

If any validation check fails:
- **Deployment will STOP**
- Error message will indicate which check failed
- Fix the issue and re-run validation
- Deployment will not proceed until all checks pass

## Next Steps

After environment preparation is complete:
- Refer to **03-project-structure.md** for Ansible project organization
- Refer to **04-prerequisites-setup.md** for Vault and inventory setup
- Refer to **05-execution-plan.md** for deployment execution

