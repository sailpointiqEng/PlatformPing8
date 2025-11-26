# Service Accounts and Vault Secrets

## Overview

This document outlines all service accounts that will be created during installation and all secrets that must be stored in HashiCorp Vault before executing the deployment. This ensures secure credential management and proper account provisioning.

## Service Accounts Created During Installation

### OS-Level Service Accounts

#### 1. pingIdentity (Primary Service Account)

**Account Name**: `pingIdentity`

**Group**: `pingIdentity`

**Home Directory**: `/home/pingIdentity`

**Created By**: `roles/common/tasks/prerequisites.yml`

**Purpose/Reason**:
- Primary service account for all Ping Identity components
- Owns all installation directories (`/opt/ping/am`, `/opt/ping/ds`, `/opt/ping/idm`, `/opt/ping/ig`)
- Runs all Ping Identity services (DS, AM, IDM, IG)
- Required for security isolation and proper file ownership
- Prevents running services as root user

**Permissions**:
- Sudo access (NOPASSWD) for installation tasks
- Read/write access to installation directories
- Execute permissions for component binaries

**When Created**: During Environment Preparation (Step 1)

---

## Application-Level Accounts

### Directory Services (DS) Accounts

#### 2. Directory Manager (DS Root Admin)

**Account Name**: `cn=Directory Manager`

**Created By**: DS setup command during installation

**Purpose/Reason**:
- Root administrator for all DS instances (Config Store, CTS, IDRepo)
- Required for initial DS setup and configuration
- Used for administrative operations (backup, restore, replication)
- Manages DS instance lifecycle

**Stored In**: Vault (must exist before installation)

**Vault Path**: `secret/ping/platform8/ds/admin_password`

**When Used**: During DS installation and administrative operations

---

#### 3. DS Monitor User

**Account Name**: `cn=monitor`

**Created By**: DS setup command during installation

**Purpose/Reason**:
- Read-only monitoring account for DS instances
- Used for health checks and monitoring tools
- Provides limited access for operational monitoring
- Does not have write permissions

**Stored In**: Vault (must exist before installation)

**Vault Path**: `secret/ping/platform8/ds/monitor_password`

**When Used**: During DS installation and monitoring operations

---

#### 4. DS Deployment ID Account

**Account Name**: Deployment ID (unique identifier)

**Deployment ID**: `AX8fkJybs4nP3qfAXN3CMw4BCYspGQ5CBVN1bkVDAOgwVKjG2Wo2ZTs` (example)

**Created By**: DS setup command during installation

**Purpose/Reason**:
- Used for certificate management and export
- Required for secure communication between DS instances
- Used by `dskeymgr` for certificate operations
- Enables secure replication setup

**Stored In**: Vault (must exist before installation)

**Vault Paths**:
- `secret/ping/platform8/ds/deployment_id`
- `secret/ping/platform8/ds/deployment_id_password`

**When Used**: During DS installation, certificate export, and replication setup

---

#### 5. AM Config Admin Account

**Account Name**: `uid=am-config,ou=admins,ou=am-config`

**Created By**: DS setup command (Config Store profile)

**Purpose/Reason**:
- Administrative account for AM configuration store
- Used by AM to read/write configuration data
- Required for AM to connect to DS Config Store
- Manages AM configuration lifecycle

**Stored In**: Vault (must exist before installation)

**Vault Path**: `secret/ping/platform8/am/admin_password` (shared with AM admin)

**When Used**: During AM installation and configuration

---

#### 6. AM CTS Admin Account

**Account Name**: `uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens`

**Created By**: DS setup command (CTS profile)

**Purpose/Reason**:
- Administrative account for AM Core Token Service (CTS)
- Used by AM to store and retrieve sessions/tokens
- Required for AM session management
- Manages CTS data lifecycle

**Stored In**: Vault (must exist before installation)

**Vault Path**: `secret/ping/platform8/am/admin_password` (shared with AM admin)

