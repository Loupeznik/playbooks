# Enterprise Linux Playbooks

Playbooks for RHEL-based systems (AlmaLinux, Rocky Linux, CentOS Stream, RHEL).

## Available Playbooks

### Basic Server Setup (`setup-server-basic.yml`)

Sets up a basic RHEL-based server with Docker and creates a management user.

**Required variables:**
None - all values are configured in the playbook

**What it does:**
- Updates all system packages
- Installs EPEL repository
- Installs essential tools (curl, git, wget, htop, etc.)
- Installs Docker and Docker Compose
- Starts and enables Docker service
- Creates a management user (`cc`) with bash shell
- Adds user to docker and wheel (sudo) groups
- Sets up SSH authorized keys for the user

**Usage:**
```bash
ansible-playbook enterprise_linux/setup-server-basic.yml \
  -i inventories/servers.yaml
```

**Note:** You should customize the SSH public key and username in the playbook before running it.

### PHP and Apache Setup (`setup-php-apache.yaml`)

Installs and configures PHP with Apache httpd on RHEL-based systems using the Remi repository for PHP versions.

**Required variables:**
None - all variables have sensible defaults

**Optional variables:**
- `target_rhel_version`: RHEL major version (8, 9, or 10) - defaults to `10`
- `target_php_version`: PHP version to install (e.g., `8.1`, `8.2`, `8.3`) - defaults to `8.3`
- `install_apache`: Whether to install and configure Apache httpd - defaults to `true`
- `php_modules`: List of PHP extensions to install - defaults to empty list

**Usage:**

Basic installation with defaults (RHEL 10, PHP 8.3, Apache enabled):
```bash
ansible-playbook enterprise_linux/setup-php-apache.yaml \
  -i inventories/webservers.yaml
```

Install PHP 8.2 on RHEL 9 with Apache:
```bash
ansible-playbook enterprise_linux/setup-php-apache.yaml \
  -i inventories/webservers.yaml \
  -e "target_rhel_version=9" \
  -e "target_php_version=8.2"
```

Install PHP 8.3 without Apache:
```bash
ansible-playbook enterprise_linux/setup-php-apache.yaml \
  -i inventories/webservers.yaml \
  -e "install_apache=false"
```

Install PHP with common extensions:
```bash
ansible-playbook enterprise_linux/setup-php-apache.yaml \
  -i inventories/webservers.yaml \
  -e "php_modules=['php-mysqlnd','php-gd','php-mbstring','php-xml','php-curl','php-zip','php-opcache','php-intl','php-bcmath']"
```

**Common PHP Extensions:**
- `php-mysqlnd` - MySQL/MariaDB database support
- `php-gd` - Image processing
- `php-mbstring` - Multibyte string handling
- `php-xml` - XML processing
- `php-curl` - cURL support
- `php-zip` - ZIP archive handling
- `php-opcache` - Opcode caching
- `php-intl` - Internationalization
- `php-ldap` - LDAP support
- `php-soap` - SOAP protocol support
- `php-bcmath` - Arbitrary precision mathematics
- `php-pgsql` - PostgreSQL database support
- `php-process` - Process control extensions (posix, shmop, sysvmsg, sysvsem, sysvshm)
- `php-pecl-redis` - Redis support

**Note:** Some extensions like `php-pdo` and `php-json` are built into the base PHP package on PHP 8.0+ and don't need to be installed separately. The playbook will automatically skip packages that aren't available or are already included.

**Installation Details:**

The playbook:
- Updates all system packages
- Installs Apache httpd (if enabled)
- Installs EPEL repository for the specified RHEL version
- Installs Remi repository for PHP packages
- Resets and enables the specified PHP module stream
- Installs PHP and requested extensions
- Starts and enables Apache httpd service (if enabled)
- Configures firewall for HTTP/HTTPS (if enabled)
- Displays installed PHP version

**Requirements:**
- RHEL-based system (AlmaLinux 8+, Rocky Linux 8+, RHEL 8+, CentOS Stream 8+)
- Internet connectivity for downloading packages and repositories
- Sudo/root privileges
- Firewalld installed (optional, for firewall configuration)

**Supported PHP Versions:**
- PHP 8.1, 8.2, 8.3 (from Remi repository)
- PHP 7.4 (legacy support)

**Supported RHEL Versions:**
- RHEL/AlmaLinux/Rocky Linux 8
- RHEL/AlmaLinux/Rocky Linux 9
- RHEL/AlmaLinux 10

## Subdirectories

- **ci_runners/** - CI/CD runner setup (GitHub Actions, Azure Pipelines)
