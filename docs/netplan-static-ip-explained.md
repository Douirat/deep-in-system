# Ubuntu Server Network Configuration — DHCP vs Static IP (Netplan)

## 1. What Netplan Is

Ubuntu Server (20.04+) uses **Netplan** to configure network interfaces. Instead of editing low-level network scripts, you write a YAML file describing how each interface should behave, and Netplan translates that into actual network settings (via `systemd-networkd` or `NetworkManager` underneath).

Configuration files live in:

```
/etc/netplan/
```

---

## 2. The Default File: What's Actually Happening

The installer (`subiquity`) automatically generates a default config, usually named `00-installer-config.yaml`:

```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp6: true
      match:
        macaddress: 08:00:27:cc:35:30
      set-name: enp0s3
  version: 2
```

**Logical walkthrough, line by line:**

| Line | What it does |
|---|---|
| `network:` | Top-level key — everything inside describes the network setup. |
| `ethernets:` | This block configures **wired** interfaces (as opposed to `wifis:` or `bridges:`). |
| `enp0s3:` | The interface being configured. The name is "predictable" — derived from its PCI bus slot, common on VMs (VirtualBox, in this case). |
| `dhcp4: true` | The interface requests an IPv4 address **automatically** from a DHCP server on the network. |
| `dhcp6: true` | Same, but for IPv6. |
| `match: macaddress: ...` | Instead of blindly trusting whatever name the kernel assigns, Netplan finds the interface **by its MAC address**. This makes the config resilient even if the kernel renames interfaces between boots. |
| `set-name: enp0s3` | Once the interface is matched by MAC, force it to be named `enp0s3` regardless of what the kernel would otherwise call it. |
| `version: 2` | The Netplan YAML schema version in use (2 is current). |

**In plain terms:** this file says *"whatever interface has this MAC address, call it enp0s3, and go ask a DHCP server for an IP address automatically."* This is a **dynamic** configuration — the IP can change over time.

---

## 3. Static Network Configuration

To make the server's address permanent, you disable DHCP and assign the IP, gateway, and DNS manually.

Edit the appropriate file:

```bash
sudo nano /etc/netplan/<your-file>.yaml
```

```yaml
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
```

**What changed, and why:**

| Line | Purpose |
|---|---|
| `dhcp4: no` | Stop requesting an IP automatically — we're assigning it ourselves. |
| `addresses:` | The fixed IP address(es) for this interface, written in **CIDR notation** (see below). |
| `routes: - to: default via: 192.168.1.1` | Declares the **default gateway** — the router that forwards traffic to networks outside `192.168.1.0/24`. This is the modern replacement for the older `gateway4:` key. |
| `nameservers: addresses:` | DNS resolvers to use, since DHCP is no longer supplying them automatically. |

---

## 4. Apply and Verify

```bash
sudo netplan generate    # validates the YAML and builds the backend config
sudo netplan try         # applies temporarily; auto-reverts if you don't confirm (safe test)
sudo netplan apply       # applies permanently

ip a                     # confirm the new static IP is set on enp0s3
ip a | grep dynamic      # should return NOTHING — dynamic = DHCP-assigned, we want none
ip route                 # confirm the gateway is correct
ping -c 5 192.168.1.1    # test the gateway is reachable
ping -c 5 8.8.8.8        # test raw internet connectivity (no DNS involved)
ping -c 5 google.com     # test DNS resolution + internet connectivity together
```

**Logical order and why it matters:**
1. `generate` first — catches YAML syntax errors before touching the live network.
2. `try` before `apply` — if the config is wrong (e.g., wrong gateway), it self-reverts after ~120 seconds, so you don't lock yourself out of an SSH session.
3. Verify the **interface** (`ip a`) before the **route** (`ip route`) before **connectivity** (`ping`) — this isolates *where* a problem is if something fails: is the IP wrong, the gateway wrong, or DNS wrong?
4. Ping the gateway *before* the internet — if the gateway isn't reachable, nothing beyond it will work either, so this narrows the fault fast.
5. Ping a raw IP (`8.8.8.8`) *before* a domain name (`google.com`) — this separates "routing/internet is broken" from "DNS is broken."

---

## 5. Core Concepts

### Netmask / CIDR
An IP address is split into two parts: a **network portion** and a **host portion**. The netmask (or CIDR suffix, e.g. `/24`) defines where that split happens.

`192.168.1.50/24` means:
- The first 24 bits (`192.168.1.x`) identify the **network**.
- The remaining 8 bits identify the **specific host** on that network.
- A `/24` has 256 total addresses (`.0` to `.255`), with 254 usable for hosts (`.0` = network address, `.255` = broadcast address).

This is why `192.168.1.50` (the server) and `192.168.1.1` (the gateway) can communicate directly without routing: they belong to the same `/24` network.

### Why a Static IP Matters for a Server
A server needs to be reachable at a **predictable, unchanging address**, because:
- **DNS records** map a domain name to a specific IP — if the IP changes, the record goes stale and the domain stops resolving correctly.
- **Firewall rules and port forwarding** on routers/firewalls are typically written against a fixed IP.
- **Other systems** (client configs, scripts, monitoring tools, other servers) often hardcode or cache the server's address.

With DHCP, a lease can expire or be reassigned — especially after a reboot — silently changing the server's IP and breaking everything that depended on the old one. A static IP removes that risk entirely: the address never moves.
