# RKE2 Cluster Node Preparation Playbook

This playbook prepares Ubuntu 24.04 nodes for RKE2 cluster deployment via Rancher custom cluster registration.

## Overview

The playbook configures Ubuntu 24.04 nodes with all necessary prerequisites for RKE2, including kernel parameters, system users, firewall rules, and network settings required by Rancher-managed RKE2 clusters.

## Prerequisites

- Ubuntu 24.04 LTS nodes
- SSH access with sudo privileges
- Ansible installed on control machine
- Properly configured inventory file

## Inventory Structure

```yaml
all:
  children:
    controlplane:
      hosts:
        master-01:
          ansible_host: 192.168.1.100
    workers:
      hosts:
        worker-01:
          ansible_host: 192.168.1.110
  vars:
    ansible_user: ubuntu
    ansible_become: true
    ansible_ssh_private_key_file: ~/.ssh/your_key
```

## Usage

```bash
ansible-playbook playbook.yml -i inventory.yml
```

## What the Playbook Does

### System Configuration
- Updates apt cache and installs required packages
- Creates etcd system user and group (required by RKE2)
- Disables swap (immediately and permanently)
- Sets unique hostnames and updates `/etc/hosts`

### Kernel Configuration
- Loads required kernel modules (`overlay`, `br_netfilter`)
- Configures sysctl parameters for Kubernetes and RKE2:
  - `net.bridge.bridge-nf-call-iptables = 1`
  - `net.bridge.bridge-nf-call-ip6tables = 1`
  - `net.ipv4.ip_forward = 1`
  - `vm.overcommit_memory = 1`
  - `kernel.panic = 10`
  - `kernel.panic_on_oops = 1`

### Firewall Configuration (UFW)
- Configures firewall rules for control plane nodes:
  - SSH (22/tcp)
  - Kubernetes API (6443/tcp)
  - RKE2 supervisor API (9345/tcp)
  - etcd (2379-2380/tcp)
  - Kubelet (10250/tcp)
  - NodePort services (30000-32767/tcp)

- Configures firewall rules for worker nodes:
  - SSH (22/tcp)
  - RKE2 supervisor API (9345/tcp)
  - Kubelet (10250/tcp)
  - NodePort services (30000-32767/tcp)

### Additional Tasks
- Configures DNS servers (if defined)
- Enables and starts chrony for time synchronization
- Optionally disables AppArmor
- Optionally prepares template for cloning (removes machine-id)
- Verifies setup (swap, kernel modules, sysctl)
- Reboots nodes to apply all changes

## Variables

Configuration is managed in `group_vars/all.yml`:

- `required_packages` - List of packages to install
- `dns_servers` - DNS server IPs
- `manage_firewall` - Enable/disable UFW configuration (default: true)
- `disable_apparmor` - Enable/disable AppArmor (default: false)
- `controlplane_ports` - Firewall rules for control plane nodes
- `worker_ports` - Firewall rules for worker nodes
- `prepare_template` - Prepare node as VM template (default: false)

## Post-Playbook Steps

After running this playbook:

1. Ensure nodes have rebooted successfully
2. Create RKE2 custom cluster in Rancher UI
3. Copy registration commands from Rancher
4. Run registration commands on each node
5. Monitor cluster provisioning in Rancher dashboard

## Important Notes

- Nodes will be rebooted at the end of playbook execution
- UFW firewall is enabled by default - ensure SSH access is maintained
- The playbook does NOT install RKE2 - that is handled by Rancher registration
- All kernel parameters and system users are pre-configured for RKE2 requirements
- Hostname resolution is configured to prevent "unable to resolve host" errors

## Troubleshooting

### SSH Connection Issues
- Verify SSH port 22 is allowed in firewall rules
- Check `controlplane_ports` and `worker_ports` in `group_vars/all.yml`

### Swap Not Disabled
- Verify playbook completed successfully
- Check `/etc/fstab` for commented swap entries
- Run `swapon --show` to verify no swap is active

### Hostname Resolution Errors
- Playbook configures `/etc/hosts` with `127.0.1.1` entry
- Verify hostname matches inventory_hostname

## Files

- `playbook.yml` - Main playbook
- `inventory.example.yml` - Example inventory structure
- `group_vars/all.yml` - Configuration variables
