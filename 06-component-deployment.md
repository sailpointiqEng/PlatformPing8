# Component Deployment Procedures

## Overview

This document provides detailed deployment procedures for each Ping Identity Platform 8 component, including idempotent logic, detection mechanisms, installation steps, and update procedures.

## Idempotent Deployment Logic

### Detection Strategy

All components use detection logic to determine if they are already installed. This enables:
- **Fresh Installation**: If component not detected
- **Incremental Update**: If component detected (preserves existing configs)

### Component Detection

#### DS Detection

**File**: `roles/ds/tasks/detect.yml`

```yaml
---
- name: Check if DS instance exists
  command: "{{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/status"
  register: ds_status_check
  changed_when: false
  failed_when: false
  when: ds_instance is defined

- name: Set DS installation status
  set_fact:
    ds_installed: "{{ ds_status_check.rc == 0 }}"
    ds_version: "{{ ds_status_check.stdout | regex_search('version: (.*)') | default('unknown') }}"
  when: ds_instance is defined
```

#### AM Detection

**File**: `roles/am/tasks/detect.yml`

```yaml
---
- name: Check if AM WAR is deployed
  stat:
    path: "{{ tomcat_webapps_dir }}/{{ am_context }}.war"
  register: am_war_file

- name: Check if AM is configured
  stat:
    path: "{{ am_cfg_dir }}"
  register: am_config_dir

- name: Check if AM directory exists
  stat:
    path: "{{ am_dir }}"
  register: am_dir_check

- name: Set AM installation status
  set_fact:
    am_installed: "{{ am_war_file.stat.exists and am_config_dir.stat.exists and am_dir_check.stat.exists }}"
```

#### IDM Detection

**File**: `roles/idm/tasks/detect.yml`

```yaml
---
- name: Check if IDM is installed
  stat:
    path: "{{ idm_extract_dir }}/startup.sh"
  register: idm_startup

- name: Set IDM installation status
  set_fact:
    idm_installed: "{{ idm_startup.stat.exists }}"
```

## Directory Services (DS) Deployment

### DS Main Orchestration

**File**: `roles/ds/tasks/main.yml`

```yaml
---
- name: Include detection tasks
  include_tasks: detect.yml

- name: Include installation tasks
  include_tasks: install.yml
  when: not ds_installed | default(false)

- name: Include update tasks
  include_tasks: update.yml
  when: ds_installed | default(false)

- name: Include replication tasks
  include_tasks: replication.yml
  when: 
    - ds_installed | default(false)
    - ds_replication_enabled | default(true)
    - ds_auto_replication | default(true)

- name: Include verification tasks
  include_tasks: verify.yml
```

### DS Fresh Installation

**File**: `roles/ds/tasks/install.yml`

```yaml
---
- name: Stop any existing DS processes
  shell: |
    pids=$(ps -ef | grep 'org.opends.server.core.DirectoryServer' | grep -v grep | awk '{print $2}') || true
    if [ -n "$pids" ]; then
      kill -9 $pids || true
    fi
  changed_when: false
  failed_when: false

- name: Create DS instance directory
  file:
    path: "{{ ds_install_dir }}/{{ ds_instance }}"
    state: directory
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    mode: '0755'

- name: Extract DS software
  unarchive:
    src: "{{ ds_zip_file }}"
    dest: "{{ ds_install_dir }}/{{ ds_instance }}/tmp"
    remote_src: false
    owner: "{{ install_user }}"
    group: "{{ install_user }}"

- name: Include instance-specific installation
  include_tasks: "{{ ds_instance }}.yml"
  when: ds_instance in ['config', 'cts', 'idrepo']
```

### DS Config Store Installation

**File**: `roles/ds/tasks/config_store.yml`

