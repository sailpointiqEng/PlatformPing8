# Environment Preparation and Validation

## Overview

This document outlines all environment preparation requirements and validation procedures that MUST be completed before deploying Ping Identity Platform 8 components. All validation checks must pass before any deployment can proceed.

## Environment Preparation Requirements

### 1. Java Version

**Requirement**: PingAM/PingIDM/PingDS 8 supports OpenJDK 17 and 21

**Validation**:
```bash
java -version
```

**Expected Output**:
```
openjdk version "21.0.x" ...
# OR
openjdk version "17.0.x" ...
```

**Ansible Task**:
```yaml
- name: Check Java version
  command: java -version
  register: java_version_check
  changed_when: false
  failed_when: false

- name: Verify Java version is 17 or 21
  assert:
    that:
      - java_version_check.rc == 0
      - "'openjdk version \"17' in java_version_check.stderr or 'openjdk version \"21' in java_version_check.stderr"
    fail_msg: "Java version check failed. OpenJDK 17 or 21 required. Found: {{ java_version_check.stderr }}"
    success_msg: "Java version verified: {{ java_version_check.stderr }}"
```

### 2. Set JAVA_HOME

**Requirement**: JAVA_HOME environment variable must be set correctly

**Validation**:
```bash
echo $JAVA_HOME
```

**Expected Output**: Valid Java installation path (e.g., `/usr/lib/jvm/java-21-openjdk`)

**Ansible Task**:
```yaml
- name: Check JAVA_HOME is set
  shell: echo $JAVA_HOME
  register: java_home_check
  changed_when: false
  failed_when: false

- name: Verify JAVA_HOME is configured
  assert:
    that:
      - java_home_check.stdout | length > 0
      - java_home_check.stdout != ""
    fail_msg: "JAVA_HOME is not set"
    success_msg: "JAVA_HOME is set to: {{ java_home_check.stdout }}"

- name: Set JAVA_HOME in /etc/environment (RedHat)
  lineinfile:
    path: /etc/environment
    regexp: '^JAVA_HOME='
    line: 'JAVA_HOME=/usr/lib/jvm/java-21-openjdk'
    state: present
  when: ansible_os_family == 'RedHat'

- name: Set JAVA_HOME in /etc/environment (Debian/Ubuntu)
  lineinfile:
    path: /etc/environment
    regexp: '^JAVA_HOME='
    line: 'JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64'
    state: present
  when: ansible_os_family == 'Debian'
```

### 3. Create Service Account

**Requirement**: Create dedicated service account for Ping Identity components

**Account Details**:
- **Username**: `pingIdentity`
- **Group**: `pingIdentity`
- **Home Directory**: `/home/pingIdentity` (or as per organization standards)
- **Permissions**: Required sudo access for installation tasks

**Ansible Task**:
```yaml
- name: Check if pingIdentity group exists
  group:
    name: pingIdentity
    state: present
    system: yes

- name: Check if pingIdentity user exists
  user:
    name: pingIdentity
    group: pingIdentity
    home: /home/pingIdentity
    shell: /bin/bash
    create_home: yes
    system: yes
    state: present

- name: Verify pingIdentity user exists
  getent:
    database: passwd
    key: "pingIdentity"
  register: pingidentity_user_check

- name: Verify pingIdentity user exists
  assert:
    that:
      - pingidentity_user_check.ansible_facts.getent_passwd['pingIdentity'] is defined
    fail_msg: "pingIdentity user does not exist"
    success_msg: "pingIdentity user exists"

- name: Configure sudo access for pingIdentity
  lineinfile:
    path: /etc/sudoers.d/pingidentity
    line: "pingIdentity ALL=(ALL) NOPASSWD: ALL"
    validate: 'visudo -cf %s'
    state: present
```

### 4. Create Installation Directories

**Requirement**: Create installation directories with correct ownership

**Directories**:
- `/opt/ping/am` - AM installation directory
- `/opt/ping/ds` - DS installation directory
- `/opt/ping/idm` - IDM installation directory

**Ownership**: `pingIdentity:pingIdentity`
**Permissions**: Read/write for pingIdentity user

**Ansible Task**:
```yaml
- name: Create installation directories
  file:
    path: "{{ item }}"
    state: directory
    owner: pingIdentity
    group: pingIdentity
    mode: '0755'
  loop:
    - /opt/ping/am
    - /opt/ping/ds
    - /opt/ping/idm

- name: Check installation directories exist
  stat:
    path: "{{ item }}"
  loop:
    - /opt/ping/am
    - /opt/ping/ds
    - /opt/ping/idm
  register: dir_check

- name: Verify installation directories exist with correct ownership
  assert:
    that:
      - item.stat.exists
      - item.stat.uid == pingidentity_user_check.ansible_facts.getent_passwd['pingIdentity'].uid
    fail_msg: "Directory {{ item.item }} does not exist or has incorrect ownership"
    success_msg: "Directory {{ item.item }} exists with correct ownership"
  loop: "{{ dir_check.results }}"
```

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

**Ansible Task**:
```yaml
- name: Check required ports are available (DS Config Store)
  wait_for:
    host: "{{ ansible_host }}"
    port: "{{ item }}"
    state: stopped
    timeout: 1
  loop:
    - "{{ ds_amconfig_server_ldap_port }}"
    - "{{ ds_amconfig_server_ldaps_port }}"
    - "{{ ds_amconfig_server_http_port }}"
    - "{{ ds_amconfig_server_https_port }}"
    - "{{ ds_amconfig_server_admin_connector_port }}"
  when: "'ds' in group_names and ds_instance == 'config'"
  failed_when: false
  register: ds_config_ports

- name: Verify DS Config Store ports are available
  assert:
    that:
      - ds_config_ports.results | selectattr('failed', 'equalto', false) | list | length == ds_config_ports.results | length
    fail_msg: "One or more DS Config Store ports are already in use"
    success_msg: "All DS Config Store ports are available"
  when: "'ds' in group_names and ds_instance == 'config'"

- name: Check required ports are available (AM)
  wait_for:
    host: "{{ ansible_host }}"
    port: "{{ item }}"
    state: stopped
    timeout: 1
  loop:
    - "{{ tomcat_http_port }}"
    - "{{ tomcat_https_port | default(8443) }}"
  when: "'am' in group_names"
  failed_when: false
  register: am_ports

- name: Verify AM ports are available
  assert:
    that:
      - am_ports.results | selectattr('failed', 'equalto', false) | list | length == am_ports.results | length
    fail_msg: "AM ports are already in use"
    success_msg: "All AM ports are available"
  when: "'am' in group_names"

- name: Check required ports are available (IDM)
  wait_for:
    host: "{{ ansible_host }}"
    port: "{{ item }}"
    state: stopped
    timeout: 1
  loop:
    - "{{ boot_port_http }}"
    - "{{ boot_port_https }}"
  when: "'idm' in group_names"
  failed_when: false
  register: idm_ports

- name: Verify IDM ports are available
  assert:
    that:
      - idm_ports.results | selectattr('failed', 'equalto', false) | list | length == idm_ports.results | length
    fail_msg: "One or more IDM ports are already in use"
    success_msg: "All IDM ports are available"
  when: "'idm' in group_names"
```

### 6. Security/Truststore

**Requirement**: Create dedicated truststore for AM to trust DS

**Steps**:
1. Create truststore directory
2. Copy Java cacerts as base
3. Import DS certificates
4. Configure Tomcat to use truststore via CATALINA_OPTS
5. Ensure pingIdentity user has read permissions

**Ansible Task**:
```yaml
- name: Create AM truststore directory
  file:
    path: "{{ am_truststore_folder }}"
    state: directory
    owner: pingIdentity
    group: pingIdentity
    mode: '0755'

- name: Copy Java cacerts as base truststore
  copy:
    src: "{{ java_cacerts_path }}"
    dest: "{{ am_truststore }}"
    owner: pingIdentity
    group: pingIdentity
    mode: '0644'
  vars:
    java_cacerts_path: "{{ java_home }}/lib/security/cacerts"

- name: Import DS certificates into AM truststore
  command: >
    keytool -importcert
    -file {{ am_truststore_folder }}/{{ item }}
    -keystore {{ am_truststore }}
    -storepass {{ truststore_password }}
    -alias {{ item | basename | regex_replace('\.pem$', '') }}
    -noprompt
  loop:
    - ds-repo-ca-cert.pem
    - ds-config-ca-cert.pem
    - ds-cts-ca-cert.pem
  when: item is file
```

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

**Ansible Task**:
```yaml
- name: Test connectivity to all VMs
  ping:
  register: ping_results
  delegate_to: "{{ item }}"
  loop: "{{ groups['all'] }}"
  failed_when: false

- name: Verify all VMs are reachable
  assert:
    that:
      - ping_results.results | selectattr('item', 'equalto', inventory_hostname) | map(attribute='ping') | first == 'pong'
    fail_msg: "VM {{ inventory_hostname }} is not reachable"
    success_msg: "All VMs are reachable"

- name: Test DNS resolution
  command: hostname -f
  register: hostname_fqdn
  changed_when: false

- name: Verify FQDN resolution
  assert:
    that:
      - hostname_fqdn.stdout is match('.*\..*')
    fail_msg: "FQDN resolution failed: {{ hostname_fqdn.stdout }}"
    success_msg: "FQDN resolution successful: {{ hostname_fqdn.stdout }}"

- name: Test reverse DNS lookup
  command: "getent hosts {{ ansible_default_ipv4.address }}"
  register: reverse_dns
  changed_when: false
  failed_when: false

- name: Verify reverse DNS works
  assert:
    that:
      - reverse_dns.rc == 0
    fail_msg: "Reverse DNS lookup failed"
    success_msg: "Reverse DNS lookup successful"
```

