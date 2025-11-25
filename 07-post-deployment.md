# Post-Deployment Configuration

## Overview

This document covers all post-deployment configuration tasks including AM REST API configuration, IDM integration, AD connector setup, and verification procedures.

## Post-Deployment Workflow

### Execution Order

1. **AM REST Configuration** - Configure AM via REST API
2. **IDM Integration** - Configure IDM integration service in AM
3. **AD Connector Setup** - Configure AD connector in IDM
4. **Verification** - Verify all integrations

## AM REST API Configuration

### AM Admin Login

**Purpose**: Authenticate as AM admin to obtain session token

**File**: `roles/am/tasks/rest_config.yml`

```yaml
---
- name: AM Admin Login
  uri:
    url: "{{ am_url }}/json/realms/root/authenticate"
    method: POST
    headers:
      Content-Type: "application/json"
      Accept-API-Version: "resource=2.0, protocol=1"
      X-OpenAM-Username: "amadmin"
      X-OpenAM-Password: "{{ vault('secret/ping/platform8/am/admin_password') }}"
    body_format: json
    body: {}
    status_code: [200, 201]
  register: am_login_response
  no_log: true

- name: Extract admin token
  set_fact:
    am_admin_token: "{{ am_login_response.json.tokenId }}"
  no_log: true
```

### Configure Alpha Realm Repository

**Purpose**: Configure identity repository for Alpha realm

```yaml
- name: Configure Alpha Realm Repository
  uri:
    url: "{{ am_url }}/json/realms/root/realms/alpha/realm-config/services/id-repositories/LDAPv3ForOpenDS/OpenDJ"
    method: PUT
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=2.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "_id": "OpenDJ"
      "ldapsettings":
        "openam-idrepo-ldapv3-mtls-enabled": false
        "openam-idrepo-ldapv3-heartbeat-timeunit": "SECONDS"
        "sun-idrepo-ldapv3-config-connection_pool_min_size": 1
        "sun-idrepo-ldapv3-config-search-scope": "SCOPE_ONE"
        "openam-idrepo-ldapv3-proxied-auth-enabled": false
        "sun-idrepo-ldapv3-config-max-result": 1000
        "sun-idrepo-ldapv3-config-organization_name": "{{ ds_idrepo_dn }}"
        "sun-idrepo-ldapv3-config-authid": "uid=am-identity-bind-account,ou=admins,{{ ds_idrepo_dn }}"
        "openam-idrepo-ldapv3-heartbeat-interval": 10
        "sun-idrepo-ldapv3-config-trust-all-server-certificates": false
        "sun-idrepo-ldapv3-config-connection-mode": "LDAPS"
        "openam-idrepo-ldapv3-affinity-enabled": false
        "sun-idrepo-ldapv3-config-ldap-server":
          - "{{ ds_idrepo_server }}:{{ ds_idrepo_server_ldaps_port }}"
        "sun-idrepo-ldapv3-config-time-limit": 10
        "sun-idrepo-ldapv3-config-connection_pool_max_size": 10
      "userconfig":
        "sun-idrepo-ldapv3-config-users-search-filter": "(objectclass=inetorgperson)"
        "sun-idrepo-ldapv3-config-users-search-attribute": "fr-idm-uuid"
        "sun-idrepo-ldapv3-config-people-container-value": "people"
        "sun-idrepo-ldapv3-config-isactive": "inetuserstatus"
        "sun-idrepo-ldapv3-config-people-container-name": "ou"
        "sun-idrepo-ldapv3-config-active": "Active"
      "groupconfig":
        "sun-idrepo-ldapv3-config-groups-search-filter": "(objectclass=groupOfUniqueNames)"
        "sun-idrepo-ldapv3-config-groups-search-attribute": "cn"
        "sun-idrepo-ldapv3-config-group-container-value": "groups"
      "authentication":
        "sun-idrepo-ldapv3-config-auth-naming-attr": "uid"
    status_code: [200, 201]
  register: alpha_repo_config
```

### Delete OpenDJ from Root Realm

**Purpose**: Remove OpenDJ configuration from root realm (only use in Alpha realm)

```yaml
- name: Delete OpenDJ from Root Realm
  uri:
    url: "{{ am_url }}/json/realms/root/realm-config/services/id-repositories/LDAPv3ForOpenDS/OpenDJ"
    method: DELETE
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=2.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    status_code: [200, 201, 404]
```

### Create OAuth2 Clients

#### IDM Resource Server