```yaml
---
- name: Install DS Config Store
  command: >
    {{ ds_install_dir }}/{{ ds_instance }}/tmp/opendj/setup
    --cli
    --rootUserDN "{{ ds_admin_dn }}"
    --rootUserPassword {{ vault('secret/ping/platform8/ds/admin_password') }}
    --monitorUserPassword {{ vault('secret/ping/platform8/ds/monitor_password') }}
    --hostname {{ ds_amconfig_server }}
    --ldapPort {{ ds_amconfig_server_ldap_port }}
    --ldapsPort {{ ds_amconfig_server_ldaps_port }}
    --httpPort {{ ds_amconfig_server_http_port }}
    --httpsPort {{ ds_amconfig_server_https_port }}
    --adminConnectorPort {{ ds_amconfig_server_admin_connector_port }}
    --deploymentId {{ vault('secret/ping/platform8/ds/deployment_id') }}
    --deploymentIdPassword {{ vault('secret/ping/platform8/ds/deployment_id_password') }}
    --profile am-config
    --set am-config/amConfigAdminPassword:{{ vault('secret/ping/platform8/am/admin_password') }}
    --start
    --quiet
    --no-prompt
    --acceptLicense
  args:
    chdir: "{{ ds_install_dir }}/{{ ds_instance }}/tmp/opendj"
  become_user: "{{ install_user }}"
  when: not ds_installed | default(false)

- name: Export DS Config Store CA certificate
  command: >
    {{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/dskeymgr export-ca-cert
    --deploymentId {{ vault('secret/ping/platform8/ds/deployment_id') }}
    --deploymentIdPassword {{ vault('secret/ping/platform8/ds/deployment_id_password') }}
    --outputFile {{ am_truststore_folder }}/ds-config-ca-cert.pem
  become_user: "{{ install_user }}"
  when: not ds_installed | default(false)
```

### DS CTS Installation

**File**: `roles/ds/tasks/cts.yml`

```yaml
---
- name: Install DS CTS Store
  command: >
    {{ ds_install_dir }}/{{ ds_instance }}/tmp/opendj/setup
    --cli
    --rootUserDN "{{ ds_admin_dn }}"
    --rootUserPassword {{ vault('secret/ping/platform8/ds/admin_password') }}
    --monitorUserPassword {{ vault('secret/ping/platform8/ds/monitor_password') }}
    --hostname {{ ds_cts_server }}
    --ldapPort {{ ds_cts_server_ldap_port }}
    --ldapsPort {{ ds_cts_server_ldaps_port }}
    --httpPort {{ ds_cts_server_http_port }}
    --httpsPort {{ ds_cts_server_https_port }}
    --adminConnectorPort {{ ds_cts_server_admin_connector_port }}
    --deploymentId {{ vault('secret/ping/platform8/ds/deployment_id') }}
    --deploymentIdPassword {{ vault('secret/ping/platform8/ds/deployment_id_password') }}
    --profile am-cts
    --set am-cts/amCtsAdminPassword:{{ vault('secret/ping/platform8/am/admin_password') }}
    --set am-cts/tokenExpirationPolicy:am-sessions-only
    --start
    --quiet
    --no-prompt
    --acceptLicense
  args:
    chdir: "{{ ds_install_dir }}/{{ ds_instance }}/tmp/opendj"
  become_user: "{{ install_user }}"
  when: not ds_installed | default(false)

- name: Export DS CTS CA certificate
  command: >
    {{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/dskeymgr export-ca-cert
    --deploymentId {{ vault('secret/ping/platform8/ds/deployment_id') }}
    --deploymentIdPassword {{ vault('secret/ping/platform8/ds/deployment_id_password') }}
    --outputFile {{ am_truststore_folder }}/ds-cts-ca-cert.pem
  become_user: "{{ install_user }}"
  when: not ds_installed | default(false)
```

### DS IDRepo Installation

**File**: `roles/ds/tasks/idrepo.yml`

```yaml
---
- name: Install DS IDRepo Store
  command: >
    {{ ds_install_dir }}/{{ ds_instance }}/tmp/opendj/setup
    --cli
    --rootUserDN "{{ ds_admin_dn }}"
    --rootUserPassword {{ vault('secret/ping/platform8/ds/admin_password') }}
    --monitorUserPassword {{ vault('secret/ping/platform8/ds/monitor_password') }}
    --hostname {{ ds_idrepo_server }}
    --ldapPort {{ ds_idrepo_server_ldap_port }}
    --ldapsPort {{ ds_idrepo_server_ldaps_port }}
    --httpPort {{ ds_idrepo_server_http_port }}
    --httpsPort {{ ds_idrepo_server_https_port }}
    --adminConnectorPort {{ ds_idrepo_server_admin_connector_port }}
    --enableStartTLS
    --deploymentId {{ vault('secret/ping/platform8/ds/deployment_id') }}
    --deploymentIdPassword {{ vault('secret/ping/platform8/ds/deployment_id_password') }}
    --profile am-identity-store:8.0.0
    --set am-identity-store/amIdentityStoreAdminPassword:{{ vault('secret/ping/platform8/am/admin_password') }}
    --profile idm-repo
    --set idm-repo/domain:{{ ds_repo_suffix }}
    --start
    --quiet
    --no-prompt
    --acceptLicense
  args:
    chdir: "{{ ds_install_dir }}/{{ ds_instance }}/tmp/opendj"
  become_user: "{{ install_user }}"
  when: not ds_installed | default(false)

- name: Export DS IDRepo CA certificate
  command: >
    {{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/dskeymgr export-ca-cert
    --deploymentId {{ vault('secret/ping/platform8/ds/deployment_id') }}
    --deploymentIdPassword {{ vault('secret/ping/platform8/ds/deployment_id_password') }}
    --outputFile {{ am_truststore_folder }}/ds-repo-ca-cert.pem
  become_user: "{{ install_user }}"
  when: not ds_installed | default(false)
```

