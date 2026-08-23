# VM Network Configuration — Explained

This document explains the output of `ip a` run inside the VirtualBox guest, and how it connects to the SSH setup in `GUIDE.md`.

## The raw output

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc ...
    link/ether 08:00:27:cc:35:30 brd ff:ff:ff:ff:ff:ff
    altname enx080027cc3530
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 82922sec preferred_lft 82922sec
    inet6 2c37:6a5c:1687:2:a66:27ff:fecc:3530/64 scope global dynamic mngtmpaddr
       valid_lft 86368sec preferred_lft 14368sec
    inet6 fe80::a00:27ff:fecc:3530/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
```

## Interface 1: `lo` (loopback)

- `127.0.0.1/8` and `::1/128` are the IPv4/IPv6 loopback addresses.
- Loopback traffic never leaves the machine — it's how a process talks to another process on the *same* host (e.g. a local database on `127.0.0.1:5432`).
- Not relevant to reaching the VM from outside; every Linux system has this interface, VM or not.

## Interface 2: `enp0s3` (the VM's network adapter)

`enp0s3` is the predictable network interface name (systemd naming scheme: **en**=ethernet, **p0**=PCI bus 0, **s3**=slot 3) for the VM's virtual NIC.

### MAC address

`08:00:27:cc:35:30` — the prefix `08:00:27` is Oracle VirtualBox's registered OUI (Organizationally Unique Identifier). Any NIC with this prefix was created by VirtualBox, which confirms this is a VirtualBox virtual adapter and not a physical one.

### IPv4 address: the key detail

`inet 10.0.2.15/24` is the address VirtualBox assigns **by default** to a guest's first NIC when that NIC is in **NAT mode** (the out-of-the-box default for new VMs). This is a strong, recognizable signature:

- VirtualBox's internal NAT network is always `10.0.2.0/24`.
- The guest always gets `10.0.2.15`.
- The (invisible here, but implied) gateway is always `10.0.2.2` — that's actually the *host* machine, as seen from inside the NAT'd VM.
- DNS is proxied through `10.0.2.3`.

**Why this matters:** NAT mode is a one-way street. The guest can reach *out* to the internet (which is how your earlier `git push` succeeded), but nothing outside the VM — including your host machine — can initiate a connection *into* the guest, because `10.0.2.15` isn't routable from anywhere except inside that VM's private NAT segment.

### IPv6 addresses

- `2c37:6a5c:1687:2:a66:27ff:fecc:3530/64` — a global-scope IPv6 address, auto-configured via SLAAC (`dynamic mngtmpaddr`). Notice the last 4 bytes (`a66:27ff:fecc:3530`) are derived from the MAC address — this is the EUI-64 interface identifier construction.
- `fe80::a00:27ff:fecc:3530/64` — the link-local address, only usable for communication with directly-connected neighbors on the same segment (`scope link`). Every IPv6 interface has one of these automatically; it's the IPv6 analog of "not globally useful."

Neither IPv6 address changes the NAT conclusion above — VirtualBox's default adapter is IPv4 NAT regardless of what IPv6 addressing shows up alongside it.

## Why this breaks the SSH steps in GUIDE.md

Your guide sets:

```
Port 2222
PasswordAuthentication no   (for luffy)
PubkeyAuthentication yes    (for luffy)
```

and expects you to `ssh` in from your host machine using a generated key. Under the current network mode, an `ssh -p 2222 <ip>` from your host **will not connect**, because `10.0.2.15` is not reachable from the host — only from processes running inside the guest itself.

## Fixing it: two options

### Option A — Port Forwarding (keep NAT mode)

Best if you don't need the VM to be reachable from your wider LAN, just from your own host.

1. VirtualBox Manager → select the VM → **Settings → Network → Adapter 1 → Advanced → Port Forwarding**.
2. Add a rule:
   | Name | Protocol | Host IP | Host Port | Guest IP | Guest Port |
   |------|----------|---------|-----------|----------|------------|
   | SSH  | TCP      | (blank) | 2222      | 10.0.2.15 | 2222      |
3. From the host: `ssh -p 2222 luffy@127.0.0.1`

This works because VirtualBox's NAT engine listens on the host's `127.0.0.1:2222` and silently relays traffic into the guest's `10.0.2.15:2222`.

### Option B — Switch to Bridged (or Host-only) networking

Best if you want the VM to behave like a real machine on your LAN, with its own IP reachable directly.

1. VM must be powered off.
2. **Settings → Network → Adapter 1 → Attached to: Bridged Adapter** (pick your host's active physical interface).
3. Boot the VM, re-run `ip a` — you'll now see an address in your LAN's actual subnet (e.g. `192.168.1.x`) instead of `10.0.2.15`.
4. From the host: `ssh -p 2222 luffy@<that-new-address>` — no port-forwarding rule needed.

**Host-only Adapter** is a middle ground: gives the VM an IP reachable only from the host (not the wider LAN, not the internet) — useful if you want direct SSH access without exposing the VM to your network.

## Quick reference: how to tell which mode you're in

| Symptom | Mode |
|---|---|
| Guest IP is `10.0.2.15/24`, gateway `10.0.2.2` | NAT (needs port forwarding to reach from host) |
| Guest IP matches your host's LAN subnet | Bridged (directly reachable from host and LAN) |
| Guest IP is in a separate `192.168.56.x`-style subnet, unreachable from LAN | Host-only (directly reachable from host only) |
