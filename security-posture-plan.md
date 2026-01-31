# Comprehensive Linux Security Posture Improvement Plan

## Executive Summary

This document outlines a complete security improvement strategy for Ubuntu 24.04 and AlmaLinux 9 infrastructure, covering compliance, monitoring, logging, and container security. All components are managed via Ansible and centralized in a lightweight monitoring stack.

---

## 1. Recommended Tool Stack

### 1.1 Compliance & Configuration Management

| Category | Tool | Purpose |
|----------|------|---------|
| CIS Benchmarks | **OpenSCAP + SCAP Security Guide** | Industry-standard compliance scanning for CIS/STIG |
| Configuration Drift | **Ansible + ansible-cmdb** | Drift detection via scheduled playbook runs with fact caching |
| Package Auditing | **Lynis** | Security auditing and hardening suggestions |
| Vulnerability Scanning | **Trivy** | OS package CVE scanning (also covers containers) |
| Automatic Updates | **dnf-automatic / unattended-upgrades** | Automated security patching |

### 1.2 Central Logging & Monitoring Stack

**Recommended: VictoriaMetrics + Grafana + Loki + Alertmanager** (Lightweight alternative to full ELK/Prometheus)

| Component | Tool | Purpose |
|-----------|------|---------|
| Metrics Collection | **VictoriaMetrics** | Drop-in Prometheus replacement, lower resource usage |
| Log Aggregation | **Grafana Loki** | Lightweight log aggregation (vs ELK) |
| Log Shipper | **Promtail / Vector** | Ship logs to Loki |
| Visualization | **Grafana** | Unified dashboards for metrics and logs |
| Alerting | **Alertmanager** | Alert routing and deduplication |
| Host Metrics | **node_exporter** | System metrics |
| PostgreSQL | **postgres_exporter** | Database metrics |
| Nginx | **nginx-prometheus-exporter** | Web server metrics |

### 1.3 Container Security

| Tool | Purpose |
|------|---------|
| **Trivy** | Dockerfile/image vulnerability scanning |
| **Dockle** | Dockerfile best practices linting |
| **Grype** | Alternative vulnerability scanner |
| **Cosign** | Image signing and verification |

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CENTRAL MONITORING SERVER                         │
│  ┌─────────────────┐  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ VictoriaMetrics │  │ Grafana Loki│  │   Grafana    │  │ Alertmanager  │  │
│  │   (metrics)     │  │   (logs)    │  │ (dashboards) │  │  (alerting)   │  │
│  └────────▲────────┘  └──────▲──────┘  └──────────────┘  └───────────────┘  │
└───────────┼──────────────────┼──────────────────────────────────────────────┘
            │                  │
            │ :8428            │ :3100
            │                  │
┌───────────┴──────────────────┴──────────────────────────────────────────────┐
│                              ALL MANAGED HOSTS                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │  node_exporter  │  │    Promtail     │  │    OpenSCAP / Lynis         │  │
│  │  (metrics)      │  │    (logs)       │  │    (compliance scans)       │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │ postgres_export │  │  nginx_export   │  │    Trivy (vuln scanning)    │  │
│  │ (db servers)    │  │  (app servers)  │  │    (all hosts + CI)         │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Ansible Repository Structure

```
ansible-security/
├── ansible.cfg
├── inventory/
│   ├── production/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   ├── ubuntu.yml
│   │   │   ├── almalinux.yml
│   │   │   ├── db_servers.yml
│   │   │   ├── app_servers.yml
│   │   │   └── build_servers.yml
│   │   └── host_vars/
│   └── staging/
├── playbooks/
│   ├── site.yml                    # Master playbook
│   ├── security-hardening.yml      # Base hardening
│   ├── compliance-scanning.yml     # CIS/OpenSCAP
│   ├── monitoring-agents.yml       # Install exporters
│   ├── logging-agents.yml          # Install Promtail
│   ├── automatic-updates.yml       # Configure auto-updates
│   ├── container-security.yml      # Docker hardening + Trivy
│   └── drift-detection.yml         # Configuration drift checks
├── roles/
│   ├── common/                     # Base packages, users, SSH
│   ├── hardening/                  # Security hardening
│   ├── firewalld/                  # Firewall rules
│   ├── fail2ban/                   # Intrusion prevention
│   ├── openscap/                   # CIS benchmark scanning
│   ├── lynis/                      # Security auditing
│   ├── trivy/                      # Vulnerability scanning
│   ├── node_exporter/              # Prometheus node metrics
│   ├── postgres_exporter/          # PostgreSQL metrics
│   ├── nginx_exporter/             # Nginx metrics
│   ├── promtail/                   # Log shipping
│   ├── automatic_updates/          # dnf-automatic/unattended-upgrades
│   └── docker_security/            # Docker hardening
├── files/
│   ├── openscap/
│   │   └── tailoring/              # Custom CIS profiles
│   └── promtail/
│       └── config.yml
├── templates/
│   ├── alerting/
│   │   └── rules/*.yml
│   └── grafana/
│       └── dashboards/*.json
└── scripts/
    ├── drift-report.sh
    └── compliance-report.sh
```

---

## 4. Implementation Phases

### Phase 1: Foundation (Week 1-2)

#### 4.1 Base Hardening Role

