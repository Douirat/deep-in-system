# Understanding the UFW Rules for FTP (Control + Passive Mode)

```bash
sudo ufw allow 20:21/tcp       # FTP control
sudo ufw allow 40000:50000/tcp # FTP passive range (must match vsftpd config)
```

This document explains **all the theoretical concepts** you need to understand why these two rules exist and how they work together.

---

## 1. FTP uses TWO channels

Unlike HTTP (a single port, 80/443), FTP splits communication into two separate connections:

| Channel | Role | Default port |
|---|---|---|
| **Control channel** (commands) | Sends commands (`USER`, `PASS`, `LIST`, `RETR`, `STOR`...) and receives server responses | **21** |
| **Data channel** (data) | Actually transfers files and directory listings | Dynamic (negotiated) |

Port 20 is historically associated with the data channel in **active mode** (see below), which is why the first rule uses the range `20:21`.

---

## 2. Active mode vs Passive mode

This is the most important concept to understand here.

### Active mode (FTP Active)
- The **client** opens the control channel to the server's port 21.
- For data transfer, the **server** initiates a connection *back to* the client (from its own port 20).
- Problem: if the client is behind NAT/a firewall (basically always true today), the server can't initiate an inbound connection to it -> **transfer fails**.

### Passive mode (FTP Passive) -- the one used here
- The client opens the control channel to port 21.
- The client then asks the server: *"give me a port for data"* (the `PASV` command).
- The server replies with a random port chosen from a **predefined range**.
- The client itself initiates the data connection to that port.
- Advantage: works much better with modern NAT/firewalls, because it's always the client initiating connections.

This is why passive mode is the standard today, and why the server side needs a **port range** opened (`40000:50000` in the example).

---

## 3. Why a port range (`40000:50000`) instead of a single port?

- Each passive data connection needs **its own port**, randomly picked by the server from the configured range.
- If multiple clients connect at once (or the same client does several transfers), each uses a different port from the range.
- This range is set on the FTP server side (e.g. `vsftpd`) via parameters like:
  ```
  pasv_min_port=40000
  pasv_max_port=50000
  ```
- **The firewall (ufw) must allow the exact same range**, otherwise the server may pick a data port the firewall blocks -> the transfer "hangs" right after authentication (classic symptom: you can log in and see `login successful`, but `LIST` or a file transfer just stalls).

---

## 4. UFW (Uncomplicated Firewall) -- the basics you need

- UFW is a simplified frontend for `iptables` on Linux (Ubuntu/Debian).
- `sudo ufw allow PORT/tcp` adds a rule allowing **inbound** traffic on that port over TCP.
- The syntax `20:21/tcp` means *"allow all ports from 20 to 21 inclusive, over TCP"*.
- FTP runs exclusively over TCP (never UDP), hence the `/tcp` suffix.
- Without these explicit rules, UFW blocks all unrecognized inbound traffic by default -- so even if `vsftpd` is running fine, connections will fail until the firewall lets the right ports through.

---

## 5. The link between the firewall and the FTP server config

This is the critical point behind the comment in the code: **"must match vsftpd config"**.

There's a chain of consistency to maintain:

```
vsftpd config (pasv_min_port / pasv_max_port)
              <->  (must be identical)
UFW rule (allowed port range)
```

If these two don't match:
- The server might pick a passive port the firewall blocks.
- Result: the client connects and authenticates fine, but file transfers fail or time out.

---

## 6. Related concepts worth knowing

- **NAT (Network Address Translation)**: translation between private/public IP addresses -- the root problem passive mode solves.
- **TCP vs UDP**: FTP is TCP-based because it needs reliable, ordered transmission (unlike UDP).
- **FTPS / SFTP**: secure alternatives to plain FTP (FTPS = FTP + TLS; SFTP = a different protocol built on SSH). Worth knowing the difference if transfer security matters.
- **Privileged ports (<1024)**: port 21 is a reserved port requiring root privileges to bind, which is why `sudo` is needed.

---

## 7. Visual summary of the full passive flow

```
Client                                FTP Server
  |--- TCP connection to port 21 ---->|   (control channel)
  |<-- "220 Service ready" -----------|
  |--- USER / PASS ------------------>|
  |--- PASV command ------------------>|
  |<-- "227 Entering Passive Mode      |
  |     (ip, port picked from 40000-50000)"
  |--- new TCP connection to the      |
  |    given port -------------------->|   (data channel)
  |<-- file transfer / listing --------|
```

---

## Key takeaways

1. FTP = 2 channels (control + data), unlike a single-port protocol like HTTP.
2. Passive mode is preferred today because it's NAT/firewall-friendly on the client side.
3. The passive port range must be **identical** between `vsftpd` and `ufw`.
4. Without a matching UFW rule, data transfer fails even if authentication succeeds.