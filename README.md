# Ansible playbooks

Collection of Ansible playbooks for infrastructure automation across Debian-based systems, RHEL-based systems, and Proxmox VE.

## Requirements

- Python 3.8+
- Ansible 2.14+
- SSH access to target hosts
- Properly configured SSH keys

## Environment Setup

### Using System Python

```bash
# Install Ansible and required collections
pip install ansible

# Install Proxmox community collection (required for Proxmox playbooks)
ansible-galaxy collection install community.general
```

### Using Python Virtual Environment with uv (Recommended)

```bash
# Install uv if not already installed
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create virtual environment
uv venv

# Activate virtual environment
source .venv/bin/activate

# Install Ansible
uv pip install ansible

# Install Proxmox community collection
ansible-galaxy collection install community.general
```

## Configuration

Create inventory files in the `inventories/` directory (gitignored). Example structure:

```yaml
group_name:
  hosts:
    hostname:
      ansible_host: 192.168.1.100
      ansible_user: admin
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

## Running Playbooks

```bash
ansible-playbook <playbook-name>.yaml -i inventories/<inventory-file>.yaml
```

## Available Playbooks

- **debian_based/** - Debian/Ubuntu system configuration and application deployments
- **enterprise_linux/** - RHEL-based system configuration (AlmaLinux, Rocky Linux, CentOS Stream, RHEL)
- **proxmox/** - Proxmox VE container management

See individual README files in subdirectories for specific playbook details.