### DS Update Procedure

**File**: `roles/ds/tasks/update.yml`

```yaml
---
- name: Backup existing DS configuration
  archive:
    path:
      - "{{ ds_install_dir }}/{{ ds_instance }}/config"
      - "{{ ds_install_dir }}/{{ ds_instance }}/db"
    dest: "/tmp/ds-{{ ds_instance }}-backup-{{ ansible_date_time.epoch }}.tar.gz"
  when: ds_installed | default(false)

- name: Check DS version compatibility
  command: "{{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/status"
  register: current_version
  changed_when: false
  when: ds_installed | default(false)

- name: Stop DS instance for update
  command: "{{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/stop-ds"
  become_user: "{{ install_user }}"
  when: 
    - ds_installed | default(false)
    - current_version.stdout is not search(ds_version)

- name: Update DS software only (preserve data)
  unarchive:
    src: "{{ ds_zip_file }}"
    dest: "{{ ds_install_dir }}/{{ ds_instance }}/tmp"
    remote_src: false
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
  when: 
    - ds_installed | default(false)
    - current_version.stdout is not search(ds_version)

- name: Run DS upgrade
  command: >
    {{ ds_install_dir }}/{{ ds_instance }}/tmp/opendj/bin/upgrade
    --acceptLicense
    --no-prompt
  args:
    chdir: "{{ ds_install_dir }}/{{ ds_instance }}/tmp/opendj"
  become_user: "{{ install_user }}"
  when: 
    - ds_installed | default(false)
    - current_version.stdout is not search(ds_version)

- name: Start DS instance
  command: "{{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/start-ds"
  become_user: "{{ install_user }}"
  when: 
    - ds_installed | default(false)
    - current_version.stdout is not search(ds_version)
```

### DS Replication Setup

**File**: `roles/ds/tasks/replication.yml`

```yaml
---
- name: Get replica information
  set_fact:
    ds_replica1_hostname: "{{ groups['ds'] | selectattr('ds_instance', 'equalto', ds_instance) | selectattr('ds_replica_id', 'equalto', 1) | map(attribute='ansible_host') | first }}"
    ds_replica2_hostname: "{{ groups['ds'] | selectattr('ds_instance', 'equalto', ds_instance) | selectattr('ds_replica_id', 'equalto', 2) | map(attribute='ansible_host') | first }}"
  when: ds_replication_enabled | default(false)

- name: Enable replication between DS instances
  command: >
    {{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/dsreplication enable
    --host1 {{ ds_replica1_hostname }}
    --port1 {{ ds_idrepo_server_ldap_port if ds_instance == 'idrepo' else (ds_cts_server_ldap_port if ds_instance == 'cts' else ds_amconfig_server_ldap_port) }}
    --bindDN1 "{{ ds_admin_dn }}"
    --bindPassword1 {{ vault('secret/ping/platform8/ds/admin_password') }}
    --replicationPort1 {{ ds_replica1_replication_port | default(8989) }}
    --host2 {{ ds_replica2_hostname }}
    --port2 {{ ds_idrepo_server_ldap_port if ds_instance == 'idrepo' else (ds_cts_server_ldap_port if ds_instance == 'cts' else ds_amconfig_server_ldap_port) }}
    --bindDN2 "{{ ds_admin_dn }}"
    --bindPassword2 {{ vault('secret/ping/platform8/ds/admin_password') }}
    --replicationPort2 {{ ds_replica2_replication_port | default(9989) }}
    --baseDN {{ ds_idrepo_dn if ds_instance == 'idrepo' else 'ou=am-config' if ds_instance == 'config' else 'ou=famrecords,ou=openam-session,ou=tokens' }}
    --adminUID admin
    --adminPassword {{ vault('secret/ping/platform8/ds/admin_password') }}
    --no-prompt
  become_user: "{{ install_user }}"
  when:
    - ds_replication_enabled | default(false)
    - ds_auto_replication | default(true)
    - ds_replica_id == 1

- name: Initialize replication
  command: >
    {{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/dsreplication initialize
    --baseDN {{ ds_idrepo_dn if ds_instance == 'idrepo' else 'ou=am-config' if ds_instance == 'config' else 'ou=famrecords,ou=openam-session,ou=tokens' }}
    --hostname {{ ds_replica1_hostname }}
    --port {{ ds_idrepo_server_ldap_port if ds_instance == 'idrepo' else (ds_cts_server_ldap_port if ds_instance == 'cts' else ds_amconfig_server_ldap_port) }}
    --bindDN "{{ ds_admin_dn }}"
    --bindPassword {{ vault('secret/ping/platform8/ds/admin_password') }}
    --no-prompt
  become_user: "{{ install_user }}"
  when:
    - ds_replication_enabled | default(false)
    - ds_auto_replication | default(true)
    - ds_replica_id == 1
```