```yaml
# roles/hardening/tasks/main.yml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family | lower }}.yml"

- name: Ensure security packages are installed
  package:
    name: "{{ security_packages }}"
    state: present

# SSH Hardening
- name: Configure SSH hardening
  template:
    src: sshd_config.j2
    dest: /etc/ssh/sshd_config
    owner: root
    group: root
    mode: '0600'
    validate: '/usr/sbin/sshd -t -f %s'
  notify: Restart sshd

- name: Set SSH alive intervals
  blockinfile:
    path: /etc/ssh/sshd_config.d/99-hardening.conf
    create: yes
    block: |
      ClientAliveInterval 300
      ClientAliveCountMax 2
      MaxAuthTries 3
      MaxSessions 4
      LoginGraceTime 60
      PermitEmptyPasswords no
      X11Forwarding no
      AllowAgentForwarding no
      AllowTcpForwarding no

# Kernel hardening via sysctl
- name: Apply kernel security parameters
  sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    sysctl_file: /etc/sysctl.d/99-security.conf
    reload: yes
  loop:
    # Network security
    - { name: 'net.ipv4.conf.all.accept_redirects', value: '0' }
    - { name: 'net.ipv4.conf.default.accept_redirects', value: '0' }
    - { name: 'net.ipv4.conf.all.send_redirects', value: '0' }
    - { name: 'net.ipv4.conf.all.accept_source_route', value: '0' }
    - { name: 'net.ipv4.conf.all.log_martians', value: '1' }
    - { name: 'net.ipv4.icmp_echo_ignore_broadcasts', value: '1' }
    - { name: 'net.ipv4.icmp_ignore_bogus_error_responses', value: '1' }
    - { name: 'net.ipv4.tcp_syncookies', value: '1' }
    - { name: 'net.ipv4.conf.all.rp_filter', value: '1' }
    - { name: 'net.ipv6.conf.all.accept_redirects', value: '0' }
    - { name: 'net.ipv6.conf.default.accept_redirects', value: '0' }
    # Kernel security
    - { name: 'kernel.randomize_va_space', value: '2' }
    - { name: 'kernel.kptr_restrict', value: '2' }
    - { name: 'kernel.dmesg_restrict', value: '1' }
    - { name: 'kernel.perf_event_paranoid', value: '3' }
    - { name: 'kernel.yama.ptrace_scope', value: '2' }
    - { name: 'fs.suid_dumpable', value: '0' }
    - { name: 'fs.protected_hardlinks', value: '1' }
    - { name: 'fs.protected_symlinks', value: '1' }

# File permissions hardening
- name: Set restrictive permissions on sensitive files
  file:
    path: "{{ item.path }}"
    owner: root
    group: root
    mode: "{{ item.mode }}"
  loop:
    - { path: '/etc/passwd', mode: '0644' }
    - { path: '/etc/shadow', mode: '0000' }
    - { path: '/etc/group', mode: '0644' }
    - { path: '/etc/gshadow', mode: '0000' }
    - { path: '/etc/ssh/sshd_config', mode: '0600' }
    - { path: '/boot/grub2', mode: '0700' }
  ignore_errors: yes

# Disable unnecessary services
- name: Disable unnecessary services
  systemd:
    name: "{{ item }}"
    state: stopped
    enabled: no
  loop: "{{ unnecessary_services }}"
  ignore_errors: yes

# Configure auditd
- name: Ensure auditd is installed
  package:
    name: audit
    state: present

- name: Deploy audit rules
  template:
    src: audit.rules.j2
    dest: /etc/audit/rules.d/99-security.rules
  notify: Restart auditd

# Password policies
- name: Configure password quality (pwquality)
  template:
    src: pwquality.conf.j2
    dest: /etc/security/pwquality.conf
    owner: root
    group: root
    mode: '0644'

# Login.defs hardening
- name: Set secure login defaults
  lineinfile:
    path: /etc/login.defs
    regexp: "^{{ item.key }}"
    line: "{{ item.key }} {{ item.value }}"
  loop:
    - { key: 'PASS_MAX_DAYS', value: '365' }
    - { key: 'PASS_MIN_DAYS', value: '1' }
    - { key: 'PASS_WARN_AGE', value: '14' }
    - { key: 'UMASK', value: '027' }
    - { key: 'SHA_CRYPT_MIN_ROUNDS', value: '10000' }

# Ensure AIDE is installed for file integrity monitoring
- name: Install AIDE
  package:
    name: aide
    state: present

- name: Initialize AIDE database (first run only)
  command: aide --init
  args:
    creates: /var/lib/aide/aide.db.new.gz
  register: aide_init

- name: Move AIDE database to production
  command: mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
  when: aide_init.changed

- name: Configure AIDE cron job
  cron:
    name: "AIDE integrity check"
    hour: "5"
    minute: "0"
    job: "/usr/sbin/aide --check | /usr/bin/mail -s 'AIDE Report - {{ inventory_hostname }}' {{ security_email }}"
    user: root
```

```yaml
# roles/hardening/vars/redhat.yml (for AlmaLinux)
---
security_packages:
  - audit
  - aide
  - policycoreutils-python-utils
  - setools-console
  - libselinux-python3

unnecessary_services:
  - cups
  - avahi-daemon
  - rpcbind
  - nfs-server
  - bluetooth
```

```yaml
# roles/hardening/vars/debian.yml (for Ubuntu)
---
security_packages:
  - auditd
  - aide
  - apparmor
  - apparmor-utils
  - libpam-pwquality

unnecessary_services:
  - cups
  - avahi-daemon
  - rpcbind
  - nfs-kernel-server
  - bluetooth
```

---

### Phase 2: Compliance & Vulnerability Scanning (Week 2-3)

#### 4.2 OpenSCAP Role for CIS Benchmarks

```yaml
# roles/openscap/tasks/main.yml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family | lower }}.yml"

- name: Install OpenSCAP packages
  package:
    name: "{{ openscap_packages }}"
    state: present

- name: Create OpenSCAP reports directory
  file:
    path: /var/log/openscap
    state: directory
    mode: '0750'

- name: Download CIS benchmark content (Ubuntu)
  get_url:
    url: "https://security-metadata.canonical.com/oval/com.ubuntu.{{ ansible_distribution_release }}.usn.oval.xml.bz2"
    dest: /usr/share/xml/scap/ubuntu-oval.xml.bz2
    mode: '0644'
  when: ansible_distribution == 'Ubuntu'

- name: Create OpenSCAP scan script
  template:
    src: openscap-scan.sh.j2
    dest: /usr/local/bin/openscap-scan.sh
    mode: '0755'

- name: Configure weekly CIS benchmark scan
  cron:
    name: "OpenSCAP CIS benchmark scan"
    weekday: "0"
    hour: "3"
    minute: "0"
    job: "/usr/local/bin/openscap-scan.sh >> /var/log/openscap/scan.log 2>&1"
    user: root

- name: Create systemd timer for compliance scanning (alternative to cron)
  template:
    src: "{{ item }}.j2"
    dest: "/etc/systemd/system/{{ item }}"
    mode: '0644'
  loop:
    - openscap-scan.service
    - openscap-scan.timer
  notify: Enable openscap timer
```

```bash
# roles/openscap/templates/openscap-scan.sh.j2
#!/bin/bash
# OpenSCAP CIS Benchmark Scanner

REPORT_DIR="/var/log/openscap"
DATE=$(date +%Y%m%d_%H%M%S)
HOSTNAME=$(hostname -f)

{% if ansible_os_family == 'RedHat' %}
# AlmaLinux/RHEL CIS scan
PROFILE="xccdf_org.ssgproject.content_profile_cis_server_l1"
DATASTREAM="/usr/share/xml/scap/ssg/content/ssg-almalinux9-ds.xml"

oscap xccdf eval \
    --profile ${PROFILE} \
    --results ${REPORT_DIR}/results-${DATE}.xml \
    --report ${REPORT_DIR}/report-${DATE}.html \
    --oval-results \
    ${DATASTREAM}

{% elif ansible_distribution == 'Ubuntu' %}
# Ubuntu CIS scan using Ubuntu Security Guide
PROFILE="xccdf_com.ubuntu.noble.cis_profile_Level_1_Server"
DATASTREAM="/usr/share/ubuntu-scap-security-guides/current/benchmarks/Canonical_Ubuntu_24.04_CIS_Benchmarks-xccdf.xml"

# Check if Ubuntu Security Guide is available
if [ -f "${DATASTREAM}" ]; then
    oscap xccdf eval \
        --profile ${PROFILE} \
        --results ${REPORT_DIR}/results-${DATE}.xml \
        --report ${REPORT_DIR}/report-${DATE}.html \
        ${DATASTREAM}
else
    echo "Ubuntu Security Guide not found, using generic scan"
    oscap oval eval \
        --results ${REPORT_DIR}/oval-results-${DATE}.xml \
        --report ${REPORT_DIR}/oval-report-${DATE}.html \
        /usr/share/xml/scap/ubuntu-oval.xml
fi
{% endif %}

# Cleanup old reports (keep last 30 days)
find ${REPORT_DIR} -name "*.html" -mtime +30 -delete
find ${REPORT_DIR} -name "*.xml" -mtime +30 -delete

# Send summary to central logging
SCORE=$(grep -oP 'score.*?>\K[0-9.]+' ${REPORT_DIR}/results-${DATE}.xml | head -1)
logger -t openscap "CIS Benchmark scan completed: score=${SCORE}%"

# Export metrics for Prometheus
echo "openscap_cis_score{host=\"${HOSTNAME}\"} ${SCORE}" > /var/lib/node_exporter/textfile_collector/openscap.prom

echo "Scan completed. Report: ${REPORT_DIR}/report-${DATE}.html"
```

