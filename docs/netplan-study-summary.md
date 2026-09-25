# Netplan — Study Summary

## 1. What is Netplan?

**Netplan** is Ubuntu's network configuration abstraction layer.

It uses YAML configuration files to describe the desired network configuration and then passes that configuration to a backend.

```text
/etc/netplan/*.yaml
        │
        ▼
      Netplan
        │
        ├── systemd-networkd
        │
        └── NetworkManager
```

On Ubuntu Server, `systemd-networkd` is commonly used.

---

## 2. Netplan configuration location

Configuration files are stored in:

```bash
/etc/netplan/
```

Check them:

```bash
ls -l /etc/netplan/
```

Typical files:

```text
00-installer-config.yaml
50-cloud-init.yaml
99-custom.yaml
```

The filename/order can matter because multiple YAML files may be processed.

---

## 3. YAML basics

Netplan uses YAML.

Example:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
```

Indentation defines the hierarchy.

```text
network
├── version
└── ethernets
    └── enp0s3
        └── dhcp4
```

Use spaces, not tabs.

---

## 4. Identify network interfaces

Before configuring Netplan, determine the actual interface names.

```bash
ip link
```

or:

```bash
ip addr
```

A modern Linux system may use:

```text
enp0s3
enp0s8
ens33
eno1
```

rather than the old:

```text
eth0
```

Never assume the interface name.

---

# 5. DHCP

Basic DHCP configuration:

```yaml
network:
  version: 2

  ethernets:
    enp0s3:
      dhcp4: true
```

DHCP can provide:

```text
IP address
subnet/prefix
default gateway
DNS servers
```

---

# 6. Static IP

Example:

```yaml
network:
  version: 2

  ethernets:
    enp0s3:
      dhcp4: false

      addresses:
        - 192.168.1.50/24

      routes:
        - to: default
          via: 192.168.1.1

      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

This configures:

```text
IP       → 192.168.1.50
Network  → 192.168.1.0/24
Gateway  → 192.168.1.1
DNS      → 1.1.1.1
           8.8.8.8
```

---

# 7. CIDR notation

This:

```text
192.168.1.50/24
```

means:

```text
IP address     = 192.168.1.50
prefix length  = 24
netmask        = 255.255.255.0
```

For a `/24` network:

```text
Network:       192.168.1.0
Usable hosts:  192.168.1.1 - 192.168.1.254
Broadcast:     192.168.1.255
```

Understanding CIDR is essential for understanding Netplan.

---

# 8. Default gateway

Modern Netplan configuration commonly uses:

```yaml
routes:
  - to: default
    via: 192.168.1.1
```

Meaning:

```text
Destination unknown/local?
        │
        ▼
192.168.1.1
```

The default IPv4 route is effectively:

```text
0.0.0.0/0
```

---

# 9. DNS

DNS configuration:

```yaml
nameservers:
  addresses:
    - 1.1.1.1
    - 8.8.8.8
```

Optional search domains:

```yaml
nameservers:
  search:
    - example.local

  addresses:
    - 192.168.1.1
    - 1.1.1.1
```

DNS converts names such as:

```text
google.com
```

into IP addresses.

DNS and routing are separate concepts.

You can have:

```bash
ping 8.8.8.8
```

working while:

```bash
ping google.com
```

fails because of DNS.

---

# 10. Netplan commands

## Generate configuration

```bash
sudo netplan generate
```

This validates/generates backend configuration from your YAML.

Conceptually:

```text
YAML
 ↓
Netplan
 ↓
backend configuration
```

---

## Apply configuration

```bash
sudo netplan apply
```

This activates the configuration.

---

## Safely test configuration

```bash
sudo netplan try
```

This is particularly useful when connected to a remote server over SSH.

Conceptually:

```text
new configuration
       ↓
temporary application
       ↓
confirm?
   /        \
 yes        no
 ↓           ↓
keep       revert
```

This reduces the risk of permanently losing remote access because of a networking mistake.

---

# 11. Verify the actual configuration

Netplan tells the system what you want.

Linux commands show what actually happened.

### IP addresses

```bash
ip addr
```

or:

```bash
ip -br addr
```

Example:

```text
enp0s3    UP    192.168.1.50/24
```

### Interfaces

```bash
ip link
```

### Routing table

```bash
ip route
```

Example:

```text
default via 192.168.1.1 dev enp0s3
192.168.1.0/24 dev enp0s3
```

### DNS

```bash
resolvectl status
```

---

# 12. Network troubleshooting sequence

When networking doesn't work, don't immediately modify Netplan.

Check from the bottom up:

```text
Interface
    ↓
IP address
    ↓
Routing
    ↓
Gateway
    ↓
External connectivity
    ↓
DNS
```

### 1. Interface

```bash
ip link
```

Is it `UP`?