## Access Management (AM) Deployment

### AM Main Orchestration

**File**: `roles/am/tasks/main.yml`

```yaml
---
- name: Include detection tasks
  include_tasks: detect.yml

- name: Include installation tasks
  include_tasks: install.yml
  when: not am_installed | default(false)

- name: Include update tasks
  include_tasks: update.yml
  when: am_installed | default(false)

- name: Include verification tasks
  include_tasks: verify.yml
```

### AM Fresh Installation

**File**: `roles/am/tasks/install.yml`

```yaml
---
- name: Stop Tomcat if running
  systemd:
    name: tomcat
    state: stopped
  when: tomcat_running | default(false)

- name: Deploy AM setenv.sh
  template:
    src: setenv.j2
    dest: "{{ tomcat_bin_dir }}/setenv.sh"
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    mode: '0755'

- name: Start Tomcat
  systemd:
    name: tomcat
    state: started
    enabled: yes

- name: Wait for Tomcat to start
  wait_for:
    port: "{{ tomcat_http_port }}"
    delay: 10
    timeout: 300

- name: Deploy AM WAR to Tomcat
  copy:
    src: "{{ am_war }}"
    dest: "{{ tomcat_webapps_dir }}/{{ am_context }}.war"
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    mode: '0644'
  notify: restart tomcat

- name: Wait for AM to deploy
  wait_for:
    port: "{{ tomcat_http_port }}"
    delay: 20
    timeout: 600

- name: Setup Amster
  include_tasks: amster.yml

- name: Run Amster installation
  include_tasks: amster_install.yml
```

### AM Amster Setup

**File**: `roles/am/tasks/amster.yml`

```yaml
---
- name: Extract Amster
  unarchive:
    src: "{{ amster_zip }}"
    dest: "{{ software_dir }}/am"
    remote_src: false
    owner: "{{ install_user }}"
    group: "{{ install_user }}"

- name: Set Amster directory
  set_fact:
    amster_dir: "{{ software_dir }}/am/amster"
```

### AM Amster Installation

**File**: `roles/am/tasks/amster_install.yml`