#### 4.3 Lynis Security Auditing Role

```yaml
# roles/lynis/tasks/main.yml
---
- name: Install Lynis
  package:
    name: lynis
    state: present

- name: Create Lynis reports directory
  file:
    path: /var/log/lynis
    state: directory
    mode: '0750'

- name: Deploy custom Lynis profile
  template:
    src: custom.prf.j2
    dest: /etc/lynis/custom.prf
    mode: '0640'

- name: Create Lynis scan script
  template:
    src: lynis-scan.sh.j2
    dest: /usr/local/bin/lynis-scan.sh
    mode: '0755'

- name: Configure daily Lynis audit
  cron:
    name: "Lynis security audit"
    hour: "4"
    minute: "30"
    job: "/usr/local/bin/lynis-scan.sh"
    user: root
```

```bash
# roles/lynis/templates/lynis-scan.sh.j2
#!/bin/bash
# Lynis Security Audit Script

REPORT_DIR="/var/log/lynis"
DATE=$(date +%Y%m%d)
HOSTNAME=$(hostname -f)

# Run Lynis audit
lynis audit system \
    --cronjob \
    --report-file ${REPORT_DIR}/lynis-report-${DATE}.dat \
    --log-file ${REPORT_DIR}/lynis-log-${DATE}.log

# Extract hardening index
HARDENING_INDEX=$(grep "hardening_index" ${REPORT_DIR}/lynis-report-${DATE}.dat | cut -d'=' -f2)

# Export metrics for Prometheus
cat > /var/lib/node_exporter/textfile_collector/lynis.prom << EOF
# HELP lynis_hardening_index Lynis hardening index score
# TYPE lynis_hardening_index gauge
lynis_hardening_index{host="${HOSTNAME}"} ${HARDENING_INDEX}
EOF

# Extract warnings count
WARNINGS=$(grep -c "^warning\[\]" ${REPORT_DIR}/lynis-report-${DATE}.dat || echo 0)
SUGGESTIONS=$(grep -c "^suggestion\[\]" ${REPORT_DIR}/lynis-report-${DATE}.dat || echo 0)

cat >> /var/lib/node_exporter/textfile_collector/lynis.prom << EOF
# HELP lynis_warnings_total Number of Lynis warnings
# TYPE lynis_warnings_total gauge
lynis_warnings_total{host="${HOSTNAME}"} ${WARNINGS}
# HELP lynis_suggestions_total Number of Lynis suggestions
# TYPE lynis_suggestions_total gauge
lynis_suggestions_total{host="${HOSTNAME}"} ${SUGGESTIONS}
EOF

# Log to syslog for Loki
logger -t lynis "Audit completed: hardening_index=${HARDENING_INDEX} warnings=${WARNINGS} suggestions=${SUGGESTIONS}"

# Cleanup old reports
find ${REPORT_DIR} -name "*.dat" -mtime +90 -delete
find ${REPORT_DIR} -name "*.log" -mtime +90 -delete
```

#### 4.4 Trivy Vulnerability Scanner Role

```yaml
# roles/trivy/tasks/main.yml
---
- name: Add Trivy repository (RedHat)
  yum_repository:
    name: trivy
    description: Trivy repository
    baseurl: https://aquasecurity.github.io/trivy-repo/rpm/releases/$basearch/
    gpgcheck: yes
    gpgkey: https://aquasecurity.github.io/trivy-repo/rpm/public.key
    enabled: yes
  when: ansible_os_family == 'RedHat'

- name: Add Trivy repository (Debian)
  block:
    - name: Add Trivy GPG key
      apt_key:
        url: https://aquasecurity.github.io/trivy-repo/deb/public.key
        state: present

    - name: Add Trivy apt repository
      apt_repository:
        repo: "deb https://aquasecurity.github.io/trivy-repo/deb {{ ansible_distribution_release }} main"
        state: present
  when: ansible_os_family == 'Debian'

- name: Install Trivy
  package:
    name: trivy
    state: present

- name: Create Trivy cache directory
  file:
    path: /var/cache/trivy
    state: directory
    mode: '0755'

- name: Create Trivy scan script
  template:
    src: trivy-scan.sh.j2
    dest: /usr/local/bin/trivy-scan.sh
    mode: '0755'

- name: Configure daily Trivy filesystem scan
  cron:
    name: "Trivy filesystem vulnerability scan"
    hour: "2"
    minute: "0"
    job: "/usr/local/bin/trivy-scan.sh filesystem >> /var/log/trivy/scan.log 2>&1"
    user: root

- name: Configure daily Trivy Docker scan (if Docker installed)
  cron:
    name: "Trivy Docker image scan"
    hour: "2"
    minute: "30"
    job: "/usr/local/bin/trivy-scan.sh docker >> /var/log/trivy/docker-scan.log 2>&1"
    user: root
  when: "'docker' in ansible_facts.packages"
```

```bash
# roles/trivy/templates/trivy-scan.sh.j2
#!/bin/bash
# Trivy Vulnerability Scanner

REPORT_DIR="/var/log/trivy"
DATE=$(date +%Y%m%d_%H%M%S)
HOSTNAME=$(hostname -f)
SCAN_TYPE=${1:-filesystem}

mkdir -p ${REPORT_DIR}

case ${SCAN_TYPE} in
    filesystem)
        echo "Running filesystem vulnerability scan..."
        trivy filesystem \
            --severity HIGH,CRITICAL \
            --format json \
            --output ${REPORT_DIR}/fs-scan-${DATE}.json \
            / 2>/dev/null

        # Count vulnerabilities
        CRITICAL=$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' ${REPORT_DIR}/fs-scan-${DATE}.json 2>/dev/null || echo 0)
        HIGH=$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="HIGH")] | length' ${REPORT_DIR}/fs-scan-${DATE}.json 2>/dev/null || echo 0)

        # Export to Prometheus
        cat > /var/lib/node_exporter/textfile_collector/trivy_fs.prom << EOF
# HELP trivy_filesystem_vulnerabilities_total Filesystem vulnerabilities by severity
# TYPE trivy_filesystem_vulnerabilities_total gauge
trivy_filesystem_vulnerabilities_total{host="${HOSTNAME}",severity="critical"} ${CRITICAL}
trivy_filesystem_vulnerabilities_total{host="${HOSTNAME}",severity="high"} ${HIGH}
EOF
        logger -t trivy "Filesystem scan: critical=${CRITICAL} high=${HIGH}"
        ;;

    docker)
        echo "Scanning Docker images..."
        
        # Get list of local images
        IMAGES=$(docker images --format "{{.Repository}}:{{.Tag}}" | grep -v "<none>")
        
        TOTAL_CRITICAL=0
        TOTAL_HIGH=0
        
        for IMAGE in ${IMAGES}; do
            IMAGE_NAME=$(echo ${IMAGE} | tr '/:' '_')
            trivy image \
                --severity HIGH,CRITICAL \
                --format json \
                --output ${REPORT_DIR}/docker-${IMAGE_NAME}-${DATE}.json \
                ${IMAGE} 2>/dev/null
            
            CRIT=$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' ${REPORT_DIR}/docker-${IMAGE_NAME}-${DATE}.json 2>/dev/null || echo 0)
            HIGH=$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="HIGH")] | length' ${REPORT_DIR}/docker-${IMAGE_NAME}-${DATE}.json 2>/dev/null || echo 0)
            
            TOTAL_CRITICAL=$((TOTAL_CRITICAL + CRIT))
            TOTAL_HIGH=$((TOTAL_HIGH + HIGH))
            
            logger -t trivy "Image ${IMAGE}: critical=${CRIT} high=${HIGH}"
        done
        
        cat > /var/lib/node_exporter/textfile_collector/trivy_docker.prom << EOF
# HELP trivy_docker_vulnerabilities_total Docker image vulnerabilities by severity
# TYPE trivy_docker_vulnerabilities_total gauge
trivy_docker_vulnerabilities_total{host="${HOSTNAME}",severity="critical"} ${TOTAL_CRITICAL}
trivy_docker_vulnerabilities_total{host="${HOSTNAME}",severity="high"} ${TOTAL_HIGH}
EOF
        ;;
esac

# Cleanup old reports (keep 7 days)
find ${REPORT_DIR} -name "*.json" -mtime +7 -delete
```

