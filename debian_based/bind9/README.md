# BIND9 DNS Server Setup

This playbook sets up and configures BIND9 DNS server on Ubuntu 24.04 with fully configurable DNS zones and records.

## Features

- Automated BIND9 installation and configuration
- Configurable forward and reverse DNS zones
- Support for multiple record types (A, NS, CNAME, MX, TXT)
- Wildcard DNS support
- Configurable forwarders and recursion
- Optional UFW firewall configuration

## Directory Structure

```
bind9/
├── setup-bind9.yaml           # Main playbook
├── vars/
│   └── dns_config.yaml        # DNS configuration variables
├── templates/
│   ├── named.conf.options.j2  # BIND9 options configuration
│   ├── named.conf.local.j2    # Zone declarations
│   ├── forward.zone.j2        # Forward zone template
│   └── reverse.zone.j2        # Reverse zone template
└── inventory.yaml.example     # Example inventory file
```

## Configuration

### DNS Configuration Variables

Edit [vars/dns_config.yaml](vars/dns_config.yaml) to customize your DNS setup:

#### BIND9 Server Settings

- `bind9_listen_on`: IPv4 listen address (default: "any")
- `bind9_listen_on_v6`: IPv6 listen address (default: "any")
- `bind9_allow_query`: List of allowed query sources (default: ["any"])
- `bind9_forwarders`: List of upstream DNS servers (default: Google DNS)
- `bind9_recursion`: Enable/disable recursion (default: "yes")
- `bind9_dnssec_validation`: DNSSEC validation mode (default: "auto")
- `configure_firewall`: Enable UFW firewall rules (default: false)

#### DNS Zone Settings

- `dns_server_primary`: Primary nameserver hostname
- `dns_server_email`: Contact email (dots replaced with @ in zone files)
- `dns_zones`: List of DNS zones to configure

#### Zone Configuration

Each zone in `dns_zones` supports:

- `domain`: Zone domain name
- `network`: Network address for reverse zone
- `netmask`: Network mask
- `reverse_zone`: Reverse DNS zone name (optional)
- `ttl`: Default TTL value
- `refresh`: SOA refresh interval
- `retry`: SOA retry interval
- `expire`: SOA expire time
- `negative_cache_ttl`: Negative cache TTL
- `serial`: Zone serial number (increment on changes)
- `records`: List of DNS records

#### Supported Record Types

##### A Record
```yaml
- name: "host1"
  type: "A"
  value: "10.13.37.10"
```

##### NS Record
```yaml
- name: "@"
  type: "NS"
  value: "ns1.example.com."
```

##### CNAME Record
```yaml
- name: "www"
  type: "CNAME"
  value: "host1.example.com."
```

##### MX Record
```yaml
- name: "@"
  type: "MX"
  priority: 10
  value: "mail.example.com."
```

##### TXT Record
```yaml
- name: "@"
  type: "TXT"
  value: "v=spf1 mx ~all"
```

##### Wildcard Record
```yaml
- name: "*.apps"
  type: "A"
  value: "10.13.37.100"
```

## Usage

### Initial Setup

1. Copy the example inventory file:
```bash
cp inventory.yaml.example inventory.yaml
```

2. Edit [inventory.yaml](inventory.yaml) with your server details:
```yaml
dns_servers:
  hosts:
    dns-01:
      ansible_host: 10.13.37.135
      ansible_user: ubuntu
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

3. Customize DNS configuration in [vars/dns_config.yaml](vars/dns_config.yaml)

4. Run the playbook:
```bash
ansible-playbook setup-bind9.yaml -i inventory.yaml
```

### Managing DNS Records

To add, modify, or remove DNS records:

1. Edit [vars/dns_config.yaml](vars/dns_config.yaml)
2. Increment the `serial` number in the zone configuration
3. Run the playbook again:
```bash
ansible-playbook setup-bind9.yaml -i inventory.yaml
```

### Adding New Zones

Add a new zone definition to the `dns_zones` list in [vars/dns_config.yaml](vars/dns_config.yaml):

```yaml
dns_zones:
  - domain: "newdomain.com"
    network: "10.10.10.0"
    netmask: "255.255.255.0"
    reverse_zone: "10.10.10.in-addr.arpa"
    ttl: 604800
    serial: 1
    records:
      - name: "@"
        type: "NS"
        value: "ns1.newdomain.com."