### 2. IP

```bash
ip addr
```

Does it have the expected address?

### 3. Route

```bash
ip route
```

Is there a default route?

### 4. Gateway

```bash
ping 192.168.1.1
```

### 5. External IP

```bash
ping 8.8.8.8
```

### 6. DNS

```bash
resolvectl status
```

Then:

```bash
getent hosts google.com
```

This lets you identify which layer is failing.

---

# 13. Multiple interfaces

You can configure multiple interfaces:

```yaml
network:
  version: 2

  ethernets:
    enp0s3:
      dhcp4: true

    enp0s8:
      addresses:
        - 192.168.10.10/24
```

Each interface can have its own:

- IP configuration
- routes
- DNS configuration
- DHCP settings

---

# 14. Interface matching

Netplan can identify an interface using its MAC address.

```yaml
match:
  macaddress: "08:00:27:12:34:56"
```

You can also rename it:

```yaml
set-name: lan0
```

Conceptually:

```text
MAC address
     ↓
find interface
     ↓
rename to lan0
```

---

# 15. Routing

Static routes can be defined with:

```yaml
routes:
  - to: 10.10.0.0/16
    via: 192.168.1.1
```

Meaning:

```text
Destination: 10.10.0.0/16
Next hop:    192.168.1.1
```

You can have multiple routes:

```yaml
routes:
  - to: 10.0.0.0/8
    via: 192.168.1.1

  - to: 172.16.0.0/12
    via: 192.168.1.254
```

---

# 16. Longest prefix match

When several routes match a destination, Linux generally prefers the most specific route.

Example:

```text
10.0.0.0/8
10.10.0.0/16
10.10.20.0/24
```

For:

```text
10.10.20.50
```

the `/24` route is more specific than `/16`, which is more specific than `/8`.

This is a fundamental routing concept, not specifically a Netplan concept.

---

# 17. Route metrics

Routes can have metrics:

```yaml
routes:
  - to: default
    via: 192.168.1.1
    metric: 100
```

Metrics help determine which route is preferred when multiple routes are available.

---

# 18. IPv6

Netplan supports IPv6.

Example:

```yaml
addresses:
  - "2001:db8::10/64"

routes:
  - to: default
    via: "2001:db8::1"
```

Important IPv6 concepts to eventually learn:

```text
IPv6 addresses
CIDR
SLAAC
DHCPv6
Router Advertisements
IPv6 routing
```

---

# 19. Wi-Fi

Netplan can configure wireless interfaces:

```yaml
network:
  version: 2

  wifis:
    wlan0:
      dhcp4: true

      access-points:
        "MyNetwork":
          password: "mypassword"
```

The important structural difference is:

```yaml
ethernets:
```

versus:

```yaml
wifis:
```

---

# 20. VLANs

Netplan can create VLAN interfaces.

Example:

```yaml
vlans:
  vlan10:
    id: 10
    link: enp0s3
    addresses:
      - 192.168.10.10/24
```

Conceptually:

```text
enp0s3
   │
   ▼
VLAN 10
   │
   ▼
vlan10
```

---

# 21. Bridges

A Linux bridge acts roughly like a software Layer-2 switch.

Example:

```yaml
bridges:
  br0:
    interfaces:
      - enp0s3
    dhcp4: true
```

Conceptually:

```text
           br0
         /     \
     enp0s3    VM
```

Bridges are particularly relevant to:

- KVM
- QEMU
- LXC
- virtual machines
- containers

---

# 22. Bonding

Network bonding combines multiple interfaces:

```text
enp1s0 ─┐
        ├── bond0
enp2s0 ─┘
```

It can be used for:

- redundancy
- availability
- specific performance configurations

This is an advanced Netplan topic.

---

# 23. Tunnels

Netplan can also configure networking technologies such as:

```text
GRE
IPIP
VXLAN
WireGuard
```

These should be studied after understanding normal routing and interfaces.

---

# 24. Netplan and systemd-networkd

A common Ubuntu Server architecture is:

```text
Netplan YAML
     ↓
Netplan
     ↓
systemd-networkd
     ↓
Linux kernel networking
```

Check the service:

```bash
systemctl status systemd-networkd
```

And:

```bash
networkctl
```

For a specific interface:

```bash
networkctl status enp0s3
```

---

# 25. Netplan and NetworkManager

Ubuntu systems can also use NetworkManager:

```text
Netplan
   ↓
NetworkManager
```

Check:

```bash
systemctl status NetworkManager
```

and:

```bash
nmcli
```

Do not confuse:

```text
Netplan
```

with:

```text
systemd-networkd
```

or:

```text
NetworkManager
```

They have different roles.

---

# 26. Netplan vs `ip`

This distinction is important.

`ip` displays or manipulates the current Linux networking state.

Examples:

```bash
ip addr
ip link
ip route
```

Netplan describes persistent desired configuration:

```text
/etc/netplan/*.yaml
```

For example:

```bash
sudo ip addr add 192.168.1.50/24 dev enp0s3
```

can modify the current state without necessarily becoming persistent.

Netplan provides persistent configuration.

---

# 27. DNS and `resolv.conf`

Modern Ubuntu systems may use:

```text
Netplan
   ↓
systemd-resolved
   ↓
DNS
```

Check:

```bash
resolvectl status
```

and:

```bash
ls -l /etc/resolv.conf
```

`/etc/resolv.conf` may be a symbolic link managed by the system.

---

# 28. Cloud-init

You may encounter:

```text
/etc/netplan/50-cloud-init.yaml
```

Cloud-init can generate network configuration.

Check:

```bash
ls -l /etc/netplan/
```

and:

```bash
cat /etc/netplan/50-cloud-init.yaml
```

Understand who owns/manages a configuration file before editing it permanently.

---

# 29. File permissions

Check Netplan files:

```bash
ls -l /etc/netplan/
```

Configuration containing things such as Wi-Fi passwords should not be unnecessarily readable by other users.

A common restrictive permission is:

```bash
sudo chmod 600 /etc/netplan/file.yaml
```

---

# 30. VirtualBox + Netplan

For an Ubuntu Server VM, the complete networking chain is:

```text
Physical network
       │
       ▼
VirtualBox networking mode
       │
       ▼
Virtual NIC
       │
       ▼
Ubuntu interface
       │
       ▼
Netplan
       │
       ▼
systemd-networkd
       │
       ▼
Linux networking
```

If VirtualBox is configured incorrectly, changing Netplan may not solve the problem.

---

# 31. NAT vs Bridged networking

### NAT

```text
Ubuntu VM
    │
    ▼
VirtualBox NAT
    │
    ▼
Host
    │
    ▼
Internet
```

The VM is behind VirtualBox's virtual NAT network.

### Bridged

```text
Ubuntu VM
    │
    ▼
VirtualBox bridge
    │
    ▼
Physical LAN
    │
    ▼
Router
```

The VM behaves more like another machine on the physical LAN.

---

# 32. Example static configuration

For a server with:

```text
Interface: enp0s3
IP:        192.168.1.6
Network:   /24
Gateway:   192.168.1.1
DNS:       1.1.1.1
```

the configuration is:

```yaml
network:
  version: 2

  ethernets:
    enp0s3:
      dhcp4: false

      addresses:
        - 192.168.1.6/24

      routes:
        - to: default
          via: 192.168.1.1

      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Then:

```bash
sudo netplan generate
sudo netplan try
```

After confirmation:

```bash
ip -br addr
ip route
resolvectl status
```

---

# 33. Most important commands

## Netplan

```bash
sudo netplan generate
sudo netplan try
sudo netplan apply
```

## Interfaces

```bash
ip link
ip addr
ip -br addr
```

## Routes

```bash
ip route
ip route get 8.8.8.8
```

## DNS

```bash
resolvectl status
resolvectl query google.com
```

## Backend

```bash
systemctl status systemd-networkd
networkctl
networkctl status enp0s3
```

## NetworkManager

```bash
systemctl status NetworkManager
nmcli
```

---

# 34. Recommended learning order

For Linux administration:

```text
1. Linux network interfaces
        ↓
2. MAC addresses
        ↓
3. IPv4
        ↓
4. Subnets + CIDR
        ↓
5. DHCP
        ↓
6. Default gateways
        ↓
7. Routing tables
        ↓
8. DNS
        ↓
9. Netplan DHCP
        ↓
10. Netplan static IP
        ↓
11. Netplan troubleshooting
        ↓
12. systemd-networkd
        ↓
13. NetworkManager
        ↓
14. Multiple interfaces
        ↓
15. Static routes
        ↓
16. IPv6
        ↓
17. VLANs
        ↓
18. Bridges
        ↓
19. Bonding
        ↓
20. Tunnels
```

---

# 35. Core mental model

The most important model is:

```text
             CONFIGURATION
                   │
                   ▼
        /etc/netplan/*.yaml
                   │
                   ▼
                Netplan
                   │
             ┌─────┴─────┐
             ▼           ▼
     systemd-networkd  NetworkManager
             │           │
             └─────┬─────┘
                   ▼
          Linux networking
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
     Address     Routes       DNS
```

For troubleshooting:

```text
Interface
    ↓
IP address
    ↓
Subnet
    ↓
Routing table
    ↓
Gateway
    ↓
External connectivity
    ↓
DNS
```

The key distinction is:

> **Netplan describes how the network should be configured; `ip`, `networkctl`, `resolvectl`, and related tools let you inspect what the system is actually doing.**