```yaml
- name: Create IDM Resource Server OAuth2 Client
  uri:
    url: "{{ am_url }}/json/realms/root/realm-config/agents/OAuth2Client/idm-resource-server"
    method: PUT
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=2.0,resource=1.0"
      If-None-Match: "*"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "coreOAuth2ClientConfig":
        "defaultScopes": []
        "redirectionUris": []
        "scopes":
          - "am-introspect-all-tokens"
          - "am-introspect-all-tokens-any-realm"
        "userpassword": "{{ vault('secret/ping/platform8/am/admin_password') }}"
    status_code: [200, 201]
```

#### IDM Provisioning Client (Alpha Realm)

```yaml
- name: Create IDM Provisioning OAuth2 Client (Alpha Realm)
  uri:
    url: "{{ am_url }}/json/realms/root/realms/alpha/realm-config/agents/OAuth2Client/idm-provisioning"
    method: PUT
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=2.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "coreOAuth2ClientConfig":
        "status":
          "inherited": false
          "value": "Active"
        "userpassword": "openidm"
        "clientType":
          "inherited": false
          "value": "Confidential"
        "scopes":
          "inherited": false
          "value":
            - "fr:idm:*"
        "defaultScopes":
          "inherited": false
          "value": []
      "advancedOAuth2ClientConfig":
        "grantTypes":
          "inherited": false
          "value":
            - "client_credentials"
        "tokenEndpointAuthMethod":
          "inherited": false
          "value": "client_secret_basic"
    status_code: [200, 201]
```

#### IDM Admin UI Client

```yaml
- name: Create IDM Admin UI OAuth2 Client (Alpha Realm)
  uri:
    url: "{{ am_url }}/json/realms/root/realms/alpha/realm-config/agents/OAuth2Client/idm-admin-ui"
    method: PUT
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=2.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "coreOAuth2ClientConfig":
        "status":
          "inherited": false
          "value": "Active"
        "clientType":
          "inherited": false
          "value": "Public"
        "redirectionUris":
          "inherited": false
          "value":
            - "http://{{ idm_hostname }}:{{ boot_port_http }}/platform/appAuthHelperRedirect.html"
            - "http://{{ idm_hostname }}:{{ boot_port_http }}/platform/sessionCheck.html"
            - "http://{{ idm_hostname }}:{{ boot_port_http }}/admin/appAuthHelperRedirect.html"
            - "http://{{ idm_hostname }}:{{ boot_port_http }}/admin/sessionCheck.html"
            - "https://{{ platform_hostname }}:{{ ig_https_port }}/platform/appAuthHelperRedirect.html"
            - "https://{{ platform_hostname }}:{{ ig_https_port }}/platform/sessionCheck.html"
            - "https://{{ platform_hostname }}:{{ ig_https_port }}/admin/appAuthHelperRedirect.html"
            - "https://{{ platform_hostname }}:{{ ig_https_port }}/admin/sessionCheck.html"
        "scopes":
          "inherited": false
          "value":
            - "openid"
            - "fr:idm:*"
      "advancedOAuth2ClientConfig":
        "grantTypes":
          "inherited": false
          "value":
            - "authorization_code"
            - "implicit"
        "tokenEndpointAuthMethod":
          "inherited": false
          "value": "none"
        "isConsentImplied":
          "inherited": false
          "value": true
    status_code: [200, 201]
```

### Configure IDM Integration Service

**Purpose**: Configure IDM integration service in AM

```yaml
- name: Configure IDM Integration Service
  uri:
    url: "{{ am_url }}/json/global-config/services/idm-integration"
    method: PUT
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=1.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "configurationCacheDuration": 0
      "idmDeploymentPath": "openidm"
      "idmDeploymentUrl": "http://{{ idm_hostname }}:{{ boot_port_http }}"
      "idmProvisioningClient": "idm-provisioning"
      "provisioningClientScopes":
        - "fr:idm:*"
      "enabled": true
      "useInternalOAuth2Provider": false
    status_code: [200, 201]
```

### Configure Validation Service

**Purpose**: Configure validation service for allowed redirect URLs

