# Linux User Management — Concepts and Commands

## 6. User Management

This section covers Linux user creation, groups, sudo privileges, SSH public-key authentication, passwords, file ownership, file permissions, and verification.

### Requirements

| User | Authentication | Privileges |
|---|---|---|
| `luffy` | SSH key only | Sudoer |
| `zoro` | Password only | Normal user, not sudoer |

The two major security concepts are:

- **Authentication**: proving who you are.
- **Authorization**: determining what you are allowed to do.

So:

- `luffy` authenticates with an SSH key and is authorized to use `sudo`.
- `zoro` authenticates with a password and does not have `sudo` privileges.

---

# 1. Creating a User

```bash
sudo adduser luffy
```

`adduser` creates a Linux user account.

A user account has several associated properties, including:

- username
- UID (User ID)
- primary group
- home directory
- password/authentication information

Typically, the home directory is:

```text
/home/luffy
```

You can inspect the account with:

```bash
id luffy
```

Example:

```text
uid=1002(luffy) gid=1002(luffy) groups=1002(luffy)
```

## UID — User ID

Linux internally identifies users primarily by their numeric **UID** rather than by their username.

For example:

```text
luffy -> UID 1002
zoro  -> UID 1003
```

The username is the human-readable name associated with the UID.

File ownership is based on UIDs and GIDs.

---

# 2. Why `sudo` Is Used

```bash
sudo adduser luffy
```

`sudo` means that the command is executed with elevated administrative privileges.

Creating or modifying system users normally requires administrative privileges.

Conceptually:

```text
Normal user
    |
    | sudo
    v
Administrative command
```

---

# 3. Adding Luffy to the `sudo` Group

```bash
sudo usermod -aG sudo luffy
```

This modifies Luffy's group membership.

## `usermod`

`usermod` modifies an existing Linux user account.

## `-G`

`-G` specifies supplementary groups.

## `-a`

`-a` means append.

Therefore:

```bash
sudo usermod -aG sudo luffy
```

means:

> Add `luffy` to the supplementary `sudo` group without removing his existing supplementary groups.

This is important.

If Luffy already belongs to:

```text
luffy
docker
developers
```

using:

```bash
usermod -G sudo luffy
```

could replace the supplementary group list.

Using:

```bash
usermod -aG sudo luffy
```

adds `sudo` while preserving the existing memberships:

```text
luffy
docker
developers
sudo
```

### Important rule

Remember:

```text
-aG = append to supplementary groups
```

---

# 4. Groups and Authorization

Linux uses groups to organize permissions and privileges.

A user can belong to multiple groups:

```text
luffy
├── luffy
├── sudo
└── docker
```

The `sudo` group is normally configured through `/etc/sudoers` and/or files under:

```text
/etc/sudoers.d/
```

to allow its members to execute commands with elevated privileges.

Therefore:

```text
luffy -> member of sudo -> can use sudo
zoro  -> not member of sudo -> normal user
```

---

# 5. Creating Luffy's SSH Directory

```bash
sudo mkdir -p /home/luffy/.ssh
```

This creates:

```text
/home/luffy/.ssh
```

The `.ssh` directory is the conventional location for SSH-related files belonging to a user.

It may contain files such as:

```text
/home/luffy/.ssh/
├── authorized_keys
├── known_hosts
└── ...
```

For this requirement, the important file is:

```text
authorized_keys
```

## `mkdir`

`mkdir` means:

> Make directory.

## `-p`

`-p` means:

> Create missing parent directories and do not fail if the directory already exists.

---

# 6. SSH Public-Key Authentication

The command:

```bash
sudo cp deep_in_system_key.pub /home/luffy/.ssh/authorized_keys
```

installs the public key as an authorized key for Luffy.

SSH public-key authentication uses a **key pair**:

```text
Private key                  Public key
-----------                  ----------
deep_in_system_key           deep_in_system_key.pub
     |                              |
     |                              |
  SECRET                         SHAREABLE
```

The private key stays on the client.

The public key is placed on the server.

The private key should never be copied to the server.

---

# 7. What Is `authorized_keys`?

The SSH server checks:

```text
/home/luffy/.ssh/authorized_keys
```

to determine which public keys are authorized to authenticate as `luffy`.

Example:

```text
ssh-ed25519 AAAAC3... user@machine
```

If the SSH client possesses the corresponding private key, the SSH server can verify the client's possession of that private key.

Conceptually:

```text
CLIENT                              SERVER

Private key                         Public key
    |                                   |
    |                                   |
    +-------- SSH authentication -------+
                    |
                    v
           Server verifies proof
                    |
                    v
                 Login
```

