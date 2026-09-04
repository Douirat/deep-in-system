# SSH — Authentication, Keys, and Hardening

## 1. What is SSH?

**SSH (Secure Shell)** provides encrypted remote access to another machine.

There are two sides:

```text
SSH CLIENT                         SSH SERVER
(your laptop)                      (VM)

ssh                                sshd
                                      │
                                      └── /etc/ssh/sshd_config
```

* `ssh` = client
* `sshd` = server/daemon
* `/etc/ssh/sshd_config` = SSH server configuration
* `~/.ssh/` = SSH files belonging to a user

---

# 2. Password Authentication

The traditional authentication model is:

```text
Client                         Server

username ───────────────────►
password ───────────────────►
                                │
                                ▼
                         verify password
                                │
                         ┌──────┴──────┐
                         │             │
                       valid         invalid
                         │             │
                       LOGIN         REJECT
```

The relevant SSH configuration is:

```text
PasswordAuthentication yes
```

To disable password authentication:

```text
PasswordAuthentication no
```

---

# 3. SSH Key Authentication

SSH can authenticate using a **cryptographic key pair** instead of a password.

One command generates two keys:

```bash
ssh-keygen -t ed25519
```

Or, with a custom filename:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/deep_in_system_key
```

This creates:

```text
~/.ssh/
├── deep_in_system_key
└── deep_in_system_key.pub
```

## Private key

```text
deep_in_system_key
```

* Secret
* Must remain on your client
* Never send it to the server
* Never publish it
* Used to prove that you possess the key

## Public key

```text
deep_in_system_key.pub
```

* Not secret
* Can be copied to servers
* Goes into the user's `authorized_keys`

---

# 4. The Key Authentication Model

The basic architecture is:

```text
YOUR COMPUTER                         SERVER

private key 🔐                       public key 🔓
     │                                    │
     │                                    │
     │                            ~/.ssh/authorized_keys
     │                                    │
     └──── cryptographic proof ───────────┘
                     │
                     ▼
                  LOGIN
```

The private key is **not sent to the server**.

Instead, SSH proves that the client possesses the private key corresponding to a public key that the server trusts.

The important principle is:

> You don't send the secret; you prove possession of the secret.

---

# 5. `authorized_keys`

On the server, a user's public keys are normally stored in:

```text
~/.ssh/authorized_keys
```

For example, for user `luffy`:

```text
/home/luffy/.ssh/authorized_keys
```

It can contain:

```text
ssh-ed25519 AAAAC3... luffy-key
```

The server uses this file as a list of trusted public keys.

If the client proves possession of the corresponding private key:

```text
public key in authorized_keys
        +
matching private key
        ↓
authentication succeeds
```

---

# 6. Installing the Public Key

From the client, a convenient command is:

```bash
ssh-copy-id -i ~/.ssh/deep_in_system_key.pub user@SERVER
```

This normally requires the user's password initially.

The command copies the **public key** to:

```text
/home/user/.ssh/authorized_keys
```

After that, key authentication can be used.

---

# 7. Using a Custom Private Key

Because the key is called:

```text
deep_in_system_key
```

rather than a default name such as `id_ed25519`, you can explicitly tell SSH which key to use:

```bash
ssh -i ~/.ssh/deep_in_system_key user@SERVER
```

Important:

```text
-i ~/.ssh/deep_in_system_key
```

uses the **private key**.

Do NOT use:

```bash
ssh -i ~/.ssh/deep_in_system_key.pub
```

The `.pub` file is the public key and belongs in `authorized_keys`.

---

# 8. SSH Agent

`ssh-agent` can hold your unlocked private keys.

Check which keys are loaded:

```bash
ssh-add -l
```

Add a key:

```bash
ssh-add ~/.ssh/deep_in_system_key
```

The architecture becomes:

```text
Your computer
      │
      ▼
 ssh-agent
      │
      └── private key 🔐
             │
             ▼
        SSH authentication
             │
             ▼
           Server