```yaml
- name: Configure Validation Service (Root Realm)
  uri:
    url: "{{ am_url }}/json/realms/root/realm-config/services/validation?_action=create"
    method: POST
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=1.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "validGotoDestinations":
        - "http://{{ admin_hostname }}:{{ admin_port }}/*"
        - "http://{{ login_hostname }}:{{ login_port }}/*"
        - "http://{{ enduser_hostname }}:{{ enduser_port }}/*"
        - "https://{{ platform_hostname }}:{{ ig_https_port }}/*"
        - "http://{{ idm_hostname }}:{{ boot_port_http }}/*"
    status_code: [200, 201]

- name: Configure Validation Service (Alpha Realm)
  uri:
    url: "{{ am_url }}/json/realms/root/realms/alpha/realm-config/services/validation?_action=create"
    method: POST
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=1.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "validGotoDestinations":
        - "http://{{ admin_hostname }}:{{ admin_port }}/*"
        - "http://{{ login_hostname }}:{{ login_port }}/*"
        - "http://{{ enduser_hostname }}:{{ enduser_port }}/*"
        - "https://{{ platform_hostname }}:{{ ig_https_port }}/*"
        - "http://{{ idm_hostname }}:{{ boot_port_http }}/*"
    status_code: [200, 201]
```

### Configure CORS Service

**Purpose**: Configure CORS for cross-origin requests

```yaml
- name: Create CORS Service
  uri:
    url: "{{ am_url }}/json/global-config/services/CorsService/configuration?_action=create"
    method: POST
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=1.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "acceptedHeaders":
        - "accept-api-version"
        - "authorization"
        - "cache-control"
        - "content-type"
        - "if-match"
        - "if-none-match"
        - "user-agent"
        - "x-forgerock-transactionid"
        - "x-openidm-nosession"
        - "x-openidm-password"
        - "x-openidm-username"
        - "x-requested-with"
      "exposedHeaders":
        - "WWW-Authenticate"
      "acceptedMethods":
        - "DELETE"
        - "GET"
        - "HEAD"
        - "PATCH"
        - "POST"
        - "PUT"
      "acceptedOrigins":
        - "http://{{ login_hostname }}:{{ login_port }}"
        - "http://{{ admin_hostname }}:{{ admin_port }}"
        - "http://{{ enduser_hostname }}:{{ enduser_port }}"
        - "http://{{ idm_hostname }}:{{ boot_port_http }}"
        - "https://{{ platform_hostname }}:{{ ig_https_port }}"
      "maxAge": 600
      "allowCredentials": true
      "enabled": true
    status_code: [200, 201]
```

### Configure Base URL Service

**Purpose**: Configure base URL for AM

```yaml
- name: Create Base URL Service (Root Realm)
  uri:
    url: "{{ am_url }}/json/realms/root/realm-config/services/baseurl?_action=create"
    method: POST
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=1.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "extensionClassName": ""
    status_code: [200, 201]

- name: Configure Base URL Service (Root Realm)
  uri:
    url: "{{ am_url }}/json/realms/root/realm-config/services/baseurl"
    method: PUT
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=1.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "extensionClassName": ""
      "fixedValue": "https://{{ platform_hostname }}:{{ ig_https_port }}"
      "contextPath": "/{{ am_context }}"
      "source": "FIXED_VALUE"
    status_code: [200, 201]
```

### Configure Cookie Domain

**Purpose**: Set AM cookie domain

```yaml
- name: Set AM Cookie Domain
  uri:
    url: "{{ am_url }}/json/global-config/services/platform"
    method: PUT
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=1.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "cookieDomains":
        - "{{ cookie_domain }}"
      "locale": "en_US"
    status_code: [200, 201]
```

### Map Self-Service Trees

**Purpose**: Map authentication trees for self-service

```yaml
- name: Map Self Service Trees
  uri:
    url: "{{ am_url }}/json/realms/root/realms/alpha/realm-config/services/selfServiceTrees?_action=create"
    method: POST
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=1.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "treeMapping":
        "registration": "PlatformRegistration"
        "login": "PlatformLogin"
        "resetPassword": "PlatformResetPassword"
    status_code: [200, 201]
```

### Configure OAuth2/OIDC Service

**Purpose**: Configure OAuth2/OIDC service

```yaml
- name: Create OAuth/OIDC Service (Root Realm)
  uri:
    url: "{{ am_url }}/json/realms/root/realm-config/services/oauth-oidc?_action=create"
    method: POST
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=1.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "advancedOAuth2Config":
        "passwordGrantAuthService": "[Empty]"
        "persistentClaims": []
        "supportedScopes":
          - "am-introspect-all-tokens"
          - "am-introspect-all-tokens-any-realm"
          - "fr:idm:*"
          - "openid"
      "clientsCanSkipConsent": true
    status_code: [200, 201]

- name: Create OAuth/OIDC Service (Alpha Realm)
  uri:
    url: "{{ am_url }}/json/realms/root/realms/alpha/realm-config/services/oauth-oidc?_action=create"
    method: POST
    headers:
      Accept: "application/json"
      Content-Type: "application/json"
      Accept-API-Version: "protocol=1.0,resource=1.0"
      Cookie: "iPlanetDirectoryPro={{ am_admin_token }}"
    body_format: json
    body:
      "advancedOAuth2Config":
        "passwordGrantAuthService": "[Empty]"
        "persistentClaims": []
        "supportedScopes":
          - "fr:idm:*"
          - "openid"
      "clientsCanSkipConsent": true
    status_code: [200, 201]
```

