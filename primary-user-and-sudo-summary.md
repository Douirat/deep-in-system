# Primary User & sudo — Summary

## 1. Linux Users and Privileges

Linux does not give every user the same level of access.

A normal user operates with limited privileges, while the special `root` user has very high privileges.

```text
Normal user
    │
    ├── Own files and processes
    ├── Limited access to system resources
    └── Cannot normally modify protected system areas

root (UID 0)
    │
    └── Can perform privileged system operations
```

The purpose of this separation is security and controlled access.

---

## 2. The Root User

`root` is the Linux superuser.

Its UID is:

```text
0
```

Root can perform operations that normal users normally cannot, such as:

```bash
cat /etc/shadow
systemctl restart nginx
apt install nginx
```

A root shell looks like:

```text
root@server:~#
```

The main danger is that commands executed from a root shell have high privileges by default.

A simple mistake can therefore affect the whole system.

---

## 3. Why Not Work as Root All the Time?

The key principle is **least privilege**.

> A user or process should have only the privileges necessary to perform its current task.

Most everyday operations do not require root.

For example:

```bash
nano hello.c
mkdir ~/project
touch ~/test.txt
```

These can normally be performed as a regular user.

Only operations that actually require administrative privileges should be elevated.

---

## 4. What Is `sudo`?

`sudo` allows an authorized user to execute a command with elevated privileges, normally as `root`.

Example:

```bash
sudo apt install nginx
```

Conceptually:

```text
Normal user
    │
    │ sudo
    ▼
Privileged command
```

The important point is that the entire user session does **not** become root.

After:

```bash
sudo apt install nginx
```

you are still the same user:

```bash
whoami
```

Output:

```text
luffy
```

Only the command executed through `sudo` receives elevated privileges.

---

## 5. `sudo` vs Root Shell

### Using `sudo`

```bash
luffy$ sudo systemctl restart nginx
```

The specific command is executed with elevated privileges.

Afterward:

```bash
luffy$ whoami
luffy
```

Your shell remains a normal user shell.

### Root shell

You can enter a root shell with:

```bash
sudo -i
```

Then:

```text
root@server:~#
```

Commands executed inside this shell are privileged by default.

Comparison:

```text
sudo command:

normal shell
    │
    └── privileged command
            │
            └── back to normal shell


root shell:

normal user
    │
    └── sudo -i
          │
          └── root shell
                ├── root command
                ├── root command
                └── root command
```

For normal administration, using `sudo` for individual commands keeps privilege elevation more controlled.

---

## 6. The `sudo` Group

Ubuntu commonly uses the `sudo` group to identify users who are allowed to use administrative privileges through `sudo`.

For example:

```bash
id
```

might show:

```text
uid=1000(luffy) gid=1000(luffy) groups=1000(luffy),27(sudo)
```

The important part is:

```text
27(sudo)
```

This means the user belongs to the `sudo` group.

You can also check with:

```bash
groups
```

Example:

```text
luffy sudo
```

Membership in the group gives the account access to the system's configured `sudo` policy. The exact permissions are ultimately determined by the sudo configuration.

---

## 7. Understanding `id`

Run:

```bash
id
```

Example:

```text
uid=1000(luffy) gid=1000(luffy) groups=1000(luffy),27(sudo)
```

### `uid`

```text
uid=1000(luffy)
```

`UID` means **User ID**.

Linux internally identifies users by numerical IDs.

Important example:

```text
root  → UID 0
luffy → UID 1000
```

The username is the human-readable representation.

### `gid`

```text
gid=1000(luffy)
```

`GID` means **Group ID**.

It identifies the user's primary group.

### `groups`

```text
groups=1000(luffy),27(sudo)
```

This shows all groups to which the user belongs.

---

## 8. Understanding `groups`

Run:

```bash
groups
```

Example:

```text
luffy sudo
```

This answers:

> Which groups does this user belong to?

It is a simpler version of the group information shown by `id`.

---

## 9. Authentication vs Authorization

These are two different concepts.

### Authentication

Authentication answers:

> Who are you?

Examples:

```text
Username + password
SSH key
```