The server does not need the private key.

---

# 8. `cp` — Copying the Public Key

```bash
sudo cp deep_in_system_key.pub /home/luffy/.ssh/authorized_keys
```

`cp` means:

> Copy a file.

Here:

```text
Source:
deep_in_system_key.pub

Destination:
/home/luffy/.ssh/authorized_keys
```

The result is that the public key becomes authorized for Luffy.

## Important consideration

`authorized_keys` can contain multiple public keys, one per line.

For example:

```text
ssh-ed25519 AAAA... laptop
ssh-ed25519 BBBB... desktop
ssh-ed25519 CCCC... server
```

Using `cp` replaces the destination file if it already exists.

Therefore, in a real server, blindly copying over an existing `authorized_keys` file could remove previously authorized keys.

---

# 9. SSH Directory Permissions

```bash
sudo chmod 700 /home/luffy/.ssh
```

`chmod` changes file or directory permissions.

Linux permissions are based on:

```text
r = read
w = write
x = execute
```

For directories, `x` means the ability to enter/traverse the directory.

## Understanding `700`

The three digits correspond to:

```text
OWNER   GROUP   OTHERS
  7       0       0
```

The values are:

```text
4 = read
2 = write
1 = execute
```

Therefore:

```text
7 = 4 + 2 + 1 = rwx
0 = ---
0 = ---
```

So:

```bash
chmod 700 /home/luffy/.ssh
```

means:

```text
Owner  -> rwx
Group  -> ---
Others -> ---
```

Only the owner should have access to the directory.

---

# 10. `authorized_keys` Permissions

```bash
sudo chmod 600 /home/luffy/.ssh/authorized_keys
```

`600` means:

```text
Owner  -> rw-
Group  -> ---
Others -> ---
```

The owner can:

- read the file
- write to the file

The group and everyone else have no permissions.

This is important because the file controls which SSH keys are authorized to log in as Luffy.

If another user could modify `authorized_keys`, they could potentially add their own public key and gain access as Luffy.

---

# 11. File Ownership

```bash
sudo chown -R luffy:luffy /home/luffy/.ssh
```

`chown` means:

> Change ownership.

The syntax is:

```bash
chown user:group file
```

Therefore:

```text
luffy:luffy
```

means:

```text
owner = luffy
group = luffy
```

## `-R`

`-R` means recursive.

It applies the ownership change to the directory and everything inside it.

So:

```bash
sudo chown -R luffy:luffy /home/luffy/.ssh
```

makes the ownership conceptually:

```text
.ssh/
owner: luffy
group: luffy

authorized_keys
owner: luffy
group: luffy
```

---

# 12. Why Ownership Matters

The SSH files should belong to the correct user.

For example:

```text
/home/luffy/.ssh
    owner -> luffy

authorized_keys
    owner -> luffy
```

If these files were incorrectly owned or writable by another user, it could create security problems.

You can verify ownership and permissions with:

```bash
ls -la /home/luffy/.ssh
```

You should see something conceptually similar to:

```text
drwx------ luffy luffy .ssh
-rw------- luffy luffy authorized_keys
```

---

# 13. Creating Zoro

```bash
sudo adduser zoro
```

This creates the `zoro` user.

During creation, `adduser` asks you to set a password.

The requirement is:

```text
zoro -> password-only
zoro -> not sudoer
```

Therefore, do NOT execute:

```bash
sudo usermod -aG sudo zoro
```

The intended result is:

```text
zoro
 |
 +-- password authentication
 |
 +-- normal user privileges
```

---

# 14. Password Authentication

When Zoro connects:

```bash
ssh zoro@server
```

the SSH server can ask for Zoro's password.

Conceptually:

```text
SSH client
    |
    | username + password
    v
SSH server
    |
    v
Authentication
    |
    v
zoro account
```

---

# 15. Key Authentication vs Password Authentication

### Luffy

```text
SSH
 |
 +-- username: luffy
 |
 +-- private key
       |
       v
server checks authorized_keys
       |
       v
authentication succeeds
```

### Zoro

```text
SSH
 |
 +-- username: zoro
 |
 +-- password
       |
       v
server verifies password
       |
       v
authentication succeeds
```

These are two different authentication mechanisms.

---

# 16. Authentication vs Authorization

This distinction is fundamental.

## Authentication

Authentication answers:

> Who are you?

Examples:

```text
Password
SSH key
MFA
Biometric authentication
```

In this assignment:

```text
luffy -> SSH key
zoro  -> password
```

## Authorization

Authorization answers:

> What are you allowed to do?

Examples:

```text
Can you use sudo?
Can you read this file?
Can you modify this directory?
Can you restart a service?
```

In this assignment:

```text
luffy -> sudo privileges
zoro  -> no sudo privileges
```

A useful way to remember it:

```text
AUTHENTICATION       AUTHORIZATION
----------------     ----------------
Who are you?        What can you do?

luffy -> SSH key    luffy -> sudo

zoro -> password    zoro -> normal user
```

---

# 17. Principle of Least Privilege

The reason Zoro is not a sudoer is related to the **principle of least privilege**.

The principle says:

> Give a user only the permissions necessary to perform their required tasks.

A normal user might be able to:

```bash
ls
cd
cat
mkdir
```

A sudo-capable user can potentially perform administrative operations such as:

```bash
sudo systemctl ...
sudo useradd ...
sudo passwd ...
```

Therefore, unnecessary sudo access increases the potential impact of a compromised account.

The intended model is:

```text
luffy -> administrator
zoro  -> ordinary user
```

---

# 18. Linux File Permission Model

Linux permissions are generally represented using three categories:

```text
OWNER     GROUP     OTHERS
  |         |         |
  v         v         v
 rwx       rwx       rwx
```

For example:

```text
-rw-------
```

means:

```text
Owner  -> rw-
Group  -> ---
Others -> ---
```

And:

```text
drwx------
```

means:

```text
Directory
Owner  -> rwx
Group  -> ---
Others -> ---
```

The leading `d` indicates a directory.

---

# 19. Permission Numbers

The numeric permission system is based on:

```text
4 = read
2 = write
1 = execute
```

Add the values together:

```text
7 = 4 + 2 + 1 = rwx
6 = 4 + 2     = rw-
5 = 4 + 1     = r-x
4 = 4         = r--
3 = 2 + 1     = -wx
2 = 2         = -w-
1 = 1         = --x
0 =            ---
```

Therefore:

```text
700
```

means:

```text
Owner  -> rwx
Group  -> ---
Others -> ---
```

And:

```text
600
```

means:

```text
Owner  -> rw-
Group  -> ---
Others -> ---
```

---

# 20. Verifying Group Membership

The requirement includes:

```bash
groups luffy
groups zoro
```

This checks group membership.

Expected output should show that:

```text
luffy -> sudo
zoro  -> not sudo
```

For example:

```text
luffy : luffy sudo
```

and:

```text
zoro : zoro
```

The exact output can vary because systems may have additional groups.

You can also use:

```bash
id luffy
id zoro
```

`id` gives more detailed information, including:

- UID
- primary GID
- supplementary groups

---

# 21. Verifying the Home Directory

The requirement includes:

```bash
echo ~
```

`~` is a shell shortcut representing the current user's home directory.

As Luffy:

```bash
luffy@server:~$ echo ~
/home/luffy
```

As Zoro:

```bash
zoro@server:~$ echo ~
/home/zoro
```

Therefore:

```text
~ as luffy -> /home/luffy
~ as zoro  -> /home/zoro
```

This verifies that each user has the expected home directory.

---

# 22. Group Changes and Existing Sessions

After:

```bash
sudo usermod -aG sudo luffy
```

an already-existing Luffy login session might not immediately have the new group membership.

A new login session normally picks up the updated groups.

Therefore, after changing group membership, it is often useful to:

```bash
exit
```

and log in again.

Then:

```bash
groups
```

should show the updated membership.

---

# 23. Complete Configuration

## Luffy

Create the user:

```bash
sudo adduser luffy
```

Add sudo privileges:

```bash
sudo usermod -aG sudo luffy
```

Create SSH directory:

```bash
sudo mkdir -p /home/luffy/.ssh
```

Install the public key:

```bash
sudo cp deep_in_system_key.pub /home/luffy/.ssh/authorized_keys
```

Restrict the SSH directory:

```bash
sudo chmod 700 /home/luffy/.ssh
```

Restrict the authorized keys file:

```bash
sudo chmod 600 /home/luffy/.ssh/authorized_keys
```

Set correct ownership:

```bash
sudo chown -R luffy:luffy /home/luffy/.ssh
```

Result:

```text
luffy
 |
 +-- SSH key authentication
 |
 +-- sudo privileges
 |
 +-- /home/luffy
      |
      +-- .ssh
           |
           +-- authorized_keys
```

---

# 24. Zoro

Create the user:

```bash
sudo adduser zoro
```

