# Static Network Configuration — VM Network Summary

## 1. Network Architecture

My host is connected to the Wi-Fi router through:

wlp0s20f3
    IP: 192.168.1.10/24

The router/default gateway is:

192.168.1.1

The local network is:

192.168.1.0/24

The VM is connected through VirtualBox:

VirtualBox
    ↓
Bridged Adapter
    ↓
VM network interface: enp0s3

The VM uses:

IP:       192.168.1.50/24
Gateway:  192.168.1.1
DNS:      8.8.8.8
          1.1.1.1


## 2. Understanding 192.168.1.0/24

The `/24` is the CIDR prefix.

IPv4 has 32 bits:

    24 bits = network portion
     8 bits = host portion

Subnet mask:

    /24 = 255.255.255.0

Therefore:

    Network address: 192.168.1.0
    Usable hosts:    192.168.1.1 - 192.168.1.254
    Broadcast:       192.168.1.255

The `/24` tells a machine which addresses belong to
its local network.

For example:

    VM:          192.168.1.50/24
    Host:        192.168.1.10/24

Both belong to:

    192.168.1.0/24

Therefore they are on the same network.


## 3. Important Distinction About the Gateway

The `/24` does NOT tell us that the router must be:

    192.168.1.1

It only tells us that `.1` is a possible host address.

We know that `192.168.1.1` is our gateway because:

    ip route

shows:

    default via 192.168.1.1 dev wlp0s20f3

So:

    /24 → tells us the network boundary
    ip route → tells us the default gateway


## 4. DHCP

DHCP = Dynamic Host Configuration Protocol.

In a typical home network, there is no separate physical DHCP
server.

The DHCP server is usually software running inside the
Wi-Fi router.

The router can provide:

    IP address
    Subnet mask / CIDR
    Default gateway
    DNS server
    Lease duration

My host received:

    192.168.1.10/24

through DHCP.

This is why `ip a` showed:

    dynamic

DHCP answers:

    "What network configuration should this device use?"


## 5. Routing

Routing answers:

    "Where should this packet go?"

My host has:

    default via 192.168.1.1 dev wlp0s20f3

This means:

    Local network:
        192.168.1.0/24

    Other networks:
        send packets to 192.168.1.1


Example:

    Destination = 192.168.1.10
    → destination is inside 192.168.1.0/24
    → communicate through the local network

    Destination = 8.8.8.8
    → destination is outside 192.168.1.0/24
    → send packet to 192.168.1.1
    → router forwards it toward the Internet


## 6. DNS

DNS translates names into IP addresses.

Example:

    google.com
        ↓
    DNS lookup
        ↓
    IP address

The Netplan configuration specifies:

    8.8.8.8
    1.1.1.1

as DNS resolvers.


## 7. VirtualBox NAT vs Bridged Adapter

With NAT:

    Host
    192.168.1.10
         ↓
    VirtualBox NAT
         ↓
    VM
    10.0.2.15

The VM is behind VirtualBox's private NAT network.

The VM can normally initiate connections outward, but
the host cannot directly reach the VM as if it were another
machine on the physical LAN.

SSH can be made to work with NAT using port forwarding:

    Host 127.0.0.1:2222
            ↓
       VirtualBox NAT
            ↓
       VM:2222


With Bridged Adapter:

    Wi-Fi Router
    192.168.1.1
          │
          │
    192.168.1.0/24
          │
       ┌──┴──┐
       │     │
      Host   VM
      .10    .50

The VM becomes another machine on the same LAN.


## 8. Netplan

Netplan is Ubuntu's network configuration layer.

It reads YAML configuration files under:

    /etc/netplan/

Example:

    network:
      version: 2
      ethernets:
        enp0s3:
          dhcp4: no
          addresses:
            - 192.168.1.50/24
          routes:
            - to: default
              via: 192.168.1.1
          nameservers:
            addresses: [8.8.8.8, 1.1.1.1]

Netplan describes the desired network configuration and
generates configuration for the network backend used by Ubuntu.


