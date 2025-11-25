# Reference Guide

## Overview

This document provides troubleshooting guides, best practices, reference links, and maintenance procedures for the Ping Identity Platform 8 Ansible automation.

## Troubleshooting

### Common Issues

#### 1. Pre-Deployment Validation Failures

**Issue**: Validation fails with connectivity errors

**Symptoms**:
- "VM is not reachable"
- "Port is already in use"
- "Java version check failed"

**Solutions**:
```bash
# Test connectivity manually
ansible all -i inventory/{env}/hosts.yml -m ping

# Check port availability
ansible all -i inventory/{env}/hosts.yml -m shell -a "netstat -tuln | grep :8081"

# Verify Java version
ansible all -i inventory/{env}/hosts.yml -m shell -a "java -version"
```

#### 2. Vault Authentication Failures

**Issue**: Cannot retrieve secrets from Vault

**Symptoms**:
- "Failed to authenticate with Vault"
- "Secret not found"

**Solutions**:
```bash
# Test Vault connection
vault auth -method=approle role_id=<ROLE_ID> secret_id=<SECRET_ID>

# Verify secret exists
vault kv get secret/ping/platform8/ds/admin_password

# Check AppRole configuration
vault read auth/approle/role/ansible-ping-platform8
```

#### 3. DS Installation Failures

**Issue**: DS setup command fails

**Symptoms**:
- "Setup command failed"
- "Port already in use"
- "Certificate import failed"

**Solutions**:
```bash
# Check if DS is already running
ansible ds -i inventory/{env}/hosts.yml -m shell -a "ps aux | grep opendj"

# Verify ports are free
ansible ds -i inventory/{env}/hosts.yml -m shell -a "netstat -tuln | grep -E ':(1389|2389|3389)'"

# Check DS logs
ansible ds -i inventory/{env}/hosts.yml -m shell -a "tail -100 {{ ds_install_dir }}/{{ ds_instance }}/logs/errors"
```

#### 4. AM Deployment Failures

**Issue**: AM WAR deployment fails or Amster fails

**Symptoms**:
- "Tomcat not responding"
- "Amster installation failed"
- "AM not accessible after deployment"

**Solutions**:
```bash
# Check Tomcat status
ansible am -i inventory/{env}/hosts.yml -m systemd -a "name=tomcat state=status"

# Check Tomcat logs
ansible am -i inventory/{env}/hosts.yml -m shell -a "tail -100 {{ tomcat_dir }}/logs/catalina.out"

# Verify AM is deployed
ansible am -i inventory/{env}/hosts.yml -m shell -a "ls -la {{ tomcat_webapps_dir }}/{{ am_context }}.war"

# Test AM accessibility
curl -I http://{{ am_hostname }}:{{ tomcat_http_port }}/{{ am_context }}
```

#### 5. IDM Deployment Failures

**Issue**: IDM fails to start or connect to DS

**Symptoms**:
- "IDM startup failed"
- "Cannot connect to DS"
- "Certificate import failed"

**Solutions**:
```bash
# Check IDM process
ansible idm -i inventory/{env}/hosts.yml -m shell -a "ps aux | grep openidm"

# Check IDM logs
ansible idm -i inventory/{env}/hosts.yml -m shell -a "tail -100 {{ idm_extract_dir }}/logs/openidm*.log"

# Verify DS certificate exists
ansible idm -i inventory/{env}/hosts.yml -m shell -a "ls -la {{ idm_cert_file }}"

# Test IDM connectivity
curl http://{{ idm_hostname }}:{{ boot_port_http }}/openidm/info/ping
```

#### 6. Replication Failures

**Issue**: DS replication setup fails

**Symptoms**:
- "Replication enable failed"
- "Replication initialization failed"
- "Replication status shows errors"

**Solutions**:
```bash
# Check replication status
ansible ds -i inventory/{env}/hosts.yml -m shell -a "{{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/dsreplication status"

# Verify connectivity between replicas
ansible ds -i inventory/{env}/hosts.yml -m shell -a "ping -c 3 {{ ds_replica_hostname }}"

# Check replication logs
ansible ds -i inventory/{env}/hosts.yml -m shell -a "tail -100 {{ ds_install_dir }}/{{ ds_instance }}/logs/replication"
```

### Log Locations

#### DS Logs
- **Error Log**: `{{ ds_install_dir }}/{{ ds_instance }}/logs/errors`
- **Access Log**: `{{ ds_install_dir }}/{{ ds_instance }}/logs/access`
- **Replication Log**: `{{ ds_install_dir }}/{{ ds_instance }}/logs/replication`

#### AM Logs
- **Tomcat Log**: `{{ tomcat_dir }}/logs/catalina.out`
- **AM Log**: `{{ am_dir }}/logs/amAuthentication.access`
- **Amster Log**: Check console output during Amster execution