**When Used**: During AM installation and CTS configuration

---

#### 7. AM Identity Store Bind Account

**Account Name**: `uid=am-identity-bind-account,ou=admins,ou=identities`

**Created By**: DS setup command (IDRepo profile)

**Purpose/Reason**:
- Bind account for AM to access identity repository
- Used by AM to read user/group data from DS IDRepo
- Required for AM authentication and authorization
- Manages identity data access

**Stored In**: Vault (must exist before installation)

**Vault Path**: `secret/ping/platform8/am/admin_password` (shared with AM admin)

**When Used**: During AM installation and identity repository configuration

---

### Access Management (AM) Accounts

#### 8. AM Admin (amadmin)

**Account Name**: `amadmin`

**Created By**: Amster during AM installation

**Purpose/Reason**:
- Primary administrative account for AM
- Used for AM console access and REST API operations
- Required for all AM configuration tasks
- Manages AM realm configuration, OAuth2 clients, authentication trees

**Stored In**: Vault (must exist before installation)

**Vault Path**: `secret/ping/platform8/am/admin_password`

**When Used**: During AM installation, Amster configuration, and REST API operations

---

#### 9. Amster Key Passphrase

**Account Name**: N/A (key passphrase, not a user account)

**Created By**: Amster during AM installation

**Purpose/Reason**:
- Passphrase for Amster SSH key pair
- Used to secure Amster configuration operations
- Required for Amster to authenticate to AM
- Enables automated AM configuration

**Stored In**: Vault (must exist before installation)

**Vault Path**: `secret/ping/platform8/am/amster_key_passphrase`

**When Used**: During AM installation and Amster configuration

---

### Identity Management (IDM) Accounts

#### 10. IDM Admin (openidm-admin)

**Account Name**: `openidm-admin`

**Created By**: IDM during installation

**Purpose/Reason**:
- Primary administrative account for IDM
- Used for IDM console access and REST API operations
- Required for IDM configuration and AD connector setup
- Manages IDM reconciliation jobs and workflows

**Stored In**: Vault (must exist before installation)

**Vault Path**: `secret/ping/platform8/idm/admin_password`

**When Used**: During IDM installation, configuration, and AD connector setup

---

### Active Directory (AD) Integration Accounts

#### 11. AD Bind Account

**Account Name**: AD service account (e.g., `CN=IDM Service Account,OU=Service Accounts,DC=example,DC=com`)

**Created By**: Must exist in AD before installation (not created by Ansible)

**Purpose/Reason**:
- Service account in Active Directory for IDM connector
- Used by IDM to bind to AD and read user/group data
- Required for AD → IDM → DS data synchronization
- Must have appropriate AD permissions (read users, groups)

**Stored In**: Vault (must exist before installation)

**Vault Paths**:
- `secret/ping/platform8/ad/bind_dn`
- `secret/ping/platform8/ad/bind_password`
- `secret/ping/platform8/ad/base_dn`

**When Used**: During AD connector configuration in IDM

**Note**: This account must be created in AD by AD administrators before deployment

---

## Vault Secrets Required Before Installation

### Complete Vault Secret Structure

All secrets must be stored in HashiCorp Vault before executing any deployment playbooks.

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
│   └── ad_connector_password           # AD connector bind password (same as ad/bind_password)
├── ad/
│   ├── bind_dn                         # AD bind DN (service account)
│   ├── bind_password                   # AD bind password
│   └── base_dn                         # AD base DN for user/group search
└── common/
    ├── truststore_password             # Java truststore password (default: changeit)
    └── keystore_password               # Keystore password (if using custom keystores)