---

### Phase 3: Automatic Updates (Week 3)

#### 4.5 Automatic Updates Role

```yaml
# roles/automatic_updates/tasks/main.yml
---
- name: Include OS-specific tasks
  include_tasks: "{{ ansible_os_family | lower }}.yml"
```

```yaml
# roles/automatic_updates/tasks/redhat.yml
---
- name: Install dnf-automatic
  dnf:
    name: dnf-automatic
    state: present

- name: Configure dnf-automatic for security updates
  template:
    src: automatic.conf.j2
    dest: /etc/dnf/automatic.conf
    mode: '0644'

- name: Enable and start dnf-automatic timer
  systemd:
    name: dnf-automatic-install.timer
    state: started
    enabled: yes
```

```yaml
# roles/automatic_updates/tasks/debian.yml
---
- name: Install unattended-upgrades
  apt:
    name:
      - unattended-upgrades
      - apt-listchanges
    state: present

- name: Configure unattended-upgrades
  template:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    mode: '0644'
  loop:
    - { src: '50unattended-upgrades.j2', dest: '/etc/apt/apt.conf.d/50unattended-upgrades' }
    - { src: '20auto-upgrades.j2', dest: '/etc/apt/apt.conf.d/20auto-upgrades' }

- name: Enable unattended-upgrades service
  systemd:
    name: unattended-upgrades
    state: started
    enabled: yes
```

```ini
# roles/automatic_updates/templates/automatic.conf.j2 (AlmaLinux)
[commands]
upgrade_type = security
random_sleep = 360
download_updates = yes
apply_updates = yes

[emitters]
emit_via = stdio,email

[email]
email_from = root@{{ inventory_hostname }}
email_to = {{ security_email }}
email_host = {{ smtp_server | default('localhost') }}

[command_email]
email_from = root@{{ inventory_hostname }}
email_to = {{ security_email }}
```

```
# roles/automatic_updates/templates/50unattended-upgrades.j2 (Ubuntu)
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
};

Unattended-Upgrade::Package-Blacklist {
    // Add packages that should not be auto-updated
};

Unattended-Upgrade::DevRelease "auto";
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "{{ auto_reboot | default('false') }}";
Unattended-Upgrade::Automatic-Reboot-Time "{{ auto_reboot_time | default('02:00') }}";
Unattended-Upgrade::Mail "{{ security_email }}";
Unattended-Upgrade::MailReport "only-on-error";
Unattended-Upgrade::SyslogEnable "true";
Unattended-Upgrade::SyslogFacility "daemon";
```

---

### Phase 4: Central Logging & Monitoring (Week 3-4)

#### 4.6 Promtail Log Shipper Role

```yaml
# roles/promtail/tasks/main.yml
---
- name: Create promtail user
  user:
    name: promtail
    system: yes
    shell: /usr/sbin/nologin
    home: /var/lib/promtail
    create_home: yes

- name: Download Promtail binary
  get_url:
    url: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-linux-amd64.zip"
    dest: /tmp/promtail.zip
    mode: '0644'

- name: Extract Promtail
  unarchive:
    src: /tmp/promtail.zip
    dest: /usr/local/bin/
    remote_src: yes
    mode: '0755'

- name: Create Promtail config directory
  file:
    path: /etc/promtail
    state: directory
    mode: '0755'

- name: Deploy Promtail configuration
  template:
    src: promtail.yml.j2
    dest: /etc/promtail/promtail.yml
    mode: '0644'
  notify: Restart promtail

- name: Create Promtail systemd service
  template:
    src: promtail.service.j2
    dest: /etc/systemd/system/promtail.service
    mode: '0644'
  notify: 
    - Reload systemd
    - Restart promtail

- name: Add promtail to required groups for log access
  user:
    name: promtail
    groups: 
      - adm
      - systemd-journal
    append: yes

- name: Start and enable Promtail
  systemd:
    name: promtail
    state: started
    enabled: yes
```