#### IDM Logs
- **IDM Log**: `{{ idm_extract_dir }}/logs/openidm*.log`
- **Audit Log**: `{{ idm_extract_dir }}/logs/audit*.log`

#### IG Logs
- **IG Log**: `{{ ig_dir }}/logs/`

### Debugging Commands

#### Check Component Status

```bash
# DS Status
ansible ds -i inventory/{env}/hosts.yml -m shell -a "{{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/status"

# AM Status
ansible am -i inventory/{env}/hosts.yml -m uri -a "url=http://{{ am_hostname }}:{{ tomcat_http_port }}/{{ am_context }}/json/serverinfo/*"

# IDM Status
ansible idm -i inventory/{env}/hosts.yml -m uri -a "url=http://{{ idm_hostname }}:{{ boot_port_http }}/openidm/info/ping"
```

#### Check Service Status

```bash
# Tomcat
ansible am -i inventory/{env}/hosts.yml -m systemd -a "name=tomcat state=status"

# IDM (check process)
ansible idm -i inventory/{env}/hosts.yml -m shell -a "pgrep -f openidm"
```

#### Check Connectivity

```bash
# Test LDAP connectivity
ansible ds -i inventory/{env}/hosts.yml -m shell -a "{{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/ldapsearch -h localhost -p {{ ds_port }} -D '{{ ds_admin_dn }}' -w {{ vault('secret/ping/platform8/ds/admin_password') }} -b '{{ ds_base_dn }}' 'objectclass=*' dn"

# Test HTTP connectivity
ansible all -i inventory/{env}/hosts.yml -m uri -a "url=http://{{ ansible_host }}:{{ http_port }}"
```

## Best Practices

### Security Best Practices

1. **Credential Management**
   - Never commit credentials to Git
   - Use HashiCorp Vault for all secrets
   - Rotate credentials regularly
   - Use AppRole for automation (not user tokens)

2. **SSH Security**
   - Use SSH keys, not passwords
   - Implement SSH key rotation
   - Use `ansible_ssh_common_args` for SSH options
   - Limit SSH access to necessary hosts

3. **Playbook Security**
   - Limit `become` usage to necessary tasks
   - Use `no_log: true` for tasks that output sensitive data
   - Implement least privilege principles
   - Review playbooks for security issues

### Deployment Best Practices

1. **Environment Progression**
   - Always test in dev first
   - Validate in test before production
   - Use check mode before production deployments
   - Document all changes

2. **Rolling Updates**
   - Update one replica at a time
   - Verify each replica before proceeding
   - Maintain service availability during updates
   - Have rollback plan ready

3. **Backup Strategy**
   - Backup before any update
   - Store backups in secure location
   - Test backup restoration
   - Document backup procedures

4. **Monitoring**
   - Monitor logs during deployment
   - Set up health checks
   - Implement alerting
   - Track deployment metrics

### Idempotency Best Practices

1. **Detection First**
   - Always detect existing installation before acting
   - Use facts to track installation status
   - Check version compatibility

2. **Conditional Execution**
   - Use `when` clauses based on detection
   - Skip unnecessary tasks
   - Only update what changed

3. **Configuration Preservation**
   - Never overwrite existing configs
   - Backup before updates
   - Merge changes carefully
   - Preserve customizations

### Maintenance Best Practices

1. **Regular Updates**
   - Keep Ansible updated
   - Update playbooks for new versions
   - Review and update documentation
   - Test updates in dev first

2. **Version Control**
   - Use Git for all playbooks
   - Tag releases
   - Document changes
   - Review changes before merging

3. **Testing**
   - Test in dev environment
   - Use check mode
   - Validate after deployment
   - Monitor for issues

## Reference Links

### Ping Identity Documentation

- **Ping Identity AM 8.0**: https://documentation.pingidentity.com/platform/8/sample-setup/am-setup-1.html
- **Ping Directory 8.0**: https://docs.ping.directory/PingDirectory/8.0/
- **Ping Directory Replication**: https://docs.ping.directory/PingDirectory/latest/cli/dsreplication.html
- **Ping Identity DevOps**: https://developer.pingidentity.com/devops/devops-landing-page.html
- **Configuration as Code**: https://developer.pingidentity.com/config-automation-promotion/concepts/configuration_as_code.html

### Ansible Documentation

- **Ansible Best Practices**: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_best_practices.html
- **Ansible Variables**: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html
- **Ansible Roles**: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html
- **Ansible Vault**: https://docs.ansible.com/ansible/latest/vault_guide/vault.html

### HashiCorp Vault Documentation