### Authorization

Authorization answers:

> What are you allowed to do?

For example:

```text
luffy → allowed to use sudo
zoro  → not allowed to use sudo
```

A user can successfully authenticate to the server without having administrative privileges.

---

## 10. `sudo` Does Not Mean "Ignore Permissions"

Linux permissions still exist.

Without sufficient privileges:

```text
luffy
  │
  ▼
permission check
  │
  ▼
denied
```

With `sudo`:

```text
luffy
  │
  ▼
sudo authorization
  │
  ▼
privileged process
  │
  ▼
permission check
  │
  ▼
allowed
```

`sudo` runs the command with an identity that has the required privileges.

---

## 11. Why `sudo` Should Not Be Used Everywhere

Do not automatically add `sudo` whenever a command fails.

Bad habit:

```bash
sudo mkdir ~/project
sudo touch ~/project/file
sudo nano ~/project/file
```

If the directory belongs to your user, these commands normally do not need root privileges.

Prefer:

```bash
mkdir ~/project
touch ~/project/file
nano ~/project/file
```

Use `sudo` when the operation actually requires administrative access.

For example:

```bash
sudo mkdir /opt/myapp
```

may be necessary because `/opt` is normally a system-managed directory.

The useful question is:

> Why does this command require elevated privileges?

---

## 12. `sudo` and Accountability

Using individual user accounts together with `sudo` allows administrative actions to be associated with the user who requested them.

For example:

```text
luffy$ sudo ...
zoro$ sudo ...
```

The system can distinguish the users.

This is preferable to having everyone share a single root account:

```text
root
```

because a shared root account makes it harder to identify who performed an administrative operation.

`sudo` can also integrate with system logging, depending on the system configuration.

---

## 13. Multiple Users and Different Privileges

A server can contain several users with different responsibilities.

Example:

```text
                 Server
                   │
        ┌──────────┼──────────┐
        │          │          │
      luffy       zoro       nami
        │          │          │
       sudo      no sudo     FTP
```

This demonstrates the principle of giving each account only the capabilities it needs.

For example, if `zoro` only needs normal access, there is no reason to give `zoro` administrative privileges.

---

## 14. Why the Assignment Checks `id` and `groups`

When the guide asks you to run:

```bash
id
groups
```

it is verifying your account configuration.

You want to confirm:

1. Your expected username exists.
2. Your user has the correct UID.
3. Your primary group is correct.
4. Your user belongs to the expected groups.
5. Most importantly, your administrative user belongs to the `sudo` group.

Example:

```bash
id
```

Possible result:

```text
uid=1000(luffy) gid=1000(luffy) groups=1000(luffy),27(sudo)
```

And:

```bash
groups
```

Possible result:

```text
luffy sudo
```

---

# Key Concepts to Remember

| Concept | Meaning |
|---|---|
| User | An identity used by Linux to control access |
| UID | Numerical identifier for a user |
| Group | Collection of users used for access control |
| GID | Numerical identifier for a group |
| `root` | Superuser, UID `0` |
| `sudo` | Execute a command with elevated privileges |
| `sudo` group | Common Ubuntu group for users authorized to use sudo |
| Authentication | Establishing who you are |
| Authorization | Determining what you can do |
| Least privilege | Give only the privileges necessary for the task |
| Root shell | A shell where commands normally run with root privileges |

---

# Commands to Know

### Check your identity

```bash
whoami
```

### Display detailed identity information

```bash
id
```

### Display your groups

```bash
groups
```

### Run one command with elevated privileges

```bash
sudo <command>
```

Example:

```bash
sudo systemctl restart nginx
```

### Start a root shell

```bash
sudo -i
```

Exit it with:

```bash
exit
```

---

# The Main Idea

The whole concept can be summarized as:

```text
Normal work
     │
     ▼
Normal user privileges
     │
     │ only when necessary
     ▼
    sudo
     │
     ▼
Specific privileged command
```

The goal is **not** to avoid administrative privileges completely.

The goal is to use them **only when necessary and only for the operation that requires them**.

This is the practical application of the **principle of least privilege**.