## 9. Meaning of the Netplan Configuration

    enp0s3:

The VM's Ethernet interface.

    dhcp4: no

Do not obtain the IPv4 configuration dynamically through DHCP.

    addresses:
      - 192.168.1.50/24

Assign the VM the static IPv4 address:

    192.168.1.50

with:

    /24 = 255.255.255.0

    routes:
      - to: default
        via: 192.168.1.1

Use `192.168.1.1` as the default gateway.

    nameservers:
      addresses: [8.8.8.8, 1.1.1.1]

Use these DNS servers.


## 10. Why the VM Needs a Static IP

A server provides services that clients need to find.

Example:

    VM
    192.168.1.50
        │
        ├── SSH
        ├── HTTP
        └── Application

A client can connect using:

    ssh user@192.168.1.50

If DHCP changes the address:

    Monday:    192.168.1.50
    Tuesday:   192.168.1.73
    Wednesday: 192.168.1.42

clients using the old address can no longer reliably reach
the server.

A static IP provides a predictable address:

    Server → 192.168.1.50


## 11. Why DNS and Firewall Rules Care

DNS:

    server.example.com
            ↓
       192.168.1.50

If the server's address changes, the DNS record may need to
be updated.

Firewall:

    ALLOW 192.168.1.10 → 192.168.1.50:22

This rule refers to a specific server address.

If the server's IP changes, the rule may no longer target
the intended machine.


## 12. Verification Commands

Apply Netplan:

    sudo netplan apply

Check the interface:

    ip a

Expected:

    enp0s3
        inet 192.168.1.50/24

Check routing:

    ip route

Expected:

    default via 192.168.1.1 dev enp0s3

and a route similar to:

    192.168.1.0/24 dev enp0s3 ... src 192.168.1.50

Check for DHCP:

    ip a | grep dynamic

For the static IPv4 address, `dynamic` should not appear.

Test the gateway:

    ping -c 5 192.168.1.1

Test Internet connectivity:

    ping -c 5 8.8.8.8

Test DNS:

    ping -c 5 google.com


## 13. Complete Mental Model

    Physical / Virtual Network
             │
             ▼
       VirtualBox
             │
       Bridged Adapter
             │
             ▼
          enp0s3
             │
             ▼
          Netplan
             │
       ┌─────┼─────────────┐
       │     │             │
       ▼     ▼             ▼
      IP    Route          DNS
      │       │             │
      ▼       ▼             ▼
    .50/24   .1         8.8.8.8
                       1.1.1.1


## 14. Core Concepts

CIDR / Netmask:

    Defines the boundary between network and host portions.

Network address:

    192.168.1.0

Static VM address:

    192.168.1.50

Default gateway:

    192.168.1.1

Broadcast:

    192.168.1.255

DHCP:

    Automatically provides network configuration.

Static configuration:

    Manually defines a predictable network configuration.

Netplan:

    Defines Ubuntu's desired network configuration using YAML.

Routing:

    Determines where packets should be sent.

DNS:

    Converts domain names into IP addresses.

VirtualBox Bridged Adapter:

    Connects the VM to the same physical network as the host.


## 15. Final Network

                         Internet
                            │
                            ▼
                    Wi-Fi Router
                     192.168.1.1
                            │
                     192.168.1.0/24
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
              Host                    VM
        192.168.1.10             192.168.1.50
        wlp0s20f3                    enp0s3
                                      │
                                   Netplan
                                      │
                       ┌──────────────┼──────────────┐
                       ▼              ▼              ▼
                      IP/CIDR       Gateway          DNS
                  192.168.1.50    192.168.1.1   8.8.8.8
                       /24                         1.1.1.1


## 16. One-Sentence Summary

VirtualBox connects the VM to the network, Netplan configures
the VM's network interface, CIDR defines the local network,
the gateway provides access to other networks, DNS resolves
names, DHCP can automatically provide these settings, and a
static IP gives a server a predictable address.