# CI/CD Runner Setup Playbooks

Playbooks for setting up self-hosted CI/CD runners on RHEL-based systems (AlmaLinux, Rocky Linux, CentOS Stream, RHEL).

## Available Playbooks

### GitHub Actions Runner (`setup-github-actions-runner.yaml`)

Installs and configures a GitHub Actions self-hosted runner (v2.328.0) on RHEL-based systems.

**Required variables:**
- `github_repo_url`: GitHub repository or organization URL (e.g., `https://github.com/yourorg/yourrepo`)
- `github_runner_token`: Runner registration token (obtain from GitHub repo/org settings)

**Optional variables:**
- `github_runner_name`: Runner name (defaults to hostname)
- `github_runner_arch`: Runner architecture - `x64` or `arm64` (defaults to `x64`)
- `github_runner_labels`: Custom labels (defaults to `self-hosted,Linux,X64` or `self-hosted,Linux,ARM64` based on architecture)
- `github_runner_user`: System user for runner (defaults to `ghrunner`)
- `github_runner_dir`: Installation directory (defaults to `/opt/ci/gh/[runner-name]`)
- `github_runner_add_to_docker_group`: Add runner user to docker group (defaults to `false`)

**Usage:**
```bash
ansible-playbook enterprise_linux/ci_runners/setup-github-actions-runner.yaml \
  -i inventories/runners.yaml \
  -e "github_repo_url=https://github.com/yourorg/yourrepo" \
  -e "github_runner_token=YOUR_TOKEN"
```

**Usage with Docker group membership:**
```bash
ansible-playbook enterprise_linux/ci_runners/setup-github-actions-runner.yaml \
  -i inventories/runners.yaml \
  -e "github_repo_url=https://github.com/yourorg/yourrepo" \
  -e "github_runner_token=YOUR_TOKEN" \
  -e "github_runner_add_to_docker_group=true"
```

**Usage for ARM64 architecture:**
```bash
ansible-playbook enterprise_linux/ci_runners/setup-github-actions-runner.yaml \
  -i inventories/runners.yaml \
  -e "github_repo_url=https://github.com/yourorg/yourrepo" \
  -e "github_runner_token=YOUR_TOKEN" \
  -e "github_runner_arch=arm64"
```

**Uninstall:**
```bash
ansible-playbook enterprise_linux/ci_runners/uninstall-github-actions-runner.yaml \
  -i inventories/runners.yaml \
  -e "github_runner_token=YOUR_TOKEN"
```

Add `-e "github_remove_user=true"` to also remove the system user.

### Azure Pipelines Agent (`setup-azure-pipelines-agent.yaml`)

Installs and configures an Azure Pipelines self-hosted agent (v4.263.0) on RHEL-based systems.

**Required variables:**
- `azure_devops_org_url`: Azure DevOps organization URL (e.g., `https://dev.azure.com/yourorg`)
- `azure_devops_pat`: Personal Access Token with agent pool management permissions
- `azure_agent_pool`: Agent pool name (e.g., `Default`)

**Optional variables:**
- `azure_agent_name`: Agent name (defaults to hostname)
- `azure_agent_user`: System user for agent (defaults to `azpagent`)
- `azure_agent_dir`: Installation directory (defaults to `/opt/ci/azp/[agent-name]`)
- `azure_agent_add_to_docker_group`: Add agent user to docker group (defaults to `false`)

**Usage:**
```bash
ansible-playbook enterprise_linux/ci_runners/setup-azure-pipelines-agent.yaml \
  -i inventories/runners.yaml \
  -e "azure_devops_org_url=https://dev.azure.com/yourorg" \
  -e "azure_devops_pat=YOUR_PAT" \
  -e "azure_agent_pool=Default"
```

**Usage with Docker group membership:**
```bash
ansible-playbook enterprise_linux/ci_runners/setup-azure-pipelines-agent.yaml \
  -i inventories/runners.yaml \
  -e "azure_devops_org_url=https://dev.azure.com/yourorg" \
  -e "azure_devops_pat=YOUR_PAT" \
  -e "azure_agent_pool=Default" \
  -e "azure_agent_add_to_docker_group=true"
```

**Uninstall:**
```bash
ansible-playbook enterprise_linux/ci_runners/uninstall-azure-pipelines-agent.yaml \
  -i inventories/runners.yaml \
  -e "azure_devops_pat=YOUR_PAT"
```

Add `-e "azure_remove_user=true"` to also remove the system user.

## Obtaining Tokens

### GitHub Runner Token

1. Navigate to your repository or organization settings
2. Go to Actions → Runners
3. Click "New self-hosted runner"
4. Copy the token from the configuration command

**Note:** Tokens expire after 1 hour

### Azure DevOps PAT

1. Navigate to Azure DevOps organization settings
2. Go to Personal Access Tokens
3. Create new token with "Agent Pools (read, manage)" scope
4. Copy the generated PAT

## Installation Details

Both playbooks:
- Update system packages using DNF
- Install EPEL repository for additional dependencies
- Install dependencies (git, jq, tar, libicu, krb5-libs)
- Create dedicated system user
- Download and extract runner/agent package to `/opt/ci/[azp|gh]/[agent-name]`
- Configure in unattended mode
- Install and start systemd service
- Clean up temporary files

The runners/agents will start automatically on system boot.

## Uninstallation

The uninstall playbooks:
- Stop the running service
- Uninstall systemd service
- Unregister from GitHub/Azure DevOps (requires token)
- Remove installation directory
- Optionally remove system user (set `[github|azure]_remove_user=true`)

## Requirements

- RHEL-based system (AlmaLinux 8+, Rocky Linux 8+, RHEL 8+, CentOS Stream 8+)
- Docker installed (if `[github|azure]_[runner|agent]_add_to_docker_group=true`)
- Internet connectivity for downloading runner/agent packages
- Sudo/root privileges