Set a password when prompted.

Do NOT add Zoro to the sudo group.

Result:

```text
zoro
 |
 +-- password authentication
 |
 +-- no sudo privileges
 |
 +-- /home/zoro
```

---

# 25. Verification Checklist

Run:

```bash
groups luffy
groups zoro
```

Check that:

```text
luffy -> sudo
zoro  -> no sudo
```

Check home directories:

```bash
echo ~
```

when logged in as each user.

Check Luffy's SSH configuration:

```bash
ls -la /home/luffy/.ssh
```

Check ownership:

```bash
ls -ld /home/luffy/.ssh
ls -l /home/luffy/.ssh/authorized_keys
```

Check account information:

```bash
id luffy
id zoro
```

---

# 26. Security Architecture

The final configuration can be visualized as:

```text
                         SERVER
                           |
             +-------------+-------------+
             |                           |
           LUFFY                        ZORO
             |                           |
        SSH key only                Password only
             |                           |
      Authentication               Authentication
             |                           |
             v                           v
       luffy account                zoro account
             |                           |
             v                           v
        sudo group                 normal user
             |                           |
             v                           v
     Administrative access        Limited privileges
```

The filesystem side looks like:

```text
/home/
|
+-- luffy/
|    |
|    +-- .ssh/
|         |
|         +-- authorized_keys
|
+-- zoro/
```

With the important SSH permissions:

```text
/home/luffy/.ssh
    permissions -> 700
    owner       -> luffy

authorized_keys
    permissions -> 600
    owner       -> luffy
```

---

# 27. Audit Questions and Answers

### Why is Luffy a sudoer?

Luffy is added to the `sudo` group with:

```bash
sudo usermod -aG sudo luffy
```

The sudo group is configured to allow its members to execute commands with elevated privileges.

### Why use `-aG`?

`-G` specifies supplementary groups, while `-a` appends the new group instead of replacing existing supplementary groups.

### Why does Luffy have `.ssh`?

`.ssh` is the conventional directory for a user's SSH configuration and authentication files.

### What is `authorized_keys`?

It contains public SSH keys authorized to authenticate as that user.

### Where is the private key?

The private key stays on the SSH client. It should remain secret.

### Why is `authorized_keys` set to `600`?

Only the owner should be able to read or modify the file because it controls which keys can authenticate as the user.

### Why is `.ssh` set to `700`?

Only the owner should be able to access the SSH directory.

### Why use `chown luffy:luffy`?

It makes Luffy the owner and group owner of the SSH directory and its contents.

### Why isn't Zoro in the sudo group?

Because Zoro is intended to be an ordinary, unprivileged user. This follows the principle of least privilege.

### What is authentication?

Authentication verifies the identity of a user.

### What is authorization?

Authorization determines what an authenticated user is allowed to do.

### What authentication method does Luffy use?

SSH public-key authentication.

### What authentication method does Zoro use?

Password authentication.

### What does `~` mean?

`~` represents the current user's home directory.

### What does `-R` mean?

Recursive; apply the operation to the directory and its contents.

### What does `700` mean?

```text
Owner  -> rwx
Group  -> ---
Others -> ---
```

### What does `600` mean?

```text
Owner  -> rw-
Group  -> ---
Others -> ---
```

---

# 28. Core Mental Model

The entire section can be reduced to four concepts:

```text
USER
 |
 +-- IDENTITY
 |     |
 |     +-- UID
 |
 +-- AUTHENTICATION
 |     |
 |     +-- SSH key
 |     +-- Password
 |
 +-- AUTHORIZATION
 |     |
 |     +-- Groups
 |     +-- File permissions
 |     +-- sudo
 |
 +-- HOME
       |
       +-- /home/username
```

The commands correspond to these concepts:

```text
adduser
    -> create the user's identity

usermod -aG sudo
    -> grant administrative authorization

mkdir ~/.ssh
    -> create SSH configuration directory

cp public_key authorized_keys
    -> authorize an SSH public key

chmod
    -> restrict filesystem permissions

chown
    -> assign correct ownership

groups / id
    -> verify group membership

echo ~
    -> verify the user's home directory
```

The main security principles demonstrated are:

1. **Authentication** — prove who the user is.
2. **Authorization** — determine what the user can do.
3. **Least privilege** — do not give unnecessary administrative access.
4. **Separation of users** — each user has an independent account and home directory.
5. **Secure SSH configuration** — protect `.ssh` and `authorized_keys`.
6. **Correct ownership** — SSH configuration belongs to the intended user.