```yaml
# roles/promtail/templates/promtail.yml.j2
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://{{ loki_server }}:3100/loki/api/v1/push
    tenant_id: {{ tenant_id | default('default') }}

scrape_configs:
  # System logs
  - job_name: syslog
    static_configs:
      - targets:
          - localhost
        labels:
          job: syslog
          host: {{ inventory_hostname }}
          env: {{ environment | default('production') }}
          __path__: /var/log/syslog
    pipeline_stages:
      - regex:
          expression: '^(?P<timestamp>\w+\s+\d+\s+\d+:\d+:\d+)\s+(?P<hostname>\S+)\s+(?P<app>\S+?)(\[(?P<pid>\d+)\])?:\s+(?P<message>.*)$'
      - labels:
          app:
      - timestamp:
          source: timestamp
          format: "Jan 2 15:04:05"

  # Auth logs (SSH, sudo, etc.)
  - job_name: auth
    static_configs:
      - targets:
          - localhost
        labels:
          job: auth
          host: {{ inventory_hostname }}
          __path__: /var/log/auth.log
    pipeline_stages:
      - match:
          selector: '{job="auth"}'
          stages:
            - regex:
                expression: '(?P<action>Failed|Accepted|Invalid|session opened|session closed)'
            - labels:
                action:

  # Audit logs
  - job_name: audit
    static_configs:
      - targets:
          - localhost
        labels:
          job: audit
          host: {{ inventory_hostname }}
          __path__: /var/log/audit/audit.log

  # Firewall logs (firewalld)
  - job_name: firewall
    static_configs:
      - targets:
          - localhost
        labels:
          job: firewall
          host: {{ inventory_hostname }}
          __path__: /var/log/firewalld
    pipeline_stages:
      - regex:
          expression: '.*?(?P<action>ACCEPT|DROP|REJECT).*'
      - labels:
          action:

  # Journal (systemd)
  - job_name: journal
    journal:
      max_age: 12h
      labels:
        job: systemd-journal
        host: {{ inventory_hostname }}
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'
      - source_labels: ['__journal__hostname']
        target_label: 'hostname'
      - source_labels: ['__journal_priority_keyword']
        target_label: 'level'

{% if 'app_servers' in group_names %}
  # Nginx access logs
  - job_name: nginx_access
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx_access
          host: {{ inventory_hostname }}
          __path__: /var/log/nginx/access.log
    pipeline_stages:
      - regex:
          expression: '^(?P<remote_addr>\S+) - (?P<remote_user>\S+) \[(?P<time_local>[^\]]+)\] "(?P<method>\S+) (?P<request_uri>\S+) (?P<protocol>\S+)" (?P<status>\d+) (?P<body_bytes_sent>\d+) "(?P<http_referer>[^"]*)" "(?P<http_user_agent>[^"]*)"'
      - labels:
          method:
          status:
      - metrics:
          http_requests_total:
            type: Counter
            description: "Total HTTP requests"
            source: status
            config:
              action: inc

  # Nginx error logs
  - job_name: nginx_error
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx_error
          host: {{ inventory_hostname }}
          __path__: /var/log/nginx/error.log
{% endif %}

{% if 'db_servers' in group_names %}
  # PostgreSQL logs
  - job_name: postgresql
    static_configs:
      - targets:
          - localhost
        labels:
          job: postgresql
          host: {{ inventory_hostname }}
          __path__: /var/log/postgresql/*.log
    pipeline_stages:
      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'
      - regex:
          expression: '^(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d+) \w+ \[(?P<pid>\d+)\] (?P<level>\w+):'
      - labels:
          level:

  # pgAudit logs (if enabled)
  - job_name: pgaudit
    static_configs:
      - targets:
          - localhost
        labels:
          job: pgaudit
          host: {{ inventory_hostname }}
          __path__: /var/log/postgresql/pgaudit.log
{% endif %}

{% if 'build_servers' in group_names %}
  # Docker container logs
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: 'service'
{% endif %}

  # Security scan reports
  - job_name: security_scans
    static_configs:
      - targets:
          - localhost
        labels:
          job: security_scans
          host: {{ inventory_hostname }}
          __path__: /var/log/{openscap,lynis,trivy}/*.log
```

#### 4.7 Node Exporter Role

```yaml
# roles/node_exporter/tasks/main.yml
---
- name: Create node_exporter user
  user:
    name: node_exporter
    system: yes
    shell: /usr/sbin/nologin

- name: Create textfile collector directory
  file:
    path: /var/lib/node_exporter/textfile_collector
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Download node_exporter
  get_url:
    url: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
    dest: /tmp/node_exporter.tar.gz
    mode: '0644'

- name: Extract node_exporter
  unarchive:
    src: /tmp/node_exporter.tar.gz
    dest: /tmp/
    remote_src: yes

- name: Install node_exporter binary
  copy:
    src: "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/node_exporter"
    dest: /usr/local/bin/node_exporter
    mode: '0755'
    remote_src: yes

- name: Create node_exporter systemd service
  template:
    src: node_exporter.service.j2
    dest: /etc/systemd/system/node_exporter.service
    mode: '0644'
  notify:
    - Reload systemd
    - Restart node_exporter

- name: Configure firewalld for node_exporter (internal only)
  firewalld:
    rich_rule: 'rule family="ipv4" source address="{{ monitoring_server_ip }}" port port="9100" protocol="tcp" accept'
    permanent: yes
    immediate: yes
    state: enabled

- name: Start and enable node_exporter
  systemd:
    name: node_exporter
    state: started
    enabled: yes
```

```ini
# roles/node_exporter/templates/node_exporter.service.j2
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
    --collector.textfile.directory=/var/lib/node_exporter/textfile_collector \
    --collector.systemd \
    --collector.processes \
    --web.listen-address=:9100

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### 4.8 PostgreSQL Exporter Role (for DB servers)

```yaml
# roles/postgres_exporter/tasks/main.yml
---
- name: Download postgres_exporter
  get_url:
    url: "https://github.com/prometheus-community/postgres_exporter/releases/download/v{{ postgres_exporter_version }}/postgres_exporter-{{ postgres_exporter_version }}.linux-amd64.tar.gz"
    dest: /tmp/postgres_exporter.tar.gz
    mode: '0644'

- name: Extract postgres_exporter
  unarchive:
    src: /tmp/postgres_exporter.tar.gz
    dest: /tmp/
    remote_src: yes

- name: Install postgres_exporter binary
  copy:
    src: "/tmp/postgres_exporter-{{ postgres_exporter_version }}.linux-amd64/postgres_exporter"
    dest: /usr/local/bin/postgres_exporter
    mode: '0755'
    remote_src: yes

- name: Create postgres_exporter user
  user:
    name: postgres_exporter
    system: yes
    shell: /usr/sbin/nologin

- name: Create PostgreSQL monitoring user
  become_user: postgres
  postgresql_user:
    name: "{{ postgres_exporter_user }}"
    password: "{{ postgres_exporter_password }}"
    role_attr_flags: "NOSUPERUSER,NOCREATEDB,NOCREATEROLE"
  
- name: Grant monitoring permissions
  become_user: postgres
  postgresql_query:
    query: |
      GRANT pg_monitor TO {{ postgres_exporter_user }};
      GRANT CONNECT ON DATABASE postgres TO {{ postgres_exporter_user }};

- name: Create environment file for postgres_exporter
  template:
    src: postgres_exporter.env.j2
    dest: /etc/default/postgres_exporter
    mode: '0600'
    owner: postgres_exporter

- name: Create postgres_exporter custom queries
  template:
    src: queries.yaml.j2
    dest: /etc/postgres_exporter/queries.yaml
    mode: '0644'

- name: Create postgres_exporter systemd service
  template:
    src: postgres_exporter.service.j2
    dest: /etc/systemd/system/postgres_exporter.service
    mode: '0644'
  notify:
    - Reload systemd
    - Restart postgres_exporter

- name: Start and enable postgres_exporter
  systemd:
    name: postgres_exporter
    state: started
    enabled: yes
```

```yaml
# roles/postgres_exporter/templates/queries.yaml.j2
# Custom queries for PostgreSQL monitoring
pg_replication:
  query: "SELECT CASE WHEN NOT pg_is_in_recovery() THEN 0 ELSE GREATEST(0, EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))) END AS lag"
  master: true
  metrics:
    - lag:
        usage: "GAUGE"
        description: "Replication lag behind master in seconds"

pg_postmaster:
  query: "SELECT pg_postmaster_start_time as start_time_seconds from pg_postmaster_start_time()"
  master: true
  metrics:
    - start_time_seconds:
        usage: "GAUGE"
        description: "Time at which postmaster started"

pg_database_size:
  query: "SELECT pg_database.datname, pg_database_size(pg_database.datname) as bytes FROM pg_database"
  master: true
  metrics:
    - datname:
        usage: "LABEL"
        description: "Database name"
    - bytes:
        usage: "GAUGE"
        description: "Database size in bytes"

pg_stat_activity_count:
  query: |
    SELECT datname, state, count(*) as count
    FROM pg_stat_activity
    WHERE pid <> pg_backend_pid()
    GROUP BY datname, state
  metrics:
    - datname:
        usage: "LABEL"
        description: "Database name"
    - state:
        usage: "LABEL"
        description: "Connection state"
    - count:
        usage: "GAUGE"
        description: "Number of connections"

