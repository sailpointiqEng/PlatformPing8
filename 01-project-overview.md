# Ping Identity Platform 8 - Ansible Automation
## Project Overview

## Executive Summary

This project automates the deployment of Ping Identity Platform 8 components (Access Management/AM, Identity Management/IDM, Directory Services/DS, Identity Gateway/IG) using Ansible with HashiCorp Vault for secure credential management. The solution supports multi-VM deployments with replication, idempotent operations (fresh install and incremental updates), and automated post-deployment configuration.

## Project Objectives

1. **Automate Deployment**: Fully automate the deployment of AM, IDM, DS, and IG components on on-prem VMs using Ansible
2. **Secure Credential Management**: Integrate HashiCorp Vault to store and retrieve all credentials securely (no plain text)
3. **Idempotent Operations**: Support both fresh installations and incremental updates without overwriting existing configurations
4. **Multi-Environment Support**: Support dev, test, and production environments with environment-specific configurations
5. **Replication Support**: Enable replication for all components with each replica on separate servers
6. **Post-Deployment Automation**: Automate all post-deployment configuration including AD integration

## Requirements Analysis

### Core Requirements

1. **Automated Deployment**
   - Deploy AM, IDM, DS, and IG on on-prem VMs using Ansible playbooks
   - Support for multiple replicas (each on separate server)
   - Infrastructure VMs for Tomcat and JDK installation

2. **Vault Integration**
   - All credentials stored in HashiCorp Vault (no plain text in repository)
   - Use AppRole authentication for Ansible automation
   - Custom lookup plugin for secret retrieval

3. **Idempotent Behavior**
   - **Fresh Installation**: If component is not installed, perform full installation
   - **Re-deployment/Update**: If component is already installed:
     - Update only what is required
     - Do NOT overwrite existing configurations (configs, CTS, identity store, extensions)
     - Ensure existing system customizations remain intact
     - Only apply incremental changes

4. **Post-Deployment Configuration**
   - Automated execution of post-deployment scripts
   - Configuration of AM (Access Management)
   - Configuration of IDM (Identity Management)
   - Configuration of DS (config store, CTS, identity store)
   - Configuration of IG (Identity Gateway) routes and policies
   - DS identity store must be shared between AM and IDM

5. **AD Integration**
   - Configure AD connector in IDM
   - Set up data flow: AD → IDM → DS
   - User and group data synchronization

6. **Environment Preparation**
   - Pre-deployment validation (connectivity, ports, prerequisites)
   - Java version verification (OpenJDK 17 or 21)
   - JAVA_HOME configuration
   - Service account creation (pingIdentity user/group)
   - Installation directories setup
   - Network/ports validation
   - Truststore setup

## Component Architecture

### Directory Services (DS)
- **Version**: 8.0.0
- **Instances**:
  - **Config Store**: AM configuration storage
  - **CTS (Core Token Service)**: Session and token storage
  - **IDRepo**: Identity repository (shared between AM and IDM)
- **Replication**: Support for multiple replicas per instance type
- **Ports**: LDAP, LDAPS, HTTP, HTTPS, Admin connector ports

### Access Management (AM)
- **Version**: 8.0.1
- **Deployment**: WAR file deployed to Tomcat
- **Configuration Modes**:
  - File-Based Configuration (FBC) mode
  - DS-based configuration mode
- **Features**: Authentication trees, OAuth2/OIDC, SAML
- **Dependencies**: Tomcat infrastructure VM, DS Config Store, DS IDRepo

### Identity Management (IDM)
- **Version**: 8.0.0
- **Purpose**: Identity lifecycle management and user self-service
- **Capabilities**: User provisioning, workflow, compliance
- **Dependencies**: DS IDRepo (shared with AM), AD connector
- **Data Flow**: AD → IDM → DS

### Identity Gateway (IG)
- **Version**: 2025.3.0
- **Purpose**: Reverse proxy and API gateway
- **Functions**: Authentication enforcement, request/response transformation
- **Dependencies**: AM, IDM

### Platform UI
- **Version**: 8.0.1.0523
- **Purpose**: Web-based user interfaces
- **Components**: Admin console, End-user portal, Login pages
- **Optional**: Can be deployed if needed

## Architecture Overview