```yaml
---
- name: Run Amster installation
  shell: |
    {{ amster_dir }}/amster <<EOF
    install-openam \
      --serverUrl {{ am_url }} \
      --adminPwd {{ vault('secret/ping/platform8/am/admin_password') }} \
      --acceptLicense \
      --pwdEncKey {{ vault('secret/ping/platform8/ds/deployment_id') }} \
      --cfgStoreDirMgr 'uid=am-config,ou=admins,ou=am-config' \
      --cfgStoreDirMgrPwd {{ vault('secret/ping/platform8/am/admin_password') }} \
      --cfgStore dirServer \
      --cfgStoreHost {{ ds_amconfig_server }} \
      --cfgStoreAdminPort {{ ds_amconfig_server_admin_connector_port }} \
      --cfgStorePort {{ ds_amconfig_server_ldaps_port }} \
      --cfgStoreRootSuffix ou=am-config \
      --cfgStoreSsl SSL \
      --userStoreDirMgr 'uid=am-identity-bind-account,ou=admins,{{ ds_idrepo_dn }}' \
      --userStoreDirMgrPwd {{ vault('secret/ping/platform8/am/admin_password') }} \
      --userStoreHost {{ ds_idrepo_server }} \
      --userStoreType LDAPv3ForOpenDS \
      --userStorePort {{ ds_idrepo_server_ldaps_port }} \
      --userStoreSsl SSL \
      --userStoreRootSuffix {{ ds_idrepo_dn }}
    :exit
    EOF
  become_user: "{{ install_user }}"
  environment:
    JAVA_HOME: "{{ java_home }}"
  when: not am_installed | default(false)

- name: Configure CTS via Amster
  shell: |
    {{ amster_dir }}/amster <<EOF
    connect -k {{ am_dir }}/security/keys/amster/amster_rsa {{ am_url }}
    update DefaultCtsDataStoreProperties --global --body '{"amconfig.org.forgerock.services.cts.store.common.section":{"org.forgerock.services.cts.store.location":"external","org.forgerock.services.cts.store.root.suffix":"ou=famrecords,ou=openam-session,ou=tokens","org.forgerock.services.cts.store.max.connections":"65","org.forgerock.services.cts.store.page.size":"0","org.forgerock.services.cts.store.vlv.page.size":"1000"},"amconfig.org.forgerock.services.cts.store.external.section":{"org.forgerock.services.cts.store.password":"{{ vault('secret/ping/platform8/am/admin_password') }}","org.forgerock.services.cts.store.loginid":"uid=openam_cts,ou=admins,ou=famrecords,ou=openam-session,ou=tokens","org.forgerock.services.cts.store.heartbeat":"10","org.forgerock.services.cts.store.ssl.enabled":"true","org.forgerock.services.cts.store.directory.name":"{{ ds_cts_server }}:{{ ds_cts_server_ldaps_port }}","org.forgerock.services.cts.store.affinity.enabled":true}}'
    :exit
    EOF
  become_user: "{{ install_user }}"
  environment:
    JAVA_HOME: "{{ java_home }}"
  when: not am_installed | default(false)

- name: Create Alpha realm
  shell: |
    {{ amster_dir }}/amster <<EOF
    connect -k {{ am_dir }}/security/keys/amster/amster_rsa {{ am_url }}
    create Realms --global --body '{"_id": "L2FscGhh", "parentPath": "/", "active": true, "name": "alpha", "aliases": []}'
    :exit
    EOF
  become_user: "{{ install_user }}"
  environment:
    JAVA_HOME: "{{ java_home }}"
  when: not am_installed | default(false)

- name: Import authentication journeys
  shell: |
    {{ amster_dir }}/amster <<EOF
    connect -k {{ am_dir }}/security/keys/amster/amster_rsa {{ am_url }}
    import-config --path {{ am_journeys_dir }}
    :exit
    EOF
  become_user: "{{ install_user }}"
  environment:
    JAVA_HOME: "{{ java_home }}"
  when: not am_installed | default(false)
```

### AM Update Procedure

**File**: `roles/am/tasks/update.yml`

```yaml
---
- name: Backup existing AM configuration
  archive:
    path:
      - "{{ am_cfg_dir }}"
      - "{{ am_dir }}"
    dest: "/tmp/am-backup-{{ ansible_date_time.epoch }}.tar.gz"
  when: am_installed | default(false)

- name: Stop Tomcat
  systemd:
    name: tomcat
    state: stopped
  when: am_installed | default(false)

- name: Deploy new AM WAR (preserve configs)
  copy:
    src: "{{ am_war }}"
    dest: "{{ tomcat_webapps_dir }}/{{ am_context }}.war"
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    mode: '0644'
  when: am_installed | default(false)

- name: Start Tomcat
  systemd:
    name: tomcat
    state: started
  when: am_installed | default(false)

- name: Wait for AM to redeploy
  wait_for:
    port: "{{ tomcat_http_port }}"
    delay: 20
    timeout: 600
  when: am_installed | default(false)
```

## Identity Management (IDM) Deployment

### IDM Main Orchestration

**File**: `roles/idm/tasks/main.yml`

```yaml
---
- name: Include detection tasks
  include_tasks: detect.yml

- name: Include installation tasks
  include_tasks: install.yml
  when: not idm_installed | default(false)

- name: Include update tasks
  include_tasks: update.yml
  when: idm_installed | default(false)

- name: Include configuration tasks
  include_tasks: config_files.yml

- name: Include verification tasks
  include_tasks: verify.yml
```