```

### Detailed Vault Secret Requirements

#### DS Secrets

| Secret Path | Description | Required For | Example Value |
|------------|------------|--------------|---------------|
| `secret/ping/platform8/ds/admin_password` | DS Directory Manager password | DS installation, replication, administrative operations | `SecurePassword123!` |
| `secret/ping/platform8/ds/deployment_id` | DS deployment ID | Certificate management, replication | `AX8fkJybs4nP3qfAXN3CMw4BCYspGQ5CBVN1bkVDAOgwVKjG2Wo2ZTs` |
| `secret/ping/platform8/ds/deployment_id_password` | DS deployment ID password | Certificate export, replication setup | `DeploymentIDPass456!` |
| `secret/ping/platform8/ds/monitor_password` | DS monitor user password | Health checks, monitoring | `MonitorPass789!` |

#### AM Secrets

| Secret Path | Description | Required For | Example Value |
|------------|------------|--------------|---------------|
| `secret/ping/platform8/am/admin_password` | AM amadmin password | AM installation, Amster config, REST API | `AMAdminPass123!` |
| `secret/ping/platform8/am/amster_key_passphrase` | Amster key passphrase | Amster SSH key encryption | `AmsterKeyPass456!` |

**Note**: AM admin password is also used for:
- AM Config Store admin account
- AM CTS admin account
- AM Identity Store bind account

#### IDM Secrets

| Secret Path | Description | Required For | Example Value |
|------------|------------|--------------|---------------|
| `secret/ping/platform8/idm/admin_password` | IDM admin password | IDM installation, REST API, AD connector config | `IDMAdminPass123!` |
| `secret/ping/platform8/idm/ad_connector_password` | AD connector bind password | AD connector configuration | `ADBindPass456!` |

**Note**: `ad_connector_password` should be the same as `ad/bind_password`

#### AD Secrets

| Secret Path | Description | Required For | Example Value |
|------------|------------|--------------|---------------|
| `secret/ping/platform8/ad/bind_dn` | AD bind DN | AD connector configuration | `CN=IDM Service Account,OU=Service Accounts,DC=example,DC=com` |
| `secret/ping/platform8/ad/bind_password` | AD bind password | AD connector authentication | `ADBindPass456!` |
| `secret/ping/platform8/ad/base_dn` | AD base DN | User/group search base | `DC=example,DC=com` |

**Note**: AD service account must be created in AD before deployment

#### Common Secrets

| Secret Path | Description | Required For | Example Value |
|------------|------------|--------------|---------------|
| `secret/ping/platform8/common/truststore_password` | Java truststore password | Truststore operations | `changeit` (default) |
| `secret/ping/platform8/common/keystore_password` | Keystore password | Custom keystore operations | `KeystorePass123!` (if using custom keystores) |

---

## Account Creation Timeline

### During Environment Preparation (Step 1)

1. **pingIdentity** (OS service account)
   - Created by: `roles/common/tasks/prerequisites.yml`
   - Purpose: Owns all installation directories and runs services

### During Infrastructure Deployment (Step 2)

No new accounts created (Java and Tomcat installation only)

### During Component Deployment (Step 4)

#### DS Installation

2. **Directory Manager** (`cn=Directory Manager`)
   - Created by: DS setup command
   - Uses: `secret/ping/platform8/ds/admin_password` from Vault

3. **DS Monitor User** (`cn=monitor`)
   - Created by: DS setup command
   - Uses: `secret/ping/platform8/ds/monitor_password` from Vault

4. **DS Deployment ID Account**
   - Created by: DS setup command
   - Uses: `secret/ping/platform8/ds/deployment_id` and `deployment_id_password` from Vault

5. **AM Config Admin Account**
   - Created by: DS setup command (Config Store profile)
   - Uses: `secret/ping/platform8/am/admin_password` from Vault

6. **AM CTS Admin Account**
   - Created by: DS setup command (CTS profile)
   - Uses: `secret/ping/platform8/am/admin_password` from Vault

7. **AM Identity Store Bind Account**
   - Created by: DS setup command (IDRepo profile)
   - Uses: `secret/ping/platform8/am/admin_password` from Vault

#### AM Installation

8. **AM Admin** (`amadmin`)
   - Created by: Amster during AM installation
   - Uses: `secret/ping/platform8/am/admin_password` from Vault

9. **Amster Key Passphrase**
   - Created by: Amster during AM installation
   - Uses: `secret/ping/platform8/am/amster_key_passphrase` from Vault

#### IDM Installation

10. **IDM Admin** (`openidm-admin`)
    - Created by: IDM during installation
    - Uses: `secret/ping/platform8/idm/admin_password` from Vault

#### AD Integration (Post-Deployment)

11. **AD Bind Account**
    - Must exist in AD before deployment
    - Uses: `secret/ping/platform8/ad/bind_dn` and `bind_password` from Vault
    - Configured in IDM during AD connector setup

---

## Pre-Installation Checklist

Before executing any deployment playbooks, ensure:

### Vault Secrets

- [ ] All DS secrets are stored in Vault
- [ ] All AM secrets are stored in Vault
- [ ] All IDM secrets are stored in Vault
- [ ] All AD secrets are stored in Vault (if AD integration enabled)
- [ ] All common secrets are stored in Vault
- [ ] Vault AppRole is configured and credentials obtained
- [ ] Vault connectivity is tested

### Service Accounts

- [ ] AD service account is created in AD (if AD integration enabled)
- [ ] AD service account has required permissions (read users, groups)
- [ ] AD service account credentials are stored in Vault

### Verification

- [ ] Test Vault connection: `ansible-playbook playbooks/test-vault-connection.yml`
- [ ] Verify all secrets can be retrieved from Vault
- [ ] Confirm AD service account exists and is accessible (if AD integration enabled)

---

## Security Best Practices

1. **Password Complexity**: All passwords should meet organizational password policies
   - Minimum 12 characters
   - Mix of uppercase, lowercase, numbers, and special characters
   - Avoid dictionary words

2. **Password Rotation**: Implement regular password rotation for all accounts
   - Update passwords in Vault
   - Update component configurations
   - Test after rotation

3. **Least Privilege**: Service accounts should have minimum required permissions
   - OS account (pingIdentity): Only sudo for installation tasks
   - Application accounts: Only required permissions for their purpose

4. **Vault Access Control**: Limit Vault access to authorized personnel only
   - Use AppRole for automation (not user tokens)
   - Implement audit logging
   - Regular access reviews

5. **Secret Management**: Never store secrets in:
   - Git repositories
   - Plain text files
   - Environment variables in code
   - Configuration files

---

## Account Summary Table

| Account Name | Type | Created By | Vault Secret | Purpose |
|-------------|------|-----------|--------------|----------|
| pingIdentity | OS User | Ansible | N/A | Service account for all components |
| cn=Directory Manager | DS Admin | DS Setup | `ds/admin_password` | DS root administrator |
| cn=monitor | DS Monitor | DS Setup | `ds/monitor_password` | DS monitoring |
| Deployment ID | DS Cert | DS Setup | `ds/deployment_id` | Certificate management |
| uid=am-config | DS Admin | DS Setup | `am/admin_password` | AM config store admin |
| uid=openam_cts | DS Admin | DS Setup | `am/admin_password` | AM CTS admin |
| uid=am-identity-bind-account | DS Bind | DS Setup | `am/admin_password` | AM identity store bind |
| amadmin | AM Admin | Amster | `am/admin_password` | AM administrator |
| Amster Key | AM Key | Amster | `am/amster_key_passphrase` | Amster authentication |
| openidm-admin | IDM Admin | IDM | `idm/admin_password` | IDM administrator |
| AD Service Account | AD Account | AD Admin | `ad/bind_dn`, `ad/bind_password` | AD connector bind |

---

## Next Steps

- Refer to **04-prerequisites-setup.md** for Vault configuration
- Refer to **02-environment-preparation.md** for service account creation
- Refer to **05-execution-plan.md** for deployment execution order
- Refer to **06-component-deployment.md** for component-specific account usage