```

## Client Configuration

### macOS Configuration

#### Option 1: Domain-Specific DNS (Recommended)

Use macOS's `/etc/resolver/` directory to configure DNS for specific domains. This is the most reliable method for internal domains:

```bash
sudo mkdir -p /etc/resolver
sudo tee /etc/resolver/pve-prod-02.dzarsky.cloud <<EOF
nameserver 10.13.37.135
EOF
```

Flush DNS cache:
```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

This ensures all queries for `*.pve-prod-02.dzarsky.cloud` use your internal DNS server while other queries use your regular DNS.

#### Option 2: WireGuard Configuration

If using WireGuard VPN, add DNS to your WireGuard configuration:

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = YOUR_VPN_IP
DNS = 10.13.37.135

[Peer]
PublicKey = PEER_PUBLIC_KEY
Endpoint = YOUR_ENDPOINT
AllowedIPs = 10.13.37.0/24
```

**Note**: macOS may not always honor WireGuard DNS settings. The `/etc/resolver/` method (Option 1) is more reliable.

#### Option 3: Network Settings

1. Open **System Settings** → **Network**
2. Select your active network interface
3. Click **Details** → **DNS**
4. Click **+** and add `10.13.37.135`
5. Drag it to the top of the DNS server list
6. Click **OK**

### Linux Configuration

#### systemd-resolved

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/dns.conf <<EOF
[Resolve]
DNS=10.13.37.135
Domains=~pve-prod-02.dzarsky.cloud
EOF

sudo systemctl restart systemd-resolved
```

#### NetworkManager

```bash
nmcli connection modify <connection-name> ipv4.dns "10.13.37.135"
nmcli connection modify <connection-name> ipv4.dns-search "pve-prod-02.dzarsky.cloud"
nmcli connection up <connection-name>
```

### Windows Configuration

1. Open **Network & Internet Settings**
2. Click **Change adapter options**
3. Right-click your network adapter → **Properties**
4. Select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**
5. Choose **Use the following DNS server addresses**
6. Enter `10.13.37.135` as Preferred DNS server
7. Click **OK**

## Testing

Verify DNS server is working:

```bash
dig @10.13.37.135 test.pve-prod-02.dzarsky.cloud

dig @10.13.37.135 -x 10.13.37.37

dig @10.13.37.135 test.apps.pve-prod-02.dzarsky.cloud
```

Test without specifying DNS server (after client configuration):

```bash
dig test.pve-prod-02.dzarsky.cloud

nslookup test.pve-prod-02.dzarsky.cloud

curl http://test.pve-prod-02.dzarsky.cloud
```

Check BIND9 configuration on the server:

```bash
named-checkconf

named-checkzone pve-prod-02.dzarsky.cloud /etc/bind/zones/db.pve-prod-02.dzarsky.cloud
```

## Firewall Configuration

If `configure_firewall: true` is set in [vars/dns_config.yaml](vars/dns_config.yaml), the playbook will configure UFW to allow DNS traffic on port 53 (TCP and UDP).

## Example Configuration

The default configuration sets up:

- Zone: `pve-prod-02.dzarsky.cloud`
- Network: `10.13.37.0/24`
- Nameserver: `ns1.pve-prod-02.dzarsky.cloud` (10.13.37.1)
- Sample hosts: host1, host2, host3
- Wildcard: `*.apps.pve-prod-02.dzarsky.cloud` → 10.13.37.100
- Reverse DNS for the 10.13.37.0/24 network

## Troubleshooting

View BIND9 logs:
```bash
journalctl -u named -f
```

Check BIND9 status:
```bash
systemctl status named
```

Validate configuration:
```bash
named-checkconf /etc/bind/named.conf
```