### NTP Synchronization

**Ansible Task**:
```yaml
- name: Check NTP synchronization
  command: ntpq -p
  register: ntp_check
  changed_when: false
  failed_when: false

- name: Verify NTP is synchronized
  assert:
    that:
      - ntp_check.rc == 0
    fail_msg: "NTP synchronization check failed"
    success_msg: "NTP is synchronized"
```

### Software Binary Validation

**Ansible Task**:
```yaml
- name: Check if DS software binary exists
  stat:
    path: "{{ ds_zip_file }}"
  register: ds_software_check
  when: "'ds' in group_names"

- name: Verify DS software binary exists
  assert:
    that:
      - ds_software_check.stat.exists
    fail_msg: "DS software binary not found: {{ ds_zip_file }}"
    success_msg: "DS software binary found: {{ ds_zip_file }}"
  when: "'ds' in group_names"

- name: Check if AM software binary exists
  stat:
    path: "{{ am_war }}"
  register: am_software_check
  when: "'am' in group_names"

- name: Verify AM software binary exists
  assert:
    that:
      - am_software_check.stat.exists
    fail_msg: "AM software binary not found: {{ am_war }}"
    success_msg: "AM software binary found: {{ am_war }}"
  when: "'am' in group_names"

- name: Check if IDM software binary exists
  stat:
    path: "{{ idm_zip }}"
  register: idm_software_check
  when: "'idm' in group_names"

- name: Verify IDM software binary exists
  assert:
    that:
      - idm_software_check.stat.exists
    fail_msg: "IDM software binary not found: {{ idm_zip }}"
    success_msg: "IDM software binary found: {{ idm_zip }}"
  when: "'idm' in group_names"
```

## Infrastructure VMs Setup

### Tomcat Infrastructure VMs

**Purpose**: Dedicated VMs for Tomcat installation

**Requirements**:
- Tomcat 10.1.46
- Owned by pingIdentity user
- Systemd service configured
- Ports configured

**Ansible Task**:
```yaml
- name: Download Tomcat
  get_url:
    url: "https://archive.apache.org/dist/tomcat/tomcat-10/v{{ tomcat_version }}/bin/apache-tomcat-{{ tomcat_version }}.tar.gz"
    dest: "/tmp/apache-tomcat-{{ tomcat_version }}.tar.gz"
    mode: '0644'

- name: Extract Tomcat
  unarchive:
    src: "/tmp/apache-tomcat-{{ tomcat_version }}.tar.gz"
    dest: "/opt"
    remote_src: true

- name: Create Tomcat symlink
  file:
    src: "/opt/apache-tomcat-{{ tomcat_version }}"
    dest: "{{ tomcat_dir }}"
    state: link

- name: Set Tomcat ownership
  file:
    path: "{{ tomcat_dir }}"
    owner: pingIdentity
    group: pingIdentity
    recurse: yes
```

### JDK Infrastructure VMs

**Purpose**: Dedicated VMs for JDK installation

**Requirements**:
- OpenJDK 21 (or 17)
- JAVA_HOME configured
- Available system-wide

**Ansible Task**:
```yaml
- name: Install OpenJDK 21
  package:
    name: "{{ 'java-21-openjdk' if ansible_os_family == 'RedHat' else 'openjdk-21-jdk' }}"
    state: present

- name: Set JAVA_HOME in /etc/environment
  lineinfile:
    path: /etc/environment
    regexp: '^JAVA_HOME='
    line: 'JAVA_HOME=/usr/lib/jvm/java-21-openjdk'
    state: present
  when: ansible_os_family == 'RedHat'

- name: Verify JDK installation
  command: java -version
  register: jdk_verify
  changed_when: false
```

## Validation Playbook

### Complete Validation Role

**File**: `roles/validation/tasks/main.yml`

```yaml
---
- name: Pre-deployment validation
  block:
    - name: Include connectivity checks
      include_tasks: connectivity.yml
    
    - name: Include port availability checks
      include_tasks: ports.yml
    
    - name: Include prerequisites validation
      include_tasks: prerequisites.yml
    
    - name: Include software binary checks
      include_tasks: software.yml
  always:
    - name: Display validation summary
      debug:
        msg: "Pre-deployment validation completed"
```

## Execution Order

1. **Environment Preparation** (run first)
   - Create service account
   - Create installation directories
   - Install Java (if not on infrastructure VM)
   - Configure JAVA_HOME
   - Create truststore

2. **Infrastructure Deployment** (run second)
   - Deploy Tomcat on infrastructure VMs
   - Deploy JDK on infrastructure VMs

3. **Pre-Deployment Validation** (run third, MUST PASS)
   - Connectivity checks
   - Port availability
   - Prerequisites validation
   - Software binary checks

4. **Component Deployment** (run only if validation passes)
   - Deploy DS
   - Deploy AM
   - Deploy IDM
   - Deploy IG (if needed)
   - Deploy UI (if needed)

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