## AD Integration

### AD Connector Configuration in IDM

**Purpose**: Configure AD connector in IDM for data synchronization

**File**: `playbooks/ad-integration.yml`

```yaml
---
- name: Configure AD Connector in IDM
  hosts: idm
  gather_facts: yes
  tasks:
    - name: Configure AD connector
      uri:
        url: "http://{{ idm_hostname }}:{{ boot_port_http }}/openidm/config/provisioner.openicf/ad"
        method: PUT
        headers:
          X-OpenIDM-Username: "openidm-admin"
          X-OpenIDM-Password: "{{ vault('secret/ping/platform8/idm/admin_password') }}"
          Content-Type: "application/json"
        body_format: json
        body:
          "connectorRef":
            "connectorHostRef": "org.identityconnectors.ldap.LdapConnector"
            "bundleVersion": "1.4.74.0"
          "configurationProperties":
            "host": "{{ ad_server }}"
            "port": {{ ad_port }}
            "ssl": "{{ ad_ssl_enabled | default(false) }}"
            "principal": "{{ vault('secret/ping/platform8/ad/bind_dn') }}"
            "credentials": "{{ vault('secret/ping/platform8/ad/bind_password') }}"
            "baseContexts":
              - "{{ vault('secret/ping/platform8/ad/base_dn') }}"
            "createBaseContext": false
            "readSchema": true
        status_code: [200, 201]
      when: ad_integration_enabled | default(false)
      no_log: true

    - name: Configure AD synchronization mapping
      uri:
        url: "http://{{ idm_hostname }}:{{ boot_port_http }}/openidm/config/sync/ad"
        method: PUT
        headers:
          X-OpenIDM-Username: "openidm-admin"
          X-OpenIDM-Password: "{{ vault('secret/ping/platform8/idm/admin_password') }}"
          Content-Type: "application/json"
        body_format: json
        body:
          "enabled": true
          "source": "system/ad/__ACCOUNT__"
          "target": "system/ds/__ACCOUNT__"
          "correlationQuery": {
            "_queryFilter": "/userName eq \"${source.sAMAccountName}\""
          }
          "properties": [
            {
              "source": "sAMAccountName"
              "target": "uid"
            },
            {
              "source": "givenName"
              "target": "givenName"
            },
            {
              "source": "sn"
              "target": "sn"
            },
            {
              "source": "mail"
              "target": "mail"
            }
          ]
        status_code: [200, 201]
      when: ad_integration_enabled | default(false)
      no_log: true

    - name: Configure reconciliation job
      uri:
        url: "http://{{ idm_hostname }}:{{ boot_port_http }}/openidm/scheduler/job/ad-reconciliation"
        method: PUT
        headers:
          X-OpenIDM-Username: "openidm-admin"
          X-OpenIDM-Password: "{{ vault('secret/ping/platform8/idm/admin_password') }}"
          Content-Type: "application/json"
        body_format: json
        body:
          "enabled": true
          "type": "cron"
          "schedule": "0 0 2 * * ?"  # Daily at 2 AM
          "persisted": true
          "misfirePolicy": "fireAndProceed"
          "invokeService": "sync"
          "invokeContext":
            "action": "reconcile"
            "mapping": "systemAdAccount_systemDsAccount"
        status_code: [200, 201]
      when: ad_integration_enabled | default(false)
      no_log: true
```

## Data Flow Configuration

### AD → IDM → DS Data Flow

**Purpose**: Configure data flow from AD through IDM to DS

**Steps**:

1. **AD Connector**: Configured in IDM (see above)
2. **IDM → DS Provisioning**: IDM provisions users to DS IDRepo
3. **AM → DS Reading**: AM reads from same DS IDRepo

**Configuration**:

The data flow is configured through:
- AD connector in IDM (provisioner.openicf/ad)
- IDM synchronization mapping (sync/ad)
- IDM repository configuration (repo.ds.json) - points to DS IDRepo
- AM identity repository configuration - points to same DS IDRepo

## Integration Verification

### Verify AM → DS Connectivity

```yaml
- name: Verify AM to DS Config Store connectivity
  uri:
    url: "{{ am_url }}/json/serverinfo/*"
    method: GET
    status_code: 200
  register: am_status

- name: Test AM can access DS
  assert:
    that:
      - am_status.status == 200
    fail_msg: "AM cannot access DS"
    success_msg: "AM to DS connectivity verified"
```