pg_locks:
  query: |
    SELECT pg_database.datname, tmp.mode, COALESCE(count, 0) as count
    FROM (VALUES ('accesssharelock'),('rowsharelock'),('rowexclusivelock'),
                 ('shareupdateexclusivelock'),('sharelock'),('sharerowexclusivelock'),
                 ('exclusivelock'),('accessexclusivelock')) AS tmp(mode)
    CROSS JOIN pg_database
    LEFT JOIN (SELECT database, lower(mode) AS mode, count(*) AS count
               FROM pg_locks WHERE database IS NOT NULL
               GROUP BY database, mode) AS tmp2
    ON tmp.mode = tmp2.mode AND pg_database.oid = tmp2.database
  metrics:
    - datname:
        usage: "LABEL"
        description: "Database name"
    - mode:
        usage: "LABEL"
        description: "Lock mode"
    - count:
        usage: "GAUGE"
        description: "Number of locks"
```

---

### Phase 5: Central Monitoring Server (Week 4-5)

#### 4.9 Monitoring Stack Playbook

```yaml
# playbooks/monitoring-stack.yml
---
- name: Deploy Central Monitoring Stack
  hosts: monitoring_server
  become: yes
  vars:
    victoria_metrics_version: "1.96.0"
    grafana_version: "10.2.3"
    loki_version: "2.9.3"
    alertmanager_version: "0.26.0"
  
  tasks:
    # VictoriaMetrics (Prometheus alternative)
    - name: Create VictoriaMetrics directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /opt/victoriametrics
        - /var/lib/victoriametrics

    - name: Download VictoriaMetrics
      get_url:
        url: "https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v{{ victoria_metrics_version }}/victoria-metrics-linux-amd64-v{{ victoria_metrics_version }}.tar.gz"
        dest: /tmp/victoriametrics.tar.gz

    - name: Extract VictoriaMetrics
      unarchive:
        src: /tmp/victoriametrics.tar.gz
        dest: /opt/victoriametrics
        remote_src: yes

    - name: Create VictoriaMetrics systemd service
      copy:
        dest: /etc/systemd/system/victoriametrics.service
        content: |
          [Unit]
          Description=VictoriaMetrics
          After=network.target

          [Service]
          Type=simple
          ExecStart=/opt/victoriametrics/victoria-metrics-prod \
            -storageDataPath=/var/lib/victoriametrics \
            -httpListenAddr=:8428 \
            -retentionPeriod=90d \
            -promscrape.config=/etc/victoriametrics/scrape.yml
          Restart=always

          [Install]
          WantedBy=multi-user.target
      notify: Restart victoriametrics

    - name: Deploy VictoriaMetrics scrape config
      template:
        src: templates/victoriametrics/scrape.yml.j2
        dest: /etc/victoriametrics/scrape.yml
      notify: Restart victoriametrics

    # Grafana Loki
    - name: Download Loki
      get_url:
        url: "https://github.com/grafana/loki/releases/download/v{{ loki_version }}/loki-linux-amd64.zip"
        dest: /tmp/loki.zip

    - name: Extract Loki
      unarchive:
        src: /tmp/loki.zip
        dest: /usr/local/bin/
        remote_src: yes
        mode: '0755'

    - name: Create Loki config
      template:
        src: templates/loki/loki-config.yml.j2
        dest: /etc/loki/loki-config.yml
      notify: Restart loki

    - name: Create Loki systemd service
      copy:
        dest: /etc/systemd/system/loki.service
        content: |
          [Unit]
          Description=Loki log aggregation system
          After=network.target

          [Service]
          Type=simple
          ExecStart=/usr/local/bin/loki-linux-amd64 -config.file=/etc/loki/loki-config.yml
          Restart=always

          [Install]
          WantedBy=multi-user.target
      notify: Restart loki

    # Alertmanager
    - name: Download Alertmanager
      get_url:
        url: "https://github.com/prometheus/alertmanager/releases/download/v{{ alertmanager_version }}/alertmanager-{{ alertmanager_version }}.linux-amd64.tar.gz"
        dest: /tmp/alertmanager.tar.gz

    - name: Deploy Alertmanager configuration
      template:
        src: templates/alertmanager/alertmanager.yml.j2
        dest: /etc/alertmanager/alertmanager.yml
      notify: Restart alertmanager

    # Grafana
    - name: Add Grafana repository
      yum_repository:
        name: grafana
        description: Grafana repository
        baseurl: https://rpm.grafana.com
        gpgcheck: yes
        gpgkey: https://rpm.grafana.com/gpg.key
        enabled: yes
      when: ansible_os_family == 'RedHat'

    - name: Install Grafana
      package:
        name: grafana
        state: present

    - name: Deploy Grafana datasources
      template:
        src: templates/grafana/datasources.yml.j2
        dest: /etc/grafana/provisioning/datasources/datasources.yml
      notify: Restart grafana

    - name: Deploy Grafana dashboards
      copy:
        src: "files/grafana/dashboards/"
        dest: /etc/grafana/provisioning/dashboards/
      notify: Restart grafana

    - name: Start all monitoring services
      systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - victoriametrics
        - loki
        - alertmanager
        - grafana-server

  handlers:
    - name: Restart victoriametrics
      systemd:
        name: victoriametrics
        state: restarted

    - name: Restart loki
      systemd:
        name: loki
        state: restarted

    - name: Restart alertmanager
      systemd:
        name: alertmanager
        state: restarted

    - name: Restart grafana
      systemd:
        name: grafana-server
        state: restarted
