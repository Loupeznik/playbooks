# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains Ansible playbooks for infrastructure automation across different platforms and Linux distributions. The playbooks are organized by target platform/OS.

## Repository Structure

- **debian_based/** - Playbooks for Debian/Ubuntu-based systems
  - Basic server setup (Docker, ZSH, user creation)
  - Application deployments (nginx, Jenkins, Go)
  - **k8s-cluster/** - Complete Kubernetes cluster setup
- **proxmox/** - Playbooks for managing Proxmox VE containers

## Running Playbooks

### Basic Execution

```bash
ansible-playbook <playbook-name>.yaml -i <inventory-file>.yaml
```

### Kubernetes Cluster Setup

The k8s cluster playbook requires a properly configured inventory file with credentials and IP addresses:

```bash
ansible-playbook debian_based/k8s-cluster/setup-k8s.yaml -i debian_based/k8s-cluster/inventory.yaml
```

This playbook:
- Deploys Kubernetes v1.31 (kubelet, kubeadm, kubectl)
- Installs Docker and cri-dockerd
- Configures Calico CNI with custom pod network (10.0.0.0/16)
- Deploys Longhorn for persistent storage
- Differentiates between master node (k8s-sbx-01) and worker nodes
- Uses hostvars to share join command from master to workers

### Proxmox Playbooks

Proxmox playbooks run locally and use the community.general.proxmox module:

```bash
ansible-playbook proxmox/deploy-containers.yaml
```

## Inventory Files

Inventory files define target hosts, connection details, and credentials. Structure:

```yaml
group_name:
  hosts:
    hostname:
      ansible_host: <IP>
      ansible_user: <username>
      ansible_ssh_private_key_file: <path_to_key>
```

Inventory files should be stored in the `inventories/` directory (gitignored).

## Important Patterns

### Debian-based Playbooks

- Always update package cache with `update_cache: true`
- Use `become: true` and `become_user: root` for privilege escalation
- Repository keys are stored in `/etc/apt/keyrings/`
- Package sources in `/etc/apt/sources.list.d/`

### K8s Cluster Playbook Specifics

- Master node is identified by `inventory_hostname == 'k8s-sbx-01'`
- Join command is created on master and stored in hostvars: `hostvars['k8s-sbx-01'].join_command`
- CRI socket specification required: `--cri-socket unix:///var/run/cri-dockerd.sock`
- Calico pod CIDR is customized using sed to replace default 192.168.0.0/16 with 10.0.0.0/16

### Proxmox Playbooks

- Use `connection: local` and `hosts: 'local'` for local execution
- Require API credentials (api_user, api_token_id, api_token_secret, api_host)
- Container network configuration is passed as JSON string in netif parameter