### Deployment Model with Load Balancing and Replication

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    CLIENT TRAFFIC                                            │
│                              (Users, Applications, APIs)                                     │
└────────────────────────────────────────┬────────────────────────────────────────────────────┘
                                         │
                                         ▼
                    ┌───────────────────────────────────────┐
                    │   Identity Gateway Load Balancer       │
                    │         (IG LB) Port: 443/9443         │
                    └───────────────┬───────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
        ┌──────────────────────┐        ┌──────────────────────┐
        │   Identity Gateway   │        │   Identity Gateway   │
        │        (IG-01)       │◄─Rep─►│        (IG-02)       │
        │   Port: 7080/9443    │        │   Port: 7080/9443    │
        └───────────┬───────────┘        └───────────┬───────────┘
                     │                                │
                     │                                │
                     │  ◄─────────────────────────────┼─────────────────────────►
                     │  Bidirectional Connection      │
                     │                                │
                     ▼                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         APPLICATION LAYER                                   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │              Access Management Load Balancer (AM LB)                 │ │
│  │                        Port: 8081/8443                                │ │
│  └────────────────────────────┬─────────────────────────────────────────┘ │
│                               │                                               │
│                ┌──────────────┴──────────────┐                              │
│                │                             │                              │
│                ▼                             ▼                              │
│    ┌──────────────────────┐      ┌──────────────────────┐                   │
│    │ Access Management   │      │ Access Management   │                   │
│    │      (AM-01)        │◄─Rep─►│      (AM-02)        │                   │
│    │  Port: 8081/8443    │      │  Port: 8081/8443    │                   │
│    └──────────┬──────────┘      └──────────┬──────────┘                   │
│               │                             │                              │
│               │  AM Connections:            │                              │
│               │  • Config LB                │                              │
│               │  • CTS LB                   │                              │
│               │  • IdentityStore LB         │                              │
│               │                             │                              │
│               └────────────┬────────────────┘                              │
│                            │                                               │
│                            │                                               │
│  ┌─────────────────────────┴──────────────────────────────────────────┐ │
│  │         Identity Management Load Balancer (IDM LB)                    │ │
│  │                        Port: 8080/8553                                 │ │
│  └────────────────────────────┬─────────────────────────────────────────┘ │
│                               │                                               │
│                ┌──────────────┴──────────────┐                              │
│                │                             │                              │
│                ▼                             ▼                              │
│    ┌──────────────────────┐      ┌──────────────────────┐                   │
│    │ Identity Management  │      │ Identity Management │                   │
│    │      (IDM-01)        │◄─Rep─►│      (IDM-02)        │                   │
│    │  Port: 8080/8553     │      │  Port: 8080/8553     │                   │
│    └──────────┬───────────┘      └──────────┬───────────┘                   │
│               │                             │                              │
│               │  IDM Connections:            │                              │
│               │  • IdentityStore LB (Shared) │                              │
│               │  • AD Connector → AD         │                              │
│               │                             │                              │
│               └────────────┬────────────────┘                              │
│                            │                                               │
│                            │  (AM and IDM do NOT connect to each other)    │
└────────────────────────────┼───────────────────────────────────────────────┘
                             │
                             │
┌────────────────────────────┼───────────────────────────────────────────────┐
│                    DIRECTORY SERVICES LAYER                                  │
│                             │                                               │
│  ┌──────────────────────────┴────────────────────────────────────────────┐ │
│  │              DS Config Store Load Balancer (Config LB)                  │ │
│  │                    Port: 38081/38443                                    │ │
│  │                    Connected by: AM ONLY                                │ │
│  └────────────────────────────┬──────────────────────────────────────────┘ │
│                               │                                             │
│                ┌──────────────┴──────────────┐                            │
│                │                             │                            │
│                ▼                             ▼                            │
│    ┌──────────────────────┐      ┌──────────────────────┐                │
│    │  DS Config Store      │      │  DS Config Store      │                │
│    │     (config01)        │◄─Rep─►│     (config02)        │                │
│    │ Port: 38081/38443     │      │ Port: 38081/38443     │                │
│    └───────────────────────┘      └───────────────────────┘                │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │              DS CTS Load Balancer (CTS LB)                          │ │
│  │                    Port: 18081/18443                                  │ │
│  │                    Connected by: AM ONLY                              │ │
│  └────────────────────────────┬─────────────────────────────────────────┘ │
│                               │                                             │
│                ┌──────────────┴──────────────┐                            │
│                │                             │                            │
│                ▼                             ▼                            │
│    ┌──────────────────────┐      ┌──────────────────────┐                │
│    │   DS CTS (Tokens)     │      │   DS CTS (Tokens)     │                │
│    │      (cts01)          │◄─Rep─►│      (cts02)          │                │
│    │ Port: 18081/18443     │      │ Port: 18081/18443     │                │
│    └───────────────────────┘      └───────────────────────┘                │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │         DS Identity Store Load Balancer (IdentityStore LB)            │ │
│  │                    Port: 28081/28443                                   │ │
│  │                    Connected by: AM (reads), IDM (writes) - SHARED     │ │
│  └────────────────────────────┬─────────────────────────────────────────┘ │
│                               │                                             │
│                ┌──────────────┴──────────────┐                            │
│                │                             │                            │
│                ▼                             ▼                            │
│    ┌──────────────────────┐      ┌──────────────────────┐                │
│    │   DS Identity Store   │      │   DS Identity Store   │                │
│    │  (identityStore01)    │◄─Rep─►│  (identityStore02)    │                │
│    │ Port: 28081/28443     │      │ Port: 28081/28443     │                │
│    └───────────────────────┘      └───────────────────────┘                │
└─────────────────────────────────────────────────────────────────────────────┘
                             │
                             │