```

```yaml
# templates/victoriametrics/scrape.yml.j2
global:
  scrape_interval: 30s
  evaluation_interval: 30s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
{% for host in groups['all'] %}
      - targets: ['{{ hostvars[host].ansible_host }}:9100']
        labels:
          host: '{{ host }}'
          env: '{{ hostvars[host].environment | default("production") }}'
{% endfor %}

  - job_name: 'postgres_exporter'
    static_configs:
{% for host in groups['db_servers'] %}
      - targets: ['{{ hostvars[host].ansible_host }}:9187']
        labels:
          host: '{{ host }}'
{% endfor %}

  - job_name: 'nginx_exporter'
    static_configs:
{% for host in groups['app_servers'] %}
      - targets: ['{{ hostvars[host].ansible_host }}:9113']
        labels:
          host: '{{ host }}'
{% endfor %}
```

---

### Phase 6: Alerting Rules (Week 5)

#### 4.10 Security Alert Rules

```yaml
# templates/alertmanager/security_rules.yml
groups:
  - name: security_alerts
    rules:
      # Failed SSH attempts
      - alert: HighSSHFailedLogins
        expr: increase(node_systemd_unit_state{name="sshd.service"}[5m]) > 0 and on() increase(syslog_ssh_failed_total[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High number of failed SSH logins on {{ $labels.host }}"
          description: "More than 10 failed SSH login attempts in the last 5 minutes"

      # Compliance score drop
      - alert: LowCISComplianceScore
        expr: openscap_cis_score < 70
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "CIS compliance score below 70% on {{ $labels.host }}"
          description: "Current score: {{ $value }}%"

      # Lynis hardening index
      - alert: LowHardeningIndex
        expr: lynis_hardening_index < 65
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Lynis hardening index below 65 on {{ $labels.host }}"
          description: "Current index: {{ $value }}"

      # Critical vulnerabilities
      - alert: CriticalVulnerabilities
        expr: trivy_filesystem_vulnerabilities_total{severity="critical"} > 0
        for: 30m
        labels:
          severity: critical
        annotations:
          summary: "Critical vulnerabilities found on {{ $labels.host }}"
          description: "{{ $value }} critical vulnerabilities detected"

      # Docker vulnerabilities
      - alert: DockerImageVulnerabilities
        expr: trivy_docker_vulnerabilities_total{severity="critical"} > 5
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Multiple critical Docker image vulnerabilities on {{ $labels.host }}"
          description: "{{ $value }} critical vulnerabilities in Docker images"

      # Disk space (important for log retention)
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100 < 15
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.host }}"
          description: "{{ $labels.mountpoint }} is {{ $value | printf \"%.1f\" }}% free"

      # Service down
      - alert: CriticalServiceDown
        expr: node_systemd_unit_state{name=~"sshd.service|firewalld.service|auditd.service",state="active"} != 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Critical service {{ $labels.name }} down on {{ $labels.host }}"

      # PostgreSQL connections
      - alert: PostgreSQLTooManyConnections
        expr: pg_stat_activity_count{state="active"} > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High number of PostgreSQL connections"

      # PostgreSQL replication lag
      - alert: PostgreSQLReplicationLag
        expr: pg_replication_lag > 300
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL replication lag > 5 minutes"

      # File integrity (AIDE)
      - alert: FileIntegrityChanged
        expr: increase(aide_changed_files_total[24h]) > 0
        labels:
          severity: warning
        annotations:
          summary: "File integrity changes detected on {{ $labels.host }}"

      # Fail2ban bans
      - alert: HighFail2banBans
        expr: increase(fail2ban_banned_total[1h]) > 50
        labels:
          severity: warning
        annotations:
          summary: "High number of fail2ban bans on {{ $labels.host }}"
```

---

## 5. Container Security

### 5.1 Docker Hardening Role

```yaml
# roles/docker_security/tasks/main.yml
---
- name: Ensure Docker daemon configuration directory exists
  file:
    path: /etc/docker
    state: directory
    mode: '0755'

- name: Configure Docker daemon security settings
  template:
    src: daemon.json.j2
    dest: /etc/docker/daemon.json
    mode: '0644'
  notify: Restart docker

- name: Ensure Docker socket permissions
  file:
    path: /var/run/docker.sock
    owner: root
    group: docker
    mode: '0660'

- name: Deploy Docker Bench Security script
  get_url:
    url: https://raw.githubusercontent.com/docker/docker-bench-security/master/docker-bench-security.sh
    dest: /usr/local/bin/docker-bench-security.sh
    mode: '0755'

- name: Create Docker security audit cron
  cron:
    name: "Docker security audit"
    hour: "3"
    minute: "30"
    weekday: "0"
    job: "/usr/local/bin/docker-bench-security.sh -l /var/log/docker-bench/audit-$(date +\\%Y\\%m\\%d).log"

- name: Install Trivy for container scanning
  include_role:
    name: trivy

- name: Install Dockle for Dockerfile linting
  get_url:
    url: "https://github.com/goodwithtech/dockle/releases/download/v{{ dockle_version }}/dockle_{{ dockle_version }}_Linux-64bit.tar.gz"
    dest: /tmp/dockle.tar.gz

- name: Extract Dockle
  unarchive:
    src: /tmp/dockle.tar.gz
    dest: /usr/local/bin/
    remote_src: yes
    mode: '0755'
```

```json
// roles/docker_security/templates/daemon.json.j2
{
  "icc": false,
  "userns-remap": "default",
  "no-new-privileges": true,
  "live-restore": true,
  "userland-proxy": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "storage-driver": "overlay2",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65535,
      "Soft": 65535
    },
    "nproc": {
      "Name": "nproc",
      "Hard": 4096,
      "Soft": 4096
    }
  }
}
```

### 5.2 CI Pipeline Container Scanning

```yaml
# .github/workflows/container-security.yml
name: Container Security Scan

on:
  push:
    paths:
      - '**/Dockerfile'
      - '**/docker-compose*.yml'
  pull_request:
    paths:
      - '**/Dockerfile'

jobs:
  dockerfile-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Dockle (Dockerfile best practices)
        uses: goodwithtech/dockle-action@v0.1.2
        with:
          image: ${{ github.repository }}:${{ github.sha }}
          format: 'list'
          exit-code: '1'
          exit-level: 'warn'
          ignore: 'CIS-DI-0001'  # Example: ignore specific rules

  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image for scanning
        run: docker build -t ${{ github.repository }}:${{ github.sha }} .
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '${{ github.repository }}:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          
      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  # Scan IaC files (docker-compose, etc.)
  config-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy config scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: '.'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
```

```yaml
# .gitlab-ci.yml (GitLab CI example)
stages:
  - build
  - security
  - deploy

container_scanning:
  stage: security
  image: docker:latest
  services:
    - docker:dind
  variables:
    TRIVY_NO_PROGRESS: "true"
    TRIVY_CACHE_DIR: ".trivycache/"
  before_script:
    - apk add --no-cache curl
    - curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - trivy image --exit-code 0 --severity HIGH --no-progress $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - trivy image --exit-code 1 --severity CRITICAL --no-progress $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  cache:
    paths:
      - .trivycache/
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
```

---

## 6. Configuration Drift Detection

### 6.1 Drift Detection Playbook

```yaml
# playbooks/drift-detection.yml
---
- name: Configuration Drift Detection
  hosts: all
  gather_facts: yes
  
  vars:
    drift_report_dir: "/var/log/ansible-drift"
    baseline_dir: "/etc/ansible/baselines"
    
  tasks:
    - name: Ensure drift report directory exists
      file:
        path: "{{ drift_report_dir }}"
        state: directory
        mode: '0755'
      delegate_to: localhost
      run_once: true

    - name: Capture current system state
      setup:
        gather_subset:
          - all
      register: current_facts

    - name: Load baseline configuration
      include_vars:
        file: "{{ baseline_dir }}/{{ inventory_hostname }}.yml"
        name: baseline
      ignore_errors: yes
      register: baseline_load

    - name: Create baseline if not exists
      copy:
        content: "{{ current_facts.ansible_facts | to_nice_yaml }}"
        dest: "{{ baseline_dir }}/{{ inventory_hostname }}.yml"
      delegate_to: localhost
      when: baseline_load.failed

    - name: Compare critical configurations
      block:
        - name: Check SSH configuration
          shell: |
            diff -u /etc/ssh/sshd_config.baseline /etc/ssh/sshd_config || true
          register: ssh_drift
          changed_when: ssh_drift.stdout != ""

        - name: Check sysctl settings
          shell: |
            sysctl -a 2>/dev/null | sort > /tmp/sysctl_current
            diff -u /etc/sysctl.baseline /tmp/sysctl_current || true
          register: sysctl_drift
          changed_when: sysctl_drift.stdout != ""

        - name: Check installed packages for unexpected changes
          package_facts:
            manager: auto
          
        - name: Compare package versions
          set_fact:
            package_drift: "{{ ansible_facts.packages | difference(baseline.packages | default({})) }}"
          when: baseline.packages is defined

        - name: Check firewall rules
          shell: firewall-cmd --list-all-zones
          register: firewall_current
          changed_when: false

        - name: Check running services
          shell: systemctl list-units --type=service --state=running --no-pager
          register: services_current
          changed_when: false

    - name: Generate drift report
      template:
        src: drift-report.md.j2
        dest: "{{ drift_report_dir }}/{{ inventory_hostname }}-{{ ansible_date_time.date }}.md"
      delegate_to: localhost

    - name: Alert on critical drift
      uri:
        url: "{{ alertmanager_webhook }}"
        method: POST
        body_format: json
        body:
          alerts:
            - labels:
                alertname: ConfigurationDrift
                host: "{{ inventory_hostname }}"
                severity: warning
              annotations:
                summary: "Configuration drift detected"
                description: |
                  SSH drift: {{ 'yes' if ssh_drift.changed else 'no' }}
                  Sysctl drift: {{ 'yes' if sysctl_drift.changed else 'no' }}
      when: ssh_drift.changed or sysctl_drift.changed
      delegate_to: localhost