### IDM Fresh Installation

**File**: `roles/idm/tasks/install.yml`

```yaml
---
- name: Stop IDM if running
  shell: |
    pid=$(pgrep -f 'openidm' || true)
    if [ -n "$pid" ]; then
      if [ -x "{{ idm_extract_dir }}/shutdown.sh" ]; then
        {{ idm_extract_dir }}/shutdown.sh || kill -9 $pid || true
      else
        kill -9 $pid || true
      fi
    fi
  changed_when: false
  failed_when: false

- name: Extract IDM
  unarchive:
    src: "{{ idm_zip }}"
    dest: "{{ idm_dir }}"
    remote_src: false
    owner: "{{ install_user }}"
    group: "{{ install_user }}"

- name: Remove default repo.ds.json
  file:
    path: "{{ idm_config_dir }}/repo.ds.json"
    state: absent
  when: idm_config_dir is defined

- name: Import DS certificate into IDM truststore
  command: >
    keytool -importcert -noprompt
    -alias ds-repo-ca-cert
    -keystore {{ idm_truststore }}
    -storepass:file {{ idm_storepass_file }}
    -file {{ idm_cert_file }}
  become_user: "{{ install_user }}"
  when: idm_cert_file is defined and idm_cert_file | exists
```

### IDM Configuration Files

**File**: `roles/idm/tasks/config_files.yml`

```yaml
---
- name: Deploy IDM configuration files
  template:
    src: "{{ item }}.j2"
    dest: "{{ idm_config_dir }}/{{ item }}"
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    mode: '0644'
  loop:
    - repo.ds.json
    - managed.json
    - authentication.json
    - access.json
    - system.properties
    - servletfilter-cors.json
    - ui-configuration.json
    - ui-themerealm.json
    - metrics.json
  notify: restart idm

- name: Deploy boot.properties
  template:
    src: boot.properties.j2
    dest: "{{ idm_extract_dir }}/resolver/boot.properties"
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    mode: '0644'
  notify: restart idm
```

### IDM Update Procedure

**File**: `roles/idm/tasks/update.yml`

```yaml
---
- name: Backup existing IDM configuration
  archive:
    path:
      - "{{ idm_config_dir }}"
      - "{{ idm_extract_dir }}/db"
      - "{{ idm_extract_dir }}/script"
    dest: "/tmp/idm-backup-{{ ansible_date_time.epoch }}.tar.gz"
  when: idm_installed | default(false)

- name: Stop IDM
  shell: |
    pid=$(pgrep -f 'openidm' || true)
    if [ -n "$pid" ]; then
      if [ -x "{{ idm_extract_dir }}/shutdown.sh" ]; then
        {{ idm_extract_dir }}/shutdown.sh || kill -9 $pid || true
      else
        kill -9 $pid || true
      fi
    fi
  changed_when: false
  when: idm_installed | default(false)

- name: Extract new IDM version (preserve configs and data)
  unarchive:
    src: "{{ idm_zip }}"
    dest: "{{ idm_dir }}"
    remote_src: false
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    extra_opts:
      - "--exclude=conf/*"
      - "--exclude=db/*"
      - "--exclude=script/*"
  when: idm_installed | default(false)

- name: Restore configuration files
  unarchive:
    src: "/tmp/idm-backup-{{ ansible_date_time.epoch }}.tar.gz"
    dest: "{{ idm_extract_dir }}"
    remote_src: true
    extra_opts:
      - "--include=conf/*"
      - "--include=db/*"
      - "--include=script/*"
  when: idm_installed | default(false)
```

## Identity Gateway (IG) Deployment

### IG Installation

**File**: `roles/ig/tasks/install.yml`

```yaml
---
- name: Extract IG
  unarchive:
    src: "{{ ig_zip }}"
    dest: "{{ base_install_dir }}"
    remote_src: false
    owner: "{{ install_user }}"
    group: "{{ install_user }}"

- name: Create IG config directory
  file:
    path: "{{ ig_cfg_dir }}"
    state: directory
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    mode: '0755'

- name: Deploy IG configuration
  template:
    src: config.json.j2
    dest: "{{ ig_config }}/config.json"
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    mode: '0644'

- name: Deploy IG routes
  template:
    src: "{{ item }}.j2"
    dest: "{{ ig_routes }}/{{ item }}"
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    mode: '0644'
  loop:
    - 01-am.json
    - 02-idm.json
    - 03-platform.json
    - 04-enduser.json
    - 05-login.json
```