┌────────────────────────────┼─────────────────────────────────────────────┐
│                      INTEGRATION LAYER                                    │
│                              │                                             │
│                              ▼                                             │
│              ┌───────────────────────────────┐                            │
│              │   Active Directory (AD)        │                            │
│              │   Users and Groups             │                            │
│              └───────────────┬───────────────┘                            │
│                              │                                             │
│                              │  (AD Connector Sync)                        │
│                              │                                             │
│                              ▼                                             │
│              ┌───────────────────────────────┐                            │
│              │   AD Connector                  │                            │
│              │   (in IDM)                      │                            │
│              └───────────────┬───────────────┘                            │
│                              │                                             │
│                              │  (Writes Identity Data)                     │
│                              │                                             │
│                              ▼                                             │
│              ┌───────────────────────────────┐                            │
│              │   IDM (via IDM LB)             │                            │
│              │   (User Provisioning)          │                            │
│              └───────────────┬───────────────┘                            │
│                              │                                             │
│                              │  (Writes to IdentityStore LB)              │
│                              │                                             │
│                              ▼                                             │
│              ┌───────────────────────────────┐                            │
│              │   IdentityStore LB              │                            │
│              │   (identityStore01, identityStore02)                        │
│              └───────────────┬───────────────┘                            │
│                              │                                             │
│                              │  (Reads Identity Data)                     │
│                              │                                             │
│                              ▼                                             │
│              ┌───────────────────────────────┐                            │
│              │   AM (via AM LB)              │                            │
│              │   (Authentication)            │                            │
│              └──────────────────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────────┘

Connection Summary:
  • IG LB (IG-01, IG-02) ◄───► AM LB (AM-01, AM-02) - Bidirectional
  • AM → Config LB → config01, config02
  • AM → CTS LB → cts01, cts02
  • AM → IdentityStore LB → identityStore01, identityStore02 (reads)
  • IDM → IdentityStore LB → identityStore01, identityStore02 (writes) - SHARED
  • IDM → AD Connector → AD (users and groups)
  • AM and IDM: NO direct connection (communicate via IdentityStore)

Legend:
  ───►  Traffic Flow
  ◄───►  Bidirectional Connection
  ◄─Rep─►  Replication (Bidirectional)
  ────  Load Balancer Distribution