### Verify IDM → DS Connectivity

```yaml
- name: Verify IDM to DS IDRepo connectivity
  uri:
    url: "http://{{ idm_hostname }}:{{ boot_port_http }}/openidm/repo/ds/account?_queryFilter=true&_pageSize=1"
    method: GET
    headers:
      X-OpenIDM-Username: "openidm-admin"
      X-OpenIDM-Password: "{{ vault('secret/ping/platform8/idm/admin_password') }}"
    status_code: 200
  register: idm_ds_test
  no_log: true

- name: Verify IDM can access DS
  assert:
    that:
      - idm_ds_test.status == 200
    fail_msg: "IDM cannot access DS"
    success_msg: "IDM to DS connectivity verified"
```

### Verify AM → IDM OAuth2 Flow

```yaml
- name: Test AM to IDM OAuth2 flow
  uri:
    url: "{{ am_url }}/oauth2/access_token"
    method: POST
    headers:
      Content-Type: "application/x-www-form-urlencoded"
    body_format: form-urlencoded
    body:
      grant_type: "client_credentials"
      client_id: "idm-provisioning"
      client_secret: "openidm"
      scope: "fr:idm:*"
    status_code: 200
  register: oauth2_test
  no_log: true

- name: Verify OAuth2 flow works
  assert:
    that:
      - oauth2_test.status == 200
      - oauth2_test.json.access_token is defined
    fail_msg: "AM to IDM OAuth2 flow failed"
    success_msg: "AM to IDM OAuth2 flow verified"
```

### Verify AD → IDM → DS Data Flow

```yaml
- name: Trigger AD reconciliation
  uri:
    url: "http://{{ idm_hostname }}:{{ boot_port_http }}/openidm/recon?_action=recon"
    method: POST
    headers:
      X-OpenIDM-Username: "openidm-admin"
      X-OpenIDM-Password: "{{ vault('secret/ping/platform8/idm/admin_password') }}"
      Content-Type: "application/json"
    body_format: json
    body:
      "mapping": "systemAdAccount_systemDsAccount"
    status_code: 200
  register: reconciliation_test
  no_log: true
  when: ad_integration_enabled | default(false)

- name: Wait for reconciliation to complete
  pause:
    seconds: 30
  when: ad_integration_enabled | default(false)

- name: Verify user was provisioned to DS
  uri:
    url: "http://{{ idm_hostname }}:{{ boot_port_http }}/openidm/repo/ds/account?_queryFilter=/uid eq \"testuser\""
    method: GET
    headers:
      X-OpenIDM-Username: "openidm-admin"
      X-OpenIDM-Password: "{{ vault('secret/ping/platform8/idm/admin_password') }}"
    status_code: 200
  register: user_check
  no_log: true
  when: ad_integration_enabled | default(false)

- name: Verify AD data flow
  assert:
    that:
      - user_check.json.result | length > 0
    fail_msg: "AD to IDM to DS data flow not working"
    success_msg: "AD to IDM to DS data flow verified"
  when: ad_integration_enabled | default(false)
```

## Post-Deployment Playbook

**File**: `playbooks/post-deploy.yml`

```yaml
---
- name: Post-deployment configuration
  hosts: am
  gather_facts: yes
  vars:
    am_admin_token: ""

  tasks:
    - name: Configure AM via REST API
      include_role:
        name: am
        tasks_from: rest_config.yml
      vars:
        am_admin_token: "{{ am_admin_token }}"

- name: Configure IDM integration
  hosts: am
  gather_facts: yes
  tasks:
    - name: Configure IDM integration service
      include_role:
        name: am
        tasks_from: idm_integration.yml

- name: Verify integrations
  hosts: all
  gather_facts: yes
  tasks:
    - name: Include verification tasks
      include_role:
        name: validation
        tasks_from: integration_verification.yml
```

## Execution

### Run Post-Deployment Configuration

```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  playbooks/post-deploy.yml
```

### Run AD Integration

```bash
ansible-playbook -i inventory/{env}/hosts.yml \
  -e "vault_role_id=<ROLE_ID>" \
  -e "vault_secret_id=<SECRET_ID>" \
  -e "ad_integration_enabled=true" \
  -e "ad_server=ad.example.com" \
  -e "ad_port=389" \
  playbooks/ad-integration.yml
```

## Next Steps

- Refer to **08-reference-guide.md** for troubleshooting and best practices
- Test all integrations after deployment
- Monitor logs for any issues