- **Vault AppRole**: https://www.vaultproject.io/docs/auth/approle
- **Vault KV Secrets**: https://www.vaultproject.io/docs/secrets/kv
- **Vault API**: https://www.vaultproject.io/api

## Maintenance Procedures

### Regular Maintenance Tasks

#### Weekly

1. Review logs for errors
2. Check component health
3. Verify replication status
4. Review security alerts

#### Monthly

1. Update Ansible playbooks
2. Review and update documentation
3. Test backup restoration
4. Review access logs

#### Quarterly

1. Rotate credentials
2. Update software versions
3. Review and update inventory
4. Performance review

### Update Procedures

#### Update Ansible Playbooks

```bash
# Pull latest changes
git pull origin main

# Review changes
git log --oneline

# Test in dev
ansible-playbook -i inventory/dev/hosts.yml playbooks/site.yml --check

# Deploy to dev
ansible-playbook -i inventory/dev/hosts.yml playbooks/site.yml
```

#### Update Component Versions

1. Update version variables in inventory
2. Place new software binaries
3. Test in dev environment
4. Deploy using update mode

#### Update Vault Secrets

```bash
# Update secret in Vault
vault kv put secret/ping/platform8/ds/admin_password value="newpassword"

# Verify update
vault kv get secret/ping/platform8/ds/admin_password

# Test with Ansible
ansible-playbook -i inventory/dev/hosts.yml playbooks/test-vault.yml
```

### Backup and Restore

#### Backup Procedures

```bash
# Backup DS
ansible ds -i inventory/{env}/hosts.yml -m archive -a "path={{ ds_install_dir }}/{{ ds_instance }}/config dest=/backup/ds-{{ ds_instance }}-{{ timestamp }}.tar.gz"

# Backup AM
ansible am -i inventory/{env}/hosts.yml -m archive -a "path={{ am_cfg_dir }},{{ am_dir }} dest=/backup/am-{{ timestamp }}.tar.gz"

# Backup IDM
ansible idm -i inventory/{env}/hosts.yml -m archive -a "path={{ idm_config_dir }},{{ idm_extract_dir }}/db dest=/backup/idm-{{ timestamp }}.tar.gz"
```

#### Restore Procedures

```bash
# Restore DS
ansible ds -i inventory/{env}/hosts.yml -m unarchive -a "src=/backup/ds-{{ ds_instance }}-{{ timestamp }}.tar.gz dest={{ ds_install_dir }}/{{ ds_instance }}"

# Restore AM
ansible am -i inventory/{env}/hosts.yml -m unarchive -a "src=/backup/am-{{ timestamp }}.tar.gz dest=/"

# Restore IDM
ansible idm -i inventory/{env}/hosts.yml -m unarchive -a "src=/backup/idm-{{ timestamp }}.tar.gz dest=/"
```

## Performance Tuning

### DS Performance

- Adjust connection pool sizes
- Tune replication settings
- Optimize indexes
- Monitor disk I/O

### AM Performance

- Tune Tomcat settings
- Adjust session timeout
- Optimize authentication trees
- Monitor memory usage

### IDM Performance

- Tune reconciliation schedules
- Optimize synchronization mappings
- Adjust connection pools
- Monitor resource usage

## Monitoring and Alerting

### Health Checks

```bash
# DS Health Check
ansible ds -i inventory/{env}/hosts.yml -m shell -a "{{ ds_install_dir }}/{{ ds_instance }}/opendj/bin/status"

# AM Health Check
ansible am -i inventory/{env}/hosts.yml -m uri -a "url=http://{{ am_hostname }}:{{ tomcat_http_port }}/{{ am_context }}/json/serverinfo/*"

# IDM Health Check
ansible idm -i inventory/{env}/hosts.yml -m uri -a "url=http://{{ idm_hostname }}:{{ boot_port_http }}/openidm/info/ping"
```

### Log Monitoring

- Set up centralized logging
- Configure log rotation
- Monitor error logs
- Alert on critical errors

## Support and Resources

### Getting Help

1. **Documentation**: Review all 8 documentation files
2. **Logs**: Check component logs for errors
3. **Validation**: Run validation playbook
4. **Community**: Ping Identity community forums

### Useful Commands

```bash
# List all playbooks
ls -la playbooks/

# List all roles
ls -la roles/

# Check inventory
ansible-inventory -i inventory/{env}/hosts.yml --list

# Test connectivity
ansible all -i inventory/{env}/hosts.yml -m ping

# Run with verbose output
ansible-playbook -i inventory/{env}/hosts.yml playbooks/site.yml -vvv
```

## Next Steps

After reviewing this guide:
- Refer to specific documentation files for detailed procedures
- Test all procedures in dev environment first
- Document any environment-specific customizations
- Keep documentation updated