```

This explains why you can close an SSH session and reconnect without specifying:

```bash
-i ~/.ssh/deep_in_system_key
```

The **SSH session** ended, but the **SSH agent** is still running and has the key loaded.

Check the agent:

```bash
ssh-add -l
```

---

# 9. Two Different Types of SSH Keys

Do not confuse these two systems.

## Server host keys

Located around:

```text
/etc/ssh/ssh_host_*
```

They identify the **server**.

The client remembers server identities in:

```text
~/.ssh/known_hosts
```

Purpose:

> "Is this really the server I connected to before?"

## User authentication keys

Client:

```text
~/.ssh/deep_in_system_key
~/.ssh/deep_in_system_key.pub
```

Server:

```text
~/.ssh/authorized_keys
```

Purpose:

> "Is this user authorized to log in?"

So:

```text
SERVER HOST KEY
    ↓
Is this really my server?

USER AUTHENTICATION KEY
    ↓
Is this user authorized?
```

---

# 10. `sshd_config`

The SSH server configuration is:

```text
/etc/ssh/sshd_config
```

Many lines may begin with:

```text
#
```

A `#` means the line is commented out.

For example:

```text
#Port 22
```

does not explicitly configure the port.

SSH uses the default or another configuration source.

The file can also contain:

```text
Include /etc/ssh/sshd_config.d/*.conf
```

Therefore, configuration may also exist in:

```text
/etc/ssh/sshd_config.d/
```

Check it with:

```bash
sudo ls -la /etc/ssh/sshd_config.d/
```

---

# 11. Important SSH Authentication Directives

## `PubkeyAuthentication`

Controls public-key authentication:

```text
PubkeyAuthentication yes
```

Allows key-based authentication.

```text
PubkeyAuthentication no
```

Disables it.

---

## `PasswordAuthentication`

Controls password authentication:

```text
PasswordAuthentication yes
```

allows passwords.

```text
PasswordAuthentication no
```

disables passwords.

---

## `PermitRootLogin`

Controls SSH login as root.

For example:

```text
PermitRootLogin no
```

means:

```text
ssh root@server
        ↓
      REJECT
```

This does not disable the root account itself. It only prevents SSH login as root.

---

# 12. `Match User`

`Match User` allows different SSH rules for different users.

Example:

```text
Match User luffy
    PasswordAuthentication no
    PubkeyAuthentication yes
```

For `luffy`:

```text
password → ✗
SSH key  → ✓
```

Another user can have a completely different policy:

```text
Match User zoro
    PasswordAuthentication yes
    PubkeyAuthentication no
```

For `zoro`:

```text
password → ✓
SSH key  → ✗
```

This is the central concept of the project.

---

# 13. Project Configuration

The project asks for:

```text
Port 2222
PermitRootLogin no

Match User luffy
    PasswordAuthentication no
    PubkeyAuthentication yes

Match User zoro
    PasswordAuthentication yes
    PubkeyAuthentication no
```

The resulting policy is:

```text
                         SSH SERVER
                             │
                 ┌───────────┼───────────┐
                 │           │           │
               luffy        zoro        root
                 │           │           │
              key only   password only    ✗
                 │           │
                 ✓           ✓
```

The SSH server listens on:

```text
TCP 2222
```

instead of the standard:

```text
TCP 22
```

---

# 14. Connecting to Port 2222

Because SSH is now listening on port `2222`:

```bash
ssh -p 2222 luffy@SERVER_IP
```

The `-p` option specifies the SSH port.

For a specific private key:

```bash
ssh -p 2222 \
    -i ~/.ssh/deep_in_system_key \
    luffy@SERVER_IP
```

---

# 15. Testing the Project

## Luffy — key authentication

```bash
ssh -p 2222 \
    -i ~/.ssh/deep_in_system_key \
    luffy@SERVER_IP
```

Expected:

```text
✓ LOGIN
```

## Luffy — password authentication

To explicitly test password authentication:

```bash
ssh -o PubkeyAuthentication=no \
    -p 2222 \
    luffy@SERVER_IP
```

Expected:

```text
✗ REJECT
```

This is important if your SSH agent already has your key loaded.

---

## Zoro — password authentication

```bash
ssh -p 2222 zoro@SERVER_IP
```

Expected:

```text
✓ LOGIN with password
```

## Zoro — key authentication

```bash
ssh -o PasswordAuthentication=no \
    -p 2222 \
    -i ~/.ssh/deep_in_system_key \
    zoro@SERVER_IP
```