```

---

## 7. Additional Security Areas

### 7.1 Areas to Focus On

| Area | Recommendation | Priority |
|------|----------------|----------|
| **Secrets Management** | Implement HashiCorp Vault or AWS Secrets Manager | High |
| **Network Segmentation** | Use firewalld zones, consider WireGuard for internal traffic | High |
| **Certificate Management** | Automate with cert-manager or Certbot | High |
| **Backup Security** | Encrypt backups, test restoration, off-site copies | High |
| **API Security** | Implement rate limiting, WAF (ModSecurity/NGINX) | Medium |
| **DNS Security** | DNSSEC, DNS-over-HTTPS for internal resolution | Medium |
| **Log Retention Policy** | Define retention periods, secure archival | Medium |
| **Incident Response** | Document runbooks, automate common responses | Medium |
| **Access Control** | Implement RBAC, audit admin access | High |
| **Build Server Isolation** | Ephemeral runners, network isolation | High |

### 7.2 Build Server Specific Hardening

```yaml
# roles/build_server_hardening/tasks/main.yml
---
# For GitHub Actions runners, Azure Pipelines agents, GitLab runners

- name: Create dedicated runner user with limited permissions
  user:
    name: "{{ runner_user }}"
    shell: /bin/bash
    groups: docker
    create_home: yes

- name: Configure resource limits for runner
  template:
    src: limits.conf.j2
    dest: /etc/security/limits.d/90-runner.conf

- name: Enable process accounting
  package:
    name: "{{ 'psacct' if ansible_os_family == 'RedHat' else 'acct' }}"
    state: present

- name: Start process accounting
  service:
    name: "{{ 'psacct' if ansible_os_family == 'RedHat' else 'acct' }}"
    state: started
    enabled: yes

- name: Configure auditd rules for build activity
  blockinfile:
    path: /etc/audit/rules.d/99-build.rules
    create: yes
    block: |
      # Monitor runner home directory
      -w /home/{{ runner_user }} -p wa -k build_activity
      # Monitor Docker socket access
      -w /var/run/docker.sock -p rwxa -k docker_access
      # Monitor sensitive file access
      -w /etc/shadow -p r -k shadow_read
      -w /etc/passwd -p wa -k passwd_changes
  notify: Restart auditd

- name: Restrict network access for build jobs (optional)
  firewalld:
    rich_rule: "rule family='ipv4' source address='127.0.0.1' service name='docker' accept"
    permanent: yes
    immediate: yes
    state: enabled
  when: restrict_build_network | default(false)

- name: Configure Docker with rootless mode (optional)
  include_tasks: docker-rootless.yml
  when: use_rootless_docker | default(false)
```

### 7.3 PostgreSQL Security Hardening

```yaml
# roles/postgres_hardening/tasks/main.yml
---
- name: Configure pg_hba.conf for secure authentication
  template:
    src: pg_hba.conf.j2
    dest: "{{ postgresql_conf_dir }}/pg_hba.conf"
    owner: postgres
    group: postgres
    mode: '0600'
  notify: Reload postgresql

- name: Configure PostgreSQL security settings
  postgresql_set:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
  loop:
    - { name: 'ssl', value: 'on' }
    - { name: 'ssl_min_protocol_version', value: 'TLSv1.3' }
    - { name: 'password_encryption', value: 'scram-sha-256' }
    - { name: 'log_connections', value: 'on' }
    - { name: 'log_disconnections', value: 'on' }
    - { name: 'log_statement', value: 'ddl' }
    - { name: 'log_min_duration_statement', value: '1000' }
  become_user: postgres
  notify: Reload postgresql

- name: Ensure pgaudit is configured
  postgresql_set:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
  loop:
    - { name: 'pgaudit.log', value: 'ddl, role, misc_set' }
    - { name: 'pgaudit.log_catalog', value: 'off' }
    - { name: 'pgaudit.log_client', value: 'on' }
    - { name: 'pgaudit.log_level', value: 'log' }
  become_user: postgres
  notify: Reload postgresql

- name: Create read-only monitoring user
  postgresql_user:
    name: monitor
    password: "{{ monitoring_password }}"
    role_attr_flags: "NOSUPERUSER,NOCREATEDB,NOCREATEROLE"
  become_user: postgres

- name: Revoke public schema privileges
  postgresql_privs:
    database: "{{ item }}"
    privs: ALL
    type: schema
    objs: public
    role: PUBLIC
    state: absent
  loop: "{{ postgresql_databases }}"
  become_user: postgres
```

---

## 8. Implementation Timeline

| Week | Phase | Activities |
|------|-------|------------|
| 1-2 | Foundation | Base hardening, SSH, sysctl, auditd, AIDE |
| 2-3 | Compliance | OpenSCAP, Lynis, Trivy installation and configuration |
| 3 | Updates | Automatic security updates setup |
| 3-4 | Logging | Promtail, log aggregation, central Loki setup |
| 4-5 | Monitoring | VictoriaMetrics, Grafana, exporters deployment |
| 5 | Alerting | Alert rules, Alertmanager, notification channels |
| 6 | Containers | Docker hardening, CI/CD pipeline security |
| 7 | Testing | Full stack testing, drift detection, documentation |
| 8+ | Maintenance | Ongoing monitoring, updates, compliance reviews |

---

## 9. Quick Reference Commands

```bash
# Run full security hardening
ansible-playbook -i inventory/production playbooks/site.yml --tags security

# Run compliance scan only
ansible-playbook -i inventory/production playbooks/compliance-scanning.yml

# Check drift on all hosts
ansible-playbook -i inventory/production playbooks/drift-detection.yml

# Update monitoring agents
ansible-playbook -i inventory/production playbooks/monitoring-agents.yml

# View Lynis report
cat /var/log/lynis/lynis-report-$(date +%Y%m%d).dat | grep "^warning"

# Run manual Trivy scan
trivy filesystem --severity HIGH,CRITICAL /

# Check OpenSCAP score
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  /usr/share/xml/scap/ssg/content/ssg-almalinux9-ds.xml | tail -20

# View security metrics
curl -s http://localhost:9100/metrics | grep -E "openscap|lynis|trivy"
```
