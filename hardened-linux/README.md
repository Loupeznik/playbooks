# Hardened Linux Ansible Playbooks

Comprehensive Ansible playbooks for hardening Ubuntu and AlmaLinux servers. This repository is designed to be used as a git submodule within the [hardened-linux-infra](https://github.com/Loupeznik/hardened-linux-infra) repository for complete infrastructure deployment with Packer and Terraform.

For full infrastructure deployment workflow (Packer + Terraform + Ansible), see the main [hardened-linux-infra](https://github.com/Loupeznik/hardened-linux-infra) repository.

## Table of Contents

- [Overview](#overview)
- [SSH Port Configuration](#ssh-port-configuration)
- [Directory Structure](#directory-structure)
- [Prerequisites](#prerequisites)
- [Server Installation Steps](#server-installation-steps)
- [Playbook Execution](#playbook-execution)
- [Playbook Descriptions](#playbook-descriptions)
- [Variables](#variables)
- [Post-Installation Verification](#post-installation-verification)
- [Troubleshooting](#troubleshooting)

## Overview

This collection provides security hardening for:

- **Ubuntu 24.04 LTS (Noble Numbat)** - Debian-based systems
- **AlmaLinux 10.x** - Enterprise Linux (RHEL-compatible)

**Playbooks Available:**

1. **Basic Setup** - Core hardening (SSH, firewall, kernel, auditing)
2. **App Server** - Nginx, Docker, application-specific hardening
3. **Database Server** - PostgreSQL hardening and security
4. **Build Server** - CI/CD runner hardening (GitHub Actions, Azure DevOps)
5. **Monitoring** - VictoriaMetrics, Grafana Loki, Alloy, exporters, WireGuard VPN
6. **Compliance** - OpenSCAP, Lynis, Trivy security scanning

## SSH Port Configuration

### Important: SSH Port Changes from 22 to 2222

The basic hardening playbooks (`01-basic-setup.yml`) change the SSH port from the default **22** to **2222** for enhanced security. This change happens automatically during the first playbook run.

**What you need to know:**

1. **Initial Connection**: First playbook run uses port 22
   ```bash
   ansible-playbook -i inventory/production.yml ubuntu/01-basic-setup.yml
   ```

2. **After First Run**: Update your inventory to use port 2222
   ```yaml
   all:
     vars:
       ansible_port: 2222  # Add this line
   ```

3. **SSH Connections**: Always use port 2222 after hardening
   ```bash
   ssh -p 2222 sysadmin@server-ip
   ```

4. **Firewall**: Port 2222 is automatically opened in firewalld

5. **SELinux (AlmaLinux)**: Port 2222 is automatically added to SSH port context
   ```bash
   # This is done automatically by the playbook:
   semanage port -a -t ssh_port_t -p tcp 2222
   ```

**Troubleshooting:**

If you get connection refused on port 22 after hardening:
- This is expected - the port has changed to 2222
- Update your inventory: `ansible_port: 2222`
- Connect using: `ssh -p 2222 user@host`

If SSH fails to restart on AlmaLinux (SELinux issue):
- Manually add port: `sudo semanage port -a -t ssh_port_t -p tcp 2222`
- Restart SSH: `sudo systemctl restart sshd`

## Directory Structure

```
ansible/playbooks/hardened-linux/
├── README.md                          # This file
├── PRE_INSTALLATION.md                # Pre-installation guide
├── QUICKSTART.md                      # Quick deployment guide
├── inventory/
│   ├── hosts.yml                      # Inventory file
│   ├── production.yml                 # Production inventory
│   ├── staging.yml                    # Staging inventory
│   └── group_vars/
│       ├── all.yml                    # Common variables
│       ├── ubuntu_servers.yml         # Ubuntu-specific variables
│       ├── almalinux_servers.yml      # AlmaLinux-specific variables
│       ├── appservers.yml             # App server variables
│       ├── dbservers.yml              # Database server variables
│       └── buildservers.yml           # Build server variables
├── ubuntu/
│   ├── 01-basic-setup.yml             # Ubuntu basic hardening
│   ├── 02-appserver.yml               # Ubuntu app server setup
│   ├── 03-dbserver.yml                # Ubuntu database server setup
│   └── 04-buildserver.yml             # Ubuntu build server setup
├── almalinux/
│   ├── 01-basic-setup.yml             # AlmaLinux basic hardening
│   ├── 02-appserver.yml               # AlmaLinux app server setup
│   ├── 03-dbserver.yml                # AlmaLinux database server setup
│   └── 04-buildserver.yml             # AlmaLinux build server setup
├── monitoring/                         # NEW: Monitoring & logging
│   ├── 01-install-agents.yml          # Install node_exporter, Alloy, exporters
│   ├── 02-setup-wireguard.yml         # WireGuard VPN for secure monitoring
│   └── 03-central-stack.yml           # VictoriaMetrics + Loki + Grafana
├── compliance/                         # NEW: Security scanning
│   ├── 01-install-scanners.yml        # Install OpenSCAP, Lynis, Trivy
│   └── 02-schedule-scans.yml          # Configure automated scans
├── roles/                              # NEW: Reusable Ansible roles
│   ├── node_exporter/                 # Prometheus node_exporter
│   ├── alloy/                         # Grafana Alloy log shipper
│   ├── postgres_exporter/             # PostgreSQL metrics
│   ├── nginx_exporter/                # Nginx metrics
│   ├── openscap/                      # CIS benchmark scanning
│   ├── lynis/                         # Security auditing
│   ├── trivy/                         # Vulnerability scanning
│   └── wireguard/                     # VPN for monitoring traffic
├── templates/
│   ├── sshd_config.j2                 # SSH daemon configuration
│   ├── sysctl-hardening.conf.j2       # Kernel parameters
│   ├── audit-rules.j2                 # Audit rules
│   ├── nginx.conf.j2                  # Nginx configuration
│   ├── postgresql.conf.j2             # PostgreSQL configuration
│   ├── pg_hba.conf.j2                 # PostgreSQL access control
│   ├── fail2ban-jail.local.j2         # Fail2ban configuration
│   ├── victoriametrics-scrape.yml.j2  # VictoriaMetrics scrape config
│   ├── loki-config.yml.j2             # Loki configuration
│   └── grafana-datasources.yml.j2     # Grafana datasources
└── files/
    ├── docker-daemon.json             # Docker daemon config
    └── aide-cron                      # AIDE scheduled check
```

## Prerequisites

### On the Control Node (Your workstation)

1. **Python Virtual Environment Setup**
   ```bash
   # Create virtual environment
   python3 -m venv venv

   # Activate virtual environment
   source venv/bin/activate

   # Install Ansible (version 2.12+)
   pip install ansible

   # Alternatively, install via system package manager
   # Ubuntu/Debian
   sudo apt update
   sudo apt install ansible

   # macOS
   brew install ansible
   ```

2. **SSH Key Pair**
   ```bash
   # Generate SSH key if you don't have one
   ssh-keygen -t ed25519 -C "ansible-control@yourdomain.com"
   ```

3. **Additional Python Modules**
   ```bash
   # Ensure your virtual environment is activated
   source venv/bin/activate

   # Install required Python modules
   pip install passlib  # For generating password hashes
   ```

4. **Ansible Collections**
   ```bash
   ansible-galaxy collection install community.general
   ansible-galaxy collection install ansible.posix
   ```

### Server Requirements

- Fresh minimal installation of Ubuntu 22.04 LTS or AlmaLinux 9.x
- Root access or sudo privileges
- Network connectivity
- Minimum 2GB RAM, 20GB disk space
- Static IP address configured (recommended)

## Server Installation Steps

See [PRE_INSTALLATION.md](PRE_INSTALLATION.md) for detailed server installation steps.

**Quick Summary:**

1. **Install OS** with minimal configuration
2. **Set hostname** and network configuration
3. **Create initial admin user** with sudo privileges
4. **Copy SSH public key** to the server
5. **Test SSH connection** from control node
6. **Update inventory file** with server details

## Playbook Execution

### Step 1: Configure Inventory

Edit `inventory/hosts.yml` to add your servers:

```yaml
all:
  children:
    ubuntu:
      children:
        ubuntu_appservers:
          hosts:
            app01.example.com:
              ansible_host: 10.0.1.10
        ubuntu_dbservers:
          hosts:
            db01.example.com:
              ansible_host: 10.0.2.10
        ubuntu_buildservers:
          hosts:
            build01.example.com:
              ansible_host: 10.0.3.10

    almalinux:
      children:
        almalinux_appservers:
          hosts:
            app02.example.com:
              ansible_host: 10.0.1.20
```

### Step 2: Configure Variables

Edit files in `inventory/group_vars/` to customize settings:

- `all.yml` - Common settings (SSH port, users, etc.)
- Distribution-specific files
- Server type-specific files

### Step 3: Test Connectivity

```bash
cd ansible/playbooks/hardened-linux
ansible all -i inventory/hosts.yml -m ping
```

### Step 4: Run Playbooks

**For Ubuntu servers:**

```bash
# Basic hardening (run first)
ansible-playbook -i inventory/hosts.yml ubuntu/01-basic-setup.yml

# Then run specific server type playbook
ansible-playbook -i inventory/hosts.yml ubuntu/02-appserver.yml
# OR
ansible-playbook -i inventory/hosts.yml ubuntu/03-dbserver.yml
# OR
ansible-playbook -i inventory/hosts.yml ubuntu/04-buildserver.yml

# Optional: Setup monitoring (after basic hardening)
ansible-playbook -i inventory/hosts.yml monitoring/02-setup-wireguard.yml
ansible-playbook -i inventory/hosts.yml monitoring/01-install-agents.yml
ansible-playbook -i inventory/hosts.yml monitoring/03-central-stack.yml --limit monitoring_server

# Optional: Enable compliance scanning
ansible-playbook -i inventory/hosts.yml compliance/01-install-scanners.yml
ansible-playbook -i inventory/hosts.yml compliance/02-schedule-scans.yml
```

**For AlmaLinux servers:**

```bash
# Basic hardening (run first)
ansible-playbook -i inventory/hosts.yml almalinux/01-basic-setup.yml

# Then run specific server type playbook
ansible-playbook -i inventory/hosts.yml almalinux/02-appserver.yml
# OR
ansible-playbook -i inventory/hosts.yml almalinux/03-dbserver.yml
# OR
ansible-playbook -i inventory/hosts.yml almalinux/04-buildserver.yml

# Optional: Setup monitoring (after basic hardening)
ansible-playbook -i inventory/hosts.yml monitoring/02-setup-wireguard.yml
ansible-playbook -i inventory/hosts.yml monitoring/01-install-agents.yml
ansible-playbook -i inventory/hosts.yml monitoring/03-central-stack.yml --limit monitoring_server

# Optional: Enable compliance scanning
ansible-playbook -i inventory/hosts.yml compliance/01-install-scanners.yml
ansible-playbook -i inventory/hosts.yml compliance/02-schedule-scans.yml
```

**Run in check mode (dry-run):**

```bash
ansible-playbook -i inventory/hosts.yml ubuntu/01-basic-setup.yml --check
```

**Run with verbose output:**

```bash
ansible-playbook -i inventory/hosts.yml ubuntu/01-basic-setup.yml -vvv
```

**Target specific hosts:**

```bash
ansible-playbook -i inventory/hosts.yml ubuntu/02-appserver.yml --limit app01.example.com
```

## Playbook Descriptions

### 01-basic-setup.yml

**Purpose:** Core security hardening applied to all servers

**What it does:**
- System updates and package installation
- SSH hardening (key-only auth, non-standard port)
- Firewall setup (firewalld)
- Fail2ban installation and configuration
- Kernel hardening (sysctl parameters)
- User management and sudo configuration
- Auditd setup and rules
- AIDE intrusion detection
- Automatic security updates
- AppArmor/SELinux configuration
- File system hardening
- Disable unnecessary services
- NTP/chrony time synchronization

**Estimated run time:** 10-15 minutes

**Requires reboot:** Yes (after completion)

### 02-appserver.yml

**Purpose:** Application server hardening for web applications

**What it does:**
- Nginx installation and hardening
- SSL/TLS configuration
- Security headers
- Rate limiting
- Docker installation and hardening (optional)
- Application user creation
- Log rotation
- Firewall rules for HTTP/HTTPS
- Resource limits

**Estimated run time:** 8-12 minutes

**Requires reboot:** No

### 03-dbserver.yml

**Purpose:** Database server hardening for PostgreSQL

**What it does:**
- PostgreSQL installation
- Database-specific firewall rules
- SSL/TLS for connections
- Access control (pg_hba.conf)
- Connection logging
- Performance tuning (basic)
- Backup user configuration
- Data directory permissions
- Network binding restrictions

**Estimated run time:** 10-15 minutes

**Requires reboot:** No

### 04-buildserver.yml

**Purpose:** CI/CD build server hardening

**What it does:**
- Docker installation and hardening
- Runner user creation
- Resource limits for builds
- Cleanup jobs (cron)
- Ephemeral runner configuration
- Build artifact cleanup
- Restricted sudo access
- Docker image pruning
- Network restrictions

**Estimated run time:** 8-12 minutes

**Requires reboot:** No

### monitoring/02-setup-wireguard.yml

**Purpose:** Establish encrypted VPN tunnels for secure monitoring traffic

**What it does:**
- WireGuard installation and configuration
- Site-to-site VPN between all hosts and monitoring server
- Automatic key generation and exchange
- Firewall rules for VPN traffic
- IP forwarding configuration (monitoring server)
- Connectivity verification

**Estimated run time:** 5-8 minutes

**Requires reboot:** No

**Network:** Creates 10.100.0.0/24 VPN subnet

### monitoring/01-install-agents.yml

**Purpose:** Deploy monitoring and logging agents on all hosts

**What it does:**
- node_exporter installation (system metrics)
- Grafana Alloy installation (log shipping to Loki)
- postgres_exporter (database servers only)
- nginx_exporter (application servers only)
- Bind agents to WireGuard interface
- Configure log collection and forwarding
- Export security scan metrics

**Estimated run time:** 8-12 minutes

**Requires reboot:** No

**Ports used:** 9100 (node_exporter), 9080 (Alloy), 9187 (postgres), 9113 (nginx)

### monitoring/03-central-stack.yml

**Purpose:** Deploy central monitoring infrastructure

**What it does:**
- VictoriaMetrics installation (metrics database)
- Grafana Loki installation (log aggregation)
- Grafana installation (visualization)
- Alertmanager installation (alerting)
- Datasource provisioning
- Scrape configuration for all hosts
- Service startup and verification

**Estimated run time:** 15-20 minutes

**Requires reboot:** No

**Access URLs:** Grafana (3000), VictoriaMetrics (8428), Loki (3100), Alertmanager (9093)

### compliance/01-install-scanners.yml

**Purpose:** Deploy security scanning and compliance tools

**What it does:**
- OpenSCAP installation (CIS benchmark scanning)
- Lynis installation (security auditing)
- Trivy installation (vulnerability scanning)
- Scan script deployment
- Report directory creation
- Integration with monitoring (metrics export)

**Estimated run time:** 8-12 minutes

**Requires reboot:** No

**Report locations:** /var/log/openscap, /var/log/lynis, /var/log/trivy

### compliance/02-schedule-scans.yml

**Purpose:** Configure automated security scanning

**What it does:**
- Cron job configuration for automated scans
- OpenSCAP weekly CIS benchmark scans
- Lynis daily security audits
- Trivy daily filesystem/Docker scans
- Log rotation for scan results
- Email notification setup
- Systemd timer alternatives

**Estimated run time:** 3-5 minutes

**Requires reboot:** No

**Scan schedules:** OpenSCAP (Sun 3am), Lynis (Daily 4:30am), Trivy (Daily 2am)

## Variables

### Common Variables (all.yml)

```yaml
# SSH Configuration
ssh_port: 2222
ssh_permit_root_login: no
ssh_password_authentication: no

# Admin user
admin_username: sysadmin
admin_ssh_key: "ssh-ed25519 AAAAC3N... user@host"

# Firewall
firewall_allowed_tcp_ports: []
firewall_allowed_udp_ports: []

# Security
enable_aide: true
enable_fail2ban: true
enable_auditd: true
```

### App Server Variables (appservers.yml)

```yaml
# Nginx
nginx_install: true
nginx_enable_ssl: true
nginx_ssl_cert_path: /etc/ssl/certs/server.crt
nginx_ssl_key_path: /etc/ssl/private/server.key

# Docker
docker_install: true
docker_users: ["appuser"]
```

### Database Server Variables (dbservers.yml)

```yaml
# PostgreSQL
postgres_version: "14"
postgres_listen_addresses: "10.0.2.10"
postgres_allowed_networks: ["10.0.1.0/24"]
postgres_ssl_enabled: true
postgres_max_connections: 100
```

### Build Server Variables (buildservers.yml)

```yaml
# Runner configuration
runner_user: runner
runner_type: github  # or: azuredevops
docker_prune_schedule: "0 2 * * *"
build_cleanup_days: 7
```

### Monitoring Variables (group_vars/all.yml)

```yaml
# WireGuard VPN
wireguard_enabled: true
wireguard_port: 51820
wireguard_network: "10.100.0.0/24"
monitoring_server_host: "monitoring-01"

# Per-host (in host_vars/)
wireguard_client_ip: "10.100.0.11"  # Unique per host

# Monitoring endpoints
loki_server: "{{ hostvars[monitoring_server_host].wireguard_client_ip }}"
victoriametrics_server: "{{ hostvars[monitoring_server_host].wireguard_client_ip }}"

# Exporter versions
node_exporter_version: "1.7.0"
alloy_version: "1.0.0"
postgres_exporter_version: "0.15.0"
nginx_exporter_version: "0.11.0"
```

### Compliance Variables (group_vars/all.yml)

```yaml
# Security Scanning
openscap_scan_enabled: true
openscap_scan_schedule_weekday: "0"  # Sunday
openscap_scan_schedule_hour: "3"

lynis_scan_enabled: true
lynis_scan_schedule_hour: "4"
lynis_scan_schedule_minute: "30"

trivy_scan_enabled: true
trivy_fs_scan_schedule_hour: "2"
trivy_docker_scan_enabled: true

# Notifications
security_scan_email: "security@example.com"
```

## Post-Installation Verification

After running the playbooks, verify the hardening:

### 1. SSH Access Test

```bash
# Test SSH with new port and key
ssh -p 2222 -i ~/.ssh/id_ed25519 sysadmin@server.example.com

# Verify root login is disabled (should fail)
ssh -p 2222 root@server.example.com
```

### 2. Firewall Status

**Ubuntu and AlmaLinux:**
```bash
sudo firewall-cmd --list-all
```

### 3. Security Services

```bash
# Fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Auditd
sudo auditctl -l

# AIDE
sudo aide --check

# Automatic updates
sudo systemctl status unattended-upgrades  # Ubuntu
sudo systemctl status dnf-automatic.timer   # AlmaLinux
```

### 4. AppArmor/SELinux

**Ubuntu (AppArmor):**
```bash
sudo aa-status
```

**AlmaLinux (SELinux):**
```bash
sudo getenforce  # Should return "Enforcing"
sudo sestatus
```

### 5. Service-Specific Checks

**App Server (Nginx):**
```bash
sudo nginx -t
curl -I https://server.example.com
```

**Database Server (PostgreSQL):**
```bash
sudo -u postgres psql -c "SHOW ssl;"
sudo -u postgres psql -c "SELECT * FROM pg_hba_file_rules;"
```

**Build Server (Docker):**
```bash
docker info
docker ps
```

### 6. Kernel Parameters

```bash
sudo sysctl -a | grep -E "net.ipv4.tcp_syncookies|kernel.randomize_va_space"
```

### 7. Check for Failed Login Attempts

```bash
# Ubuntu
sudo journalctl -u ssh | grep -i failed

# AlmaLinux
sudo grep "Failed password" /var/log/secure
```

### 8. Monitoring & Logging (if deployed)

**WireGuard VPN:**
```bash
# Check WireGuard status
sudo wg show

# Test connectivity to monitoring server
ping -c 3 10.100.0.1

# Verify VPN interface
ip addr show wg0
```

**Monitoring Agents:**
```bash
# Check exporter services
sudo systemctl status node_exporter alloy

# Test metrics endpoint (via WireGuard)
curl -s http://10.100.0.11:9100/metrics | head

# Check Alloy log shipping
sudo journalctl -u alloy -n 50
```

**Central Monitoring Stack:**
```bash
# Access from monitoring server
curl -s http://localhost:8428/api/v1/targets  # VictoriaMetrics targets
curl -s http://localhost:3100/ready           # Loki readiness

# Access Grafana
# Navigate to http://monitoring-server:3000
# Default: admin/admin (change immediately!)
```

### 9. Compliance Scanning (if deployed)

**Check Scan Schedules:**
```bash
# View cron jobs
sudo crontab -l | grep -E "openscap|lynis|trivy"
```

**Review Scan Results:**
```bash
# OpenSCAP CIS benchmark
ls -lh /var/log/openscap/report-*.html
sudo cat /var/log/openscap/scan.log | tail

# Lynis hardening index
sudo grep hardening_index /var/log/lynis/lynis-report-*.dat | tail -1

# Trivy vulnerabilities
sudo cat /var/log/trivy/scan.log | tail
```

**Check Metrics Export:**
```bash
# Scan metrics exported to node_exporter
curl -s http://localhost:9100/metrics | grep -E "openscap|lynis|trivy"
```

## Troubleshooting

### SSH Connection Issues

**Problem:** Cannot connect after running basic-setup playbook

**Solutions:**
1. Check if SSH port changed:
   ```bash
   ssh -p 2222 user@server
   ```

2. Verify firewall allows SSH:
   ```bash
   sudo firewall-cmd --add-port=2222/tcp --permanent
   sudo firewall-cmd --reload
   ```

3. Check SSH service status:
   ```bash
   sudo systemctl status sshd
   ```

### Playbook Fails During Execution

**Problem:** Playbook stops with error

**Solutions:**
1. Run with verbose output:
   ```bash
   ansible-playbook -i inventory/hosts.yml playbook.yml -vvv
   ```

2. Check connectivity:
   ```bash
   ansible all -i inventory/hosts.yml -m ping
   ```

3. Verify sudo access:
   ```bash
   ansible all -i inventory/hosts.yml -m shell -a "sudo whoami" -b
   ```

### SELinux Denials (AlmaLinux)

**Problem:** Services fail due to SELinux

**Solutions:**
1. Check for denials:
   ```bash
   sudo ausearch -m avc -ts recent
   ```

2. Temporarily set to permissive (troubleshooting only):
   ```bash
   sudo setenforce 0
   ```

3. Generate policy module:
   ```bash
   sudo ausearch -m avc -ts recent | audit2allow -M mypolicy
   sudo semodule -i mypolicy.pp
   ```

### Firewall Blocking Legitimate Traffic

**Problem:** Application cannot connect through firewall

**Solutions:**
1. Check current rules:
   ```bash
   sudo firewall-cmd --list-all
   ```

2. Add exception:
   ```bash
   sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.1.0/24" port port="5432" protocol="tcp" accept'
   sudo firewall-cmd --reload
   ```

### Docker Permission Issues

**Problem:** User cannot run Docker commands

**Solutions:**
1. Verify user in docker group:
   ```bash
   groups username
   ```

2. Add user to docker group:
   ```bash
   sudo usermod -aG docker username
   ```

3. User must log out and back in

### PostgreSQL Connection Refused

**Problem:** Cannot connect to PostgreSQL from application server

**Solutions:**
1. Check listen addresses:
   ```bash
   sudo grep listen_addresses /etc/postgresql/14/main/postgresql.conf
   ```

2. Check pg_hba.conf:
   ```bash
   sudo cat /etc/postgresql/14/main/pg_hba.conf
   ```

3. Verify firewall:
   ```bash
   sudo firewall-cmd --list-all | grep 5432
   ```

4. Test connection:
   ```bash
   psql -h db_server_ip -U username -d database
   ```

### WireGuard VPN Not Working

**Problem:** Cannot connect via WireGuard VPN

**Solutions:**
1. Check WireGuard service:
   ```bash
   sudo systemctl status wg-quick@wg0
   sudo wg show
   ```

2. Verify firewall allows UDP/51820:
   ```bash
   sudo firewall-cmd --list-ports | grep 51820
   ```

3. Check peer configuration:
   ```bash
   sudo cat /etc/wireguard/wg0.conf
   ```

4. Test connectivity:
   ```bash
   ping -c 3 10.100.0.1  # Monitoring server VPN IP
   ```

5. Review logs:
   ```bash
   sudo journalctl -u wg-quick@wg0 -n 50
   ```

### Monitoring Agents Not Reporting

**Problem:** Metrics not appearing in VictoriaMetrics

**Solutions:**
1. Check exporter is running:
   ```bash
   sudo systemctl status node_exporter
   curl http://localhost:9100/metrics
   ```

2. Verify WireGuard connectivity:
   ```bash
   ping -c 3 {{ monitoring_server_wireguard_ip }}
   ```

3. Check VictoriaMetrics scrape targets (from monitoring server):
   ```bash
   curl http://localhost:8428/api/v1/targets
   ```

4. Review exporter logs:
   ```bash
   sudo journalctl -u node_exporter -n 50
   ```

### Logs Not Appearing in Loki

**Problem:** Logs not being forwarded to Loki

**Solutions:**
1. Check Alloy service:
   ```bash
   sudo systemctl status alloy
   sudo journalctl -u alloy -n 50
   ```

2. Verify Alloy configuration:
   ```bash
   sudo alloy fmt /etc/alloy/config.river --check
   ```

3. Test Loki endpoint (from client):
   ```bash
   curl -v http://{{ loki_server }}:3100/ready
   ```

4. Check Loki is receiving logs (monitoring server):
   ```bash
   curl -G -s "http://localhost:3100/loki/api/v1/query" --data-urlencode 'query={job="syslog"}'
   ```

### Compliance Scans Not Running

**Problem:** Automated scans not executing

**Solutions:**
1. Verify cron jobs exist:
   ```bash
   sudo crontab -l | grep -E "openscap|lynis|trivy"
   ```

2. Check scan script permissions:
   ```bash
   ls -l /usr/local/bin/*-scan.sh
   sudo chmod +x /usr/local/bin/*-scan.sh
   ```

3. Run scan manually to test:
   ```bash
   sudo /usr/local/bin/openscap-scan.sh
   sudo /usr/local/bin/lynis-scan.sh
   sudo /usr/local/bin/trivy-scan.sh filesystem
   ```

4. Check cron execution logs:
   ```bash
   sudo journalctl -t CRON | grep -E "openscap|lynis|trivy"
   ```

### High Resource Usage from Alloy

**Problem:** Alloy consuming excessive CPU/memory

**Solutions:**
1. Monitor resource usage:
   ```bash
   ps aux | grep alloy
   ```

2. Reduce log collection scope in `/etc/alloy/config.river`:
   - Decrease file watch frequency
   - Limit journald max_age
   - Filter out verbose logs

3. Expected resource usage:
   - CPU: 1-2 cores (10x more than Promtail)
   - Memory: 300Mi-1GiB (2-6x more than Promtail)

## Additional Resources

### Internal Documentation
- [Security Posture Plan](../security-posture-plan.md) - Comprehensive security implementation guide with monitoring, compliance, and VPN setup

### External Resources
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/)
- [NIST Security Guidelines](https://www.nist.gov/cybersecurity)
- [Ansible Documentation](https://docs.ansible.com/)
- [Ubuntu Security](https://ubuntu.com/security)
- [AlmaLinux Security](https://wiki.almalinux.org/documentation/security.html)

### Monitoring & Logging
- [VictoriaMetrics Documentation](https://docs.victoriametrics.com/)
- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/latest/)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)
- [WireGuard Documentation](https://www.wireguard.com/quickstart/)

### Compliance Tools
- [OpenSCAP User Manual](https://www.open-scap.org/resources/documentation/)
- [Lynis Documentation](https://cisofy.com/documentation/lynis/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)

## Maintenance

### Regular Tasks

#### Security Checks (Weekly)

1. **Review logs:**
   ```bash
   sudo journalctl -p err -S today
   ```

2. **Check AIDE reports:**
   ```bash
   sudo aide --check
   ```

3. **Review fail2ban bans:**
   ```bash
   sudo fail2ban-client status sshd
   ```

4. **Review audit logs:**
   ```bash
   sudo ausearch -ts today -i
   ```

#### Monitoring & Compliance (Weekly)

5. **Review compliance scan results:**
   ```bash
   # Check OpenSCAP scores
   ls -lt /var/log/openscap/report-*.html | head -3

   # Check Lynis hardening index
   grep hardening_index /var/log/lynis/lynis-report-*.dat | tail -5

   # Review critical vulnerabilities
   grep -i critical /var/log/trivy/scan.log | tail -20
   ```

6. **Verify monitoring health:**
   ```bash
   # Check WireGuard connectivity
   sudo wg show

   # Verify exporters are running
   sudo systemctl status node_exporter alloy

   # Check metrics availability
   curl -s http://localhost:9100/metrics | head
   ```

7. **Review Grafana dashboards** for anomalies and alerts

#### Updates (Monthly)

8. **Update playbooks and re-run** to ensure compliance:
   ```bash
   git pull
   ansible-playbook -i inventory/hosts.yml ubuntu/01-basic-setup.yml --check
   ```

9. **Update security scanner databases:**
   ```bash
   sudo trivy image --download-db-only
   sudo lynis update info
   ```

10. **Review and update firewall rules** based on traffic patterns

## Support and Contributions

For issues or improvements, please follow the standard contribution process for this repository.

## License

Internal use only - follows company security policies and compliance requirements.
