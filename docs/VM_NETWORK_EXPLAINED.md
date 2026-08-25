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

## The host's side: `ip a` on your ThinkPad

This is the crucial piece the guest output alone doesn't show. Running `ip a` on your **host machine** (bendoe-ThinkPad-T590) gives:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp0s31f6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
    link/ether 00:2b:67:20:9c:90 brd ff:ff:ff:ff:ff:ff
3: wlp0s20f3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 74:d8:3e:b0:6a:17 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.111/24 brd 192.168.1.255 scope global dynamic noprefixroute wlp0s20f3
       valid_lft 82796sec preferred_lft 82796sec
    inet6 fe80::8460:c6f2:531c:b610/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
4: wwan0: <BROADCAST,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 46:c7:11:76:27:d0 brd ff:ff:ff:ff:ff:ff
5: br-2d9ecff5397a: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 9a:81:f4:78:1a:63 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-2d9ecff5397a
       valid_lft forever preferred_lft forever
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 4a:d7:b6:0b:6a:8a brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

### Reading the host's interfaces

| Interface | State | What it is |
|---|---|---|
| `lo` | — | Same loopback story as the guest, irrelevant to VM reachability. |
| `enp0s31f6` | `NO-CARRIER` / `DOWN` | The ThinkPad's built-in Ethernet port. Physically up (cable unplugged or no link), no IP assigned. Not in use. |
| `wlp0s20f3` | `UP`, has an IP | **This is the host's real, active connection** — Wi-Fi, address `192.168.1.111/24`. This is the subnet your host actually lives on. |
| `wwan0` | `DOWN` | A mobile broadband/WWAN modem interface, unused. |
| `br-2d9ecff5397a`, `docker0` | `DOWN`, but with IPs `172.18.0.1/16` and `172.17.0.1/16` | Docker's virtual bridge networks, created automatically by the Docker daemon for container-to-container and container-to-host networking. Unrelated to the VM entirely — these exist because Docker is installed on this host, not because of VirtualBox. |

### Why this is the piece that makes NAT vs. Bridged click

Now the comparison from the earlier section has concrete numbers behind it:

- **Guest (VM), NAT mode:** `10.0.2.15/24` — VirtualBox's private, internal-only NAT segment.
- **Host, real network:** `192.168.1.111/24` — the actual LAN, reachable by other devices on your Wi-Fi (router, phone, etc.).

These are two *completely different, non-routing subnets*. That's the whole reason `ssh luffy@10.0.2.15` from the host fails outright (`10.0.2.0/24` doesn't exist as far as the host's routing table is concerned — it's a network that only exists *inside* VirtualBox's NAT engine).

If you switch the VM to **Bridged mode** (Option B below), the guest's `enp0s3` would instead pick up an address in the **same `192.168.1.0/24` block as `wlp0s20f3`** — e.g. `192.168.1.150` — because it would be attached directly to your Wi-Fi's network segment via your host's wireless adapter, rather than to VirtualBox's isolated NAT network. At that point the guest and host are just two ordinary machines on the same LAN, and plain `ssh -p 2222 luffy@192.168.1.150` works with no forwarding rule at all.

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