## Platform UI Deployment

### UI Installation

**File**: `roles/ui/tasks/main.yml`

```yaml
---
- name: Extract Platform UI
  unarchive:
    src: "{{ ui_zip }}"
    dest: "{{ ui_software_dir }}/tmp"
    remote_src: false
    owner: "{{ install_user }}"
    group: "{{ install_user }}"

- name: Run variable replacement script
  shell: |
    cd {{ ui_software_dir }}/tmp/PlatformUI
    export AM_URL="https://{{ platform_hostname }}:{{ ig_https_port }}/{{ am_context }}"
    export AM_ADMIN_URL="https://{{ platform_hostname }}:{{ ig_https_port }}/{{ am_context }}/ui-admin"
    export IDM_REST_URL="https://{{ platform_hostname }}:{{ ig_https_port }}/openidm"
    export IDM_ADMIN_URL="https://{{ platform_hostname }}:{{ ig_https_port }}/admin"
    ./variable_replacement.sh \
      www/platform/js/*.js \
      www/enduser/js/*.js \
      www/login/js/*.js
  become_user: "{{ install_user }}"

- name: Deploy UI to Tomcat
  copy:
    src: "{{ ui_software_dir }}/tmp/PlatformUI/www/{{ item.src }}"
    dest: "{{ tomcat_webapps_dir }}/{{ item.dest }}"
    owner: "{{ install_user }}"
    group: "{{ install_user }}"
    mode: '0755'
  loop:
    - { src: "platform", dest: "{{ am_context }}/XUI/platform" }
    - { src: "enduser", dest: "{{ am_context }}/XUI/enduser" }
    - { src: "login", dest: "{{ am_context }}/XUI/login" }
  notify: restart tomcat
```

## Verification Tasks

### DS Verification

**File**: `roles/ds/tasks/verify.yml`

```yaml
---
- name: Verify DS instance is running
  command: "{{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/status"
  register: ds_status
  changed_when: false

- name: Test LDAP connectivity
  command: >
    {{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/ldapsearch
    -h {{ ansible_host }}
    -p {{ ds_instance_port }}
    -D "{{ ds_admin_dn }}"
    -w {{ vault('secret/ping/platform8/ds/admin_password') }}
    -b "{{ ds_base_dn }}"
    -Z
    --trustStorePath {{ am_truststore }}
    "objectclass=*" dn
  changed_when: false
  failed_when: false
  register: ldap_test

- name: Verify LDAP connectivity succeeded
  assert:
    that:
      - ldap_test.rc == 0
    fail_msg: "LDAP connectivity test failed"
    success_msg: "LDAP connectivity verified"
```

### AM Verification

**File**: `roles/am/tasks/verify.yml`

```yaml
---
- name: Verify AM is accessible
  uri:
    url: "{{ am_url }}/json/serverinfo/*"
    method: GET
    status_code: [200, 401]
  register: am_status

- name: Verify AM is running
  assert:
    that:
      - am_status.status in [200, 401]
    fail_msg: "AM is not accessible"
    success_msg: "AM is accessible"
```

### IDM Verification

**File**: `roles/idm/tasks/verify.yml`

```yaml
---
- name: Verify IDM is accessible
  uri:
    url: "http://{{ idm_hostname }}:{{ boot_port_http }}/openidm/info/ping"
    method: GET
    status_code: 200
  register: idm_status

- name: Verify IDM is running
  assert:
    that:
      - idm_status.status == 200
    fail_msg: "IDM is not accessible"
    success_msg: "IDM is accessible"
```

## Configuration Preservation Strategy

### Key Principles

1. **Never Overwrite**: Existing configs, CTS, identity store, and extensions are never overwritten
2. **Backup First**: Always backup before updates
3. **Incremental Changes**: Only apply necessary changes
4. **Version Tracking**: Track component versions to determine if update is needed

### Backup Locations

- **DS**: `/tmp/ds-{instance}-backup-{timestamp}.tar.gz`
- **AM**: `/tmp/am-backup-{timestamp}.tar.gz`
- **IDM**: `/tmp/idm-backup-{timestamp}.tar.gz`

## Next Steps

- Refer to **07-post-deployment.md** for post-deployment configuration
- Refer to **08-reference-guide.md** for troubleshooting and best practices