```

### Data Flow

#### Client Request Flow
1. **Client → IG Load Balancer → Identity Gateway**
   - All client traffic (users, applications, APIs) first hits the Identity Gateway Load Balancer
   - Load balancer distributes traffic across IG-01 and IG-02 instances
   - IG instances handle SSL termination and route requests to backend services

2. **IG LB ↔ AM LB (Bidirectional Connection)**
   - IG Load Balancer (IG-01, IG-02) connects bidirectionally to AM Load Balancer (AM-01, AM-02)
   - IG routes authentication requests to AM Load Balancer
   - AM Load Balancer distributes traffic across AM-01 and AM-02 instances
   - Ensures high availability and load distribution

3. **IG → IDM Load Balancer → IDM Instances**
   - IG routes identity management requests to IDM Load Balancer
   - IDM Load Balancer distributes traffic across IDM-01 and IDM-02 instances

#### Access Management (AM) Connections
4. **AM → Config LB → config01, config02**
   - AM instances (AM-01, AM-02) connect to Config Load Balancer
   - Config LB distributes requests to config01 and config02
   - AM reads configuration data from replicated Config Store instances
   - Used for AM realm configuration, authentication trees, and policies

5. **AM → CTS LB → cts01, cts02**
   - AM instances (AM-01, AM-02) connect to CTS Load Balancer
   - CTS LB distributes requests to cts01 and cts02
   - AM stores sessions and tokens in replicated CTS instances
   - Enables session persistence and sharing across AM instances

6. **AM → IdentityStore LB → identityStore01, identityStore02**
   - AM instances (AM-01, AM-02) connect to IdentityStore Load Balancer
   - IdentityStore LB distributes requests to identityStore01 and identityStore02
   - AM reads identity data from replicated Identity Store instances
   - Used for user authentication and authorization

#### Identity Management (IDM) Connections
7. **IDM → IdentityStore LB → identityStore01, identityStore02 (SHARED)**
   - IDM instances (IDM-01, IDM-02) connect ONLY to IdentityStore Load Balancer
   - IdentityStore LB distributes requests to identityStore01 and identityStore02
   - IDM writes user data to replicated Identity Store instances
   - IDM does NOT connect to Config LB or CTS LB
   - Replication ensures data consistency across all Identity Store replicas
   - This is the SHARED connection: AM reads, IDM writes

8. **IDM → AD Connector → AD (Users and Groups)**
   - IDM instances connect to AD Connector (configured within IDM)
   - AD Connector synchronizes users and groups from Active Directory
   - AD Connector provisions synchronized data to IdentityStore via IdentityStore LB

#### Identity Synchronization Flow
9. **AD → AD Connector → IDM → IdentityStore LB → AM**
   - Active Directory users/groups are synchronized to IDM via AD Connector
   - IDM provisions users to shared IdentityStore (via IdentityStore LB)
   - AM reads from same IdentityStore (via IdentityStore LB) for authentication
   - AM and IDM do NOT connect directly - they communicate via IdentityStore
   - Replication ensures all AM instances see consistent identity data

#### Component Isolation
10. **AM and IDM Separation**
    - AM and IDM are completely separate and do NOT connect to each other
    - AM connects to: Config LB, CTS LB, and IdentityStore LB
    - IDM connects to: IdentityStore LB ONLY (and AD via AD Connector)
    - IdentityStore is the shared component that enables AM and IDM to work together

#### Replication
11. **Component Replication**
    - All components have 2 replicas: IG-01/IG-02, AM-01/AM-02, IDM-01/IDM-02
    - DS instances: config01/config02, cts01/cts02, identityStore01/identityStore02
    - Replication is bidirectional and automatic
    - Ensures high availability and data consistency
    - Load balancers handle failover automatically

3. **Replication**
   - Each DS instance type (Config, CTS, IDRepo) can have multiple replicas
   - Each replica runs on a separate server
   - Replication can be automated or configured manually

## Environment Support

### Development Environment
- Multiple replicas for testing replication
- Full infrastructure setup
- Production-like configuration

### Test Environment
- Multiple replicas for testing replication
- Full infrastructure setup
- Production-like configuration

### Production Environment
- Full replication for high availability
- Multiple infrastructure VMs
- Production-grade security and monitoring

## Key Design Principles

1. **Idempotency**: All tasks must be idempotent - safe to run multiple times
2. **Configuration Preservation**: Never overwrite existing configs, CTS, identity store, or extensions
3. **Incremental Updates**: Only apply changes that are necessary
4. **Security First**: All credentials stored in Vault, never in plain text
5. **Validation Before Deployment**: All prerequisites must be validated before deployment
6. **Flexibility**: Support both automated and manual replication setup
7. **Multi-VM Ready**: Inventory structure supports distributed deployments
8. **Replication Support**: All components support multiple replicas
9. **Environment Separation**: Clear separation between dev, test, and production
10. **Infrastructure Isolation**: Separate VMs for Tomcat and JDK

## Success Criteria

- ✅ All components deploy successfully via Ansible
- ✅ No credentials stored in plain text
- ✅ Idempotent operations work correctly (fresh install and updates)
- ✅ Replication configured successfully
- ✅ Post-deployment configuration automated
- ✅ AD integration functional
- ✅ All environments (dev, test, production) supported
- ✅ Pre-deployment validation prevents deployment failures

## Project Deliverables

1. **Ansible Playbooks and Roles**: Complete automation code
2. **Inventory Files**: Multi-environment inventory structure
3. **Vault Integration**: Custom lookup plugin and configuration
4. **Documentation**: Comprehensive documentation (8 files)
5. **Validation Scripts**: Pre-deployment validation procedures
6. **Configuration Templates**: Jinja2 templates for all configurations

## Next Steps

Refer to the following documentation files for detailed implementation:

- **02-environment-preparation.md**: Environment setup and validation
- **03-project-structure.md**: Complete Ansible project structure
- **04-prerequisites-setup.md**: Prerequisites and dependencies
- **05-execution-plan.md**: Step-by-step execution procedures
- **06-component-deployment.md**: Component deployment details
- **07-post-deployment.md**: Post-deployment configuration
- **08-reference-guide.md**: Reference materials and troubleshooting
- **09-service-accounts-and-vault-secrets.md**: Service accounts and Vault secrets reference
- **10-vault-secrets-setup.md**: Vault secrets setup instructions (MUST BE DONE FIRST)

