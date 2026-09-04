# UFW Firewall — Concepts You Need to Master for the Audit

This guide explains **every concept behind the 5 rules** you were given, so you can justify each open port instead of just reciting commands.

```bash
sudo ufw allow 2222/tcp        # SSH
sudo ufw allow 80/tcp          # HTTP / WordPress
sudo ufw allow 20:21/tcp       # FTP control
sudo ufw allow 40000:50000/tcp # FTP passive range
sudo ufw enable
sudo ufw status verbose
```

---

## 1. What UFW actually is

- **UFW (Uncomplicated Firewall)** is a user-friendly front-end for **iptables/nftables**, the real packet-filtering engine in the Linux kernel.
- It doesn't replace iptables — it *generates* iptables rules for you and stores them so they persist across reboots.
- Default behavior (before you touch anything):
  - `deny incoming` — nothing gets in unless explicitly allowed.
  - `allow outgoing` — the server can freely reach out (updates, DNS, package installs, etc.).
- **Audit angle**: you must know and state the default policy (`sudo ufw status verbose` shows it) — this is the *baseline* on top of which your rules are exceptions.

**Core principle to repeat in the audit: "default deny, explicit allow."** Every open port is a deliberate exception, never an accident.

---

## 2. Ports, protocols, and the "why" behind each one

### a) SSH — `2222/tcp`
- SSH normally listens on **port 22**. You moved it to **2222** in `sshd_config` (`Port 2222`) — this rule must match that config exactly, or you lock yourself out.
- **Why change the port at all?** It's *not* real security (this is called **"security through obscurity"**) — it mainly reduces noise from automated bots that scan port 22 by default. You must be honest about this in the audit: it reduces log spam and low-effort scans, it does **not** stop a targeted attacker.
- **Why TCP?** SSH is connection-oriented (needs reliable, ordered delivery for an interactive shell) — TCP, not UDP.
- Real security for SSH comes from: key-based auth (disable password auth), `PermitRootLogin no`, fail2ban, etc. — mention these if asked "how do you *actually* secure SSH."

### b) HTTP — `80/tcp`
- Port 80 is the **IANA-registered standard port for HTTP**, which is why browsers connect there by default without you specifying a port.
- Required because WordPress is a web application served over HTTP by Apache/Nginx.
- **Audit trap**: if you don't have HTTPS (443) configured, be ready to explain that traffic (including WordPress admin login) is unencrypted. If the audit expects HTTPS, you'd need 443 too — but never open a port you don't actually use.

### c) FTP control — `20:21/tcp`
This is the one most people get wrong. You need to understand **active vs. passive FTP**:

- **Port 21** = the **control channel**. It's where the client sends commands (`LIST`, `RETR`, `USER`, `PASS`...) and stays open for the whole session.
- **Port 20** = the **data channel in active mode**. In active FTP, the *server* initiates a connection back to the client from port 20 to transfer files. This is why `20:21` is opened as a range.
- **Why is FTP considered risky?** It transmits credentials **in plaintext** (unless you use FTPS/SFTP). Be ready to justify why FTP was chosen over SFTP if asked — usually "required by the project spec" is the honest answer, not "it's secure."

### d) FTP passive range — `40000:50000/tcp`
- Modern FTP clients (and anything behind NAT/firewalls) use **passive mode (PASV)** instead of active mode, because active mode requires the *server* to open a new connection into the client's network — which most client-side firewalls/NATs block.
- In passive mode, the **client** initiates both connections; the server tells the client which port to connect to for data transfer, chosen from a configured range.
- **This range must be explicitly configured in `vsftpd.conf`**:
  ```
  pasv_min_port=40000
  pasv_max_port=50000
  ```
  If the UFW range and the vsftpd range don't match exactly, passive FTP transfers will silently fail (control connection works, but `LIST`/file transfer hangs or times out).
- **Audit justification**: this isn't "one port," it's a *pool* the OS/vsftpd picks from dynamically for concurrent data transfers — you must be able to explain why it's a range and not a single port, and why 10,000 ports is a deliberate (not arbitrary) sizing choice matching expected concurrent connections. A smaller range (e.g. 1000 ports) is often more defensible than a huge one — "as small as functionally necessary" is the principle.

---

## 3. `sudo ufw enable`
- Activates the firewall and makes the rules persistent across reboots (writes to `/etc/ufw/` and hooks into systemd).
- **Ordering matters**: always add your `allow` rules (especially SSH) *before* enabling, or you can lock yourself out of a remote server instantly if the default incoming policy is deny.

---

## 4. `sudo ufw status verbose`
- Shows: default policies (incoming/outgoing/routed), each rule with its **action** (ALLOW/DENY/REJECT), **direction**, and **logging level**.
- In the audit, this is your evidence — the examiner will likely run this command and expect you to explain every single line, not just the ones you remember.
- Key things to be able to read from the output:
  - Which ports are open, on which protocol (tcp/udp).
  - Whether a rule is limited to a specific IP/subnet (`ufw allow from <ip> to any port <port>`) vs. open to `Anywhere`.
  - Logging status (`ufw logging on`) — audits often expect logging enabled so intrusion attempts are traceable.

---

## 5. The audit mindset: "justify every open port"

For **each** rule, be ready to answer three questions:
1. **What service needs it, and why does that service need this exact port/range?**
2. **What happens if I close it?** (breaks SSH access / breaks WordPress / breaks FTP transfers)
3. **What's the risk of leaving it open, and what mitigates that risk?** (e.g., FTP is plaintext → mitigated by it being an internal/isolated project network; SSH port change → mitigated by key auth, not by the port number itself)

**Rule of least privilege**: never open a range wider than needed "just in case." If you don't run FTP, don't open 20:21 or the passive range — an auditor will specifically probe for unused open ports as a sign you don't understand your own configuration.

---

## 6. Quick concept glossary

| Term | Meaning |
|---|---|
| **Stateful firewall** | Tracks connection state (NEW, ESTABLISHED, RELATED) so return traffic for an allowed outbound/inbound connection is automatically permitted without a separate rule. UFW/iptables work this way. |
| **ALLOW / DENY / REJECT** | ALLOW = let through. DENY = silently drop (no response, looks like the port doesn't exist). REJECT = drop but send back an ICMP/TCP error. UFW defaults to DENY for incoming. |
| **Ephemeral/dynamic ports** | High-numbered ports (roughly 32768–60999 or configurable, like your 40000–50000) used temporarily for one connection, then freed. |
| **Well-known ports** | 0–1023, standardized by IANA (22=SSH, 80=HTTP, 443=HTTPS, 21=FTP). |
| **Active vs passive FTP** | Active = server connects back to client (port 20). Passive = client connects to server on a negotiated port from a configured range. Passive is firewall/NAT-friendly. |

---

## 7. What to rehearse before the defense

- Explain the **default deny** policy before touching any rule.
- Walk through each rule **in the order you'd add them**, and why that order matters (SSH first).
- Be able to say, unprompted, which ports you *intentionally did not* open.
- Know where each service's matching config lives (`sshd_config`, `vsftpd.conf`) and that the port numbers must agree with UFW.
- Be honest about FTP's and the port-change's security limitations rather than overselling them — auditors often ask "is this actually secure?" specifically to see if you understand the difference between *hiding* something and *securing* it.
