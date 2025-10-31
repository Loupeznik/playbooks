# Quick Start Guide

Get your hardened Linux servers up and running in minutes!

## Prerequisites

- Ansible 2.12+ installed on your control node
- Fresh Ubuntu 24.04 or AlmaLinux 10.x server(s)
- SSH access to the servers
- Sudo privileges on the servers

## 5-Minute Setup

### 1. Install Ansible Collections

```bash
ansible-galaxy collection install community.general ansible.posix community.postgresql community.docker
pip3 install passlib
```

### 2. Generate SSH Key (if needed)

```bash
ssh-keygen -t ed25519 -C "ansible@yourdomain.com"
ssh-copy-id -i ~/.ssh/id_ed25519.pub youruser@yourserver
```

### 3. Configure Inventory

Edit `inventory/production.yml` with your server details:

```yaml
all:
  vars:
    ansible_user: sysadmin
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519
    ansible_become: true

  children:
    web_servers:
      hosts:
        web-1:
          ansible_host: 10.9.11.20
        web-2:
          ansible_host: 10.9.11.21

    db_servers:
      hosts:
        db-1:
          ansible_host: 10.9.11.22

    ubuntu_servers:
      children:
        web_servers:

    almalinux_servers:
      children:
        db_servers:
```

### 4. Update Variables

Edit `inventory/group_vars/all.yml`:

```yaml
# Update your SSH public key
admin_ssh_key: "ssh-ed25519 AAAAC3NzaC1... ansible@yourdomain.com"

# Set your SSH port (default is 2222)
ssh_port: 2222
```

### 5. Test Connectivity

```bash
cd hardened-linux
ansible all -i inventory/production.yml -m ping
```

### 6. Run Basic Hardening

**For Ubuntu servers:**
```bash
ansible-playbook -i inventory/production.yml ubuntu/01-basic-setup.yml --limit ubuntu_servers
```

**For AlmaLinux servers:**
```bash
ansible-playbook -i inventory/production.yml almalinux/01-basic-setup.yml --limit almalinux_servers
```

### 7. Reboot Servers

After basic hardening completes:

```bash
ansible all -i inventory/production.yml -b -m reboot
```

**IMPORTANT:** After reboot, SSH port will change to 2222 (or your configured port)

### 8. Run Server-Specific Playbooks

**Web/App Servers (Ubuntu):**
```bash
ansible-playbook -i inventory/production.yml ubuntu/02-appserver.yml \
  --limit web_servers \
  -e ansible_port=2222
```

**Database Servers (AlmaLinux):**
```bash
ansible-playbook -i inventory/production.yml almalinux/03-dbserver.yml \
  --limit db_servers \
  -e ansible_port=2222
```

**Build Servers (Ubuntu):**
```bash
ansible-playbook -i inventory/production.yml ubuntu/04-buildserver.yml \
  --limit build_servers \
  -e ansible_port=2222
```

**Build Servers (AlmaLinux):**
```bash
ansible-playbook -i inventory/production.yml almalinux/04-buildserver.yml \
  --limit build_servers \
  -e ansible_port=2222
```

## Testing Hardening

### Verify SSH Configuration

```bash
ssh -p 2222 youruser@yourserver
```

### Check Firewall

**Ubuntu and AlmaLinux:**
```bash
ssh -p 2222 yourserver "sudo firewall-cmd --list-all"
```

### Verify Services

```bash
# Check fail2ban
sudo fail2ban-client status

# Check auditd
sudo auditctl -l

# Check SELinux (AlmaLinux)
sudo getenforce

# Check AppArmor (Ubuntu)
sudo aa-status
```

## Common Issues

### Can't SSH After Running Playbook

**Problem:** SSH port changed to 2222

**Solution:**
```bash
ssh -p 2222 youruser@yourserver
```

### Playbook Fails on Package Installation

**Problem:** Package cache out of date

**Solution:**
```bash
# Ubuntu
ansible all -i inventory/hosts.yml -b -m apt -a "update_cache=yes"

# AlmaLinux
ansible all -i inventory/hosts.yml -b -m dnf -a "name=* state=latest update_cache=yes"
```

### Permission Denied

**Problem:** Need to provide sudo password

**Solution:**
```bash
ansible-playbook -i inventory/hosts.yml ubuntu/01-basic-setup.yml --ask-become-pass
```

## Next Steps

1. **Review variables** in `inventory/group_vars/` and customize for your environment
2. **Change default passwords** (especially database passwords)
3. **Configure SSL certificates** for web servers
4. **Set up monitoring** and log aggregation
5. **Test backups** (especially for database servers)
6. **Document your configuration**

## Security Checklist

- [ ] SSH key-only authentication configured
- [ ] Firewall rules tested and verified
- [ ] Root account locked
- [ ] Password policy enforced
- [ ] Automatic updates enabled
- [ ] Audit logging configured
- [ ] Fail2ban monitoring SSH
- [ ] SELinux/AppArmor enforcing
- [ ] Default passwords changed
- [ ] Backup scripts tested

## Example: Complete Setup for Web Servers

```bash
# 1. Navigate to playbooks directory
cd hardened-linux

# 2. Test connectivity
ansible web_servers -i inventory/production.yml -m ping

# 3. Run basic hardening (targets all Ubuntu servers including web_servers)
ansible-playbook -i inventory/production.yml ubuntu/01-basic-setup.yml --limit ubuntu_servers

# 4. Reboot
ansible web_servers -i inventory/production.yml -b -m reboot

# 5. Wait for server to come back up (2-3 minutes)

# 6. Update SSH config to use port 2222
# Edit ~/.ssh/config:
# Host web-1
#     HostName 10.9.11.20
#     Port 2222
# Host web-2
#     HostName 10.9.11.21
#     Port 2222

# 7. Run web server specific hardening
ansible-playbook -i inventory/production.yml ubuntu/02-appserver.yml \
  --limit web_servers \
  -e ansible_port=2222

# 8. Verify
ssh -p 2222 sysadmin@10.9.11.20 "sudo nginx -t && sudo docker ps"
```

## Getting Help

- Review full documentation in [README.md](README.md)
- Check pre-installation guide in [PRE_INSTALLATION.md](PRE_INSTALLATION.md)
- Review troubleshooting section in README.md

## Important Notes

- **Always test in non-production first**
- **Take snapshots/backups before running playbooks**
- **Review all variables before applying**
- **Update default passwords immediately**
- **Document any customizations**

Good luck with your hardened Linux servers!