Expected:

```text
✗ REJECT
```

---

## Root

```bash
ssh -p 2222 root@SERVER_IP
```

Expected:

```text
✗ REJECT
```

because:

```text
PermitRootLogin no
```

---

# 16. SSH Configuration Validation

Before restarting SSH, always check the configuration:

```bash
sudo sshd -t
```

If there is no output, the configuration syntax is valid.

You can inspect the effective configuration with:

```bash
sudo sshd -T
```

This is useful because the final configuration can be affected by:

```text
sshd_config
      +
sshd_config.d/*.conf
      +
SSH defaults
      ↓
effective configuration
```

---

# 17. Restarting SSH

After validating:

```bash
sudo sshd -t
```

restart SSH:

```bash
sudo systemctl restart ssh
```

When changing SSH configuration:

> **Keep your existing SSH session open until you have successfully tested a new connection.**

If you make a mistake and restart SSH, your current connection may remain alive while new connections fail.

---

# 18. SSH Debugging

Use:

```bash
ssh -v user@SERVER
```

for verbose output.

For example:

```bash
ssh -v -p 2222 luffy@SERVER_IP
```

You may see messages indicating that SSH is:

```text
Offering public key
Server accepts key
Authenticated
```

This helps determine which authentication method is actually being used.

---

# 19. Important Commands

### Generate a key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/deep_in_system_key
```

### Display public key

```bash
cat ~/.ssh/deep_in_system_key.pub
```

### Copy public key to server

```bash
ssh-copy-id -i ~/.ssh/deep_in_system_key.pub user@SERVER
```

### Add key to agent

```bash
ssh-add ~/.ssh/deep_in_system_key
```

### List keys in agent

```bash
ssh-add -l
```

### Connect with a specific key

```bash
ssh -i ~/.ssh/deep_in_system_key user@SERVER
```

### Connect to a specific port

```bash
ssh -p 2222 user@SERVER
```

### Validate SSH configuration

```bash
sudo sshd -t
```

### Show effective configuration

```bash
sudo sshd -T
```

### Restart SSH

```bash
sudo systemctl restart ssh
```

### Check SSH service

```bash
sudo systemctl status ssh
```

### View SSH logs

```bash
sudo journalctl -u ssh
```

---

# 20. Concepts to Learn

For this project, understand these concepts rather than simply memorizing commands:

### Core

* SSH client vs SSH server
* `ssh` vs `sshd`
* TCP ports
* `/etc/ssh/sshd_config`
* `sshd_config.d`
* password authentication
* public-key authentication
* private key
* public key
* `authorized_keys`
* `known_hosts`
* server host keys
* user authentication keys

### Key management

* `ssh-keygen`
* `ssh-copy-id`
* `ssh-agent`
* `ssh-add`
* key passphrases
* SSH key permissions

### SSH hardening

* `PasswordAuthentication`
* `PubkeyAuthentication`
* `PermitRootLogin`
* `Match User`
* `AllowUsers`
* `AllowGroups`
* `MaxAuthTries`
* `MaxSessions`
* `AllowTcpForwarding`
* `X11Forwarding`

### Administration

* `sshd -t`
* `sshd -T`
* `systemctl`
* `journalctl`
* SSH verbose mode (`ssh -v`)

---

# 21. The Complete Mental Model

```text
                         SSH CONNECTION
                               │
                               ▼
                         SSH SERVER
                            sshd
                               │
                     /etc/ssh/sshd_config
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
            Which user?                Which port?
                 │                           │
          ┌──────┼──────┐                  2222
          │      │      │
        luffy   zoro   root
          │      │      │
          ▼      ▼      ▼
        Match   Match   denied
        rules   rules
          │      │
          │      │
       key only password only
          │      │
          ✓      ✓
```

The fundamental key-authentication model is:

```text
                 CLIENT
                   │
          private key 🔐
                   │
                   │ proves possession
                   ▼
                 SERVER
                   │
          authorized_keys
                   │
          public key 🔓
                   │
                   ▼
              authentication
                   │
                   ▼
                 LOGIN
```

The most important rule to remember:

> **The private key stays with the client. The public key goes to the server.**
