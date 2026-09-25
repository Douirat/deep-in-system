# Linux Users and Groups

## 1. The Big Picture

Linux uses **users** and **groups** to identify identities and organize access to system resources.

The basic relationship is:

```text
                    Linux System
                         │
                ┌────────┴────────┐
                │                 │
              Users             Groups
                │                 │
           identities        collections
                │                 │
                └────────┬────────┘
                         │
                    Permissions
                         │
             ┌───────────┼───────────┐
             │           │           │
           Files      Directories   Services
```

A useful mental model:

> **Users are identities. Groups are collections of users. Permissions control what those users/groups can do.**

---

# 2. Users

A **user** represents an identity on the Linux system.

Examples:

```text
luffy
zoro
nami
server
```

A user account is associated with information such as:

```text
Username
UID
Primary GID
Home directory
Login shell
Authentication information
Supplementary groups
```

You can inspect a user with:

```bash
id luffy
```

Example:

```text
uid=1001(luffy) gid=1001(luffy) groups=1001(luffy),27(sudo)
```

This means:

* `luffy` has UID `1001`
* `luffy`'s primary group has GID `1001`
* `luffy` also belongs to the `sudo` group

---

# 3. Groups

A **group** is a collection of users.

Groups are mainly used to organize users and manage permissions.

Create a group:

```bash
sudo groupadd developers
```

This creates:

```text
developers
```

Then users can be added to it:

```bash
sudo usermod -aG developers luffy
sudo usermod -aG developers zoro
```

The relationship becomes:

```text
Group: developers
│
├── luffy
└── zoro
```

`developers` is:

* not a user
* not a directory
* not a file

It is a **group**.

---

# 4. Users vs Groups vs Directories

It is important to distinguish these concepts.

| Name         | What it is |
| ------------ | ---------- |
| `luffy`      | User       |
| `developers` | Group      |
| `/project`   | Directory  |

They can be connected through permissions:

```text
/project
    │
    ├── owner → root
    └── group → developers
                     │
                     ├── luffy
                     └── zoro
```

---

# 5. Why Groups Exist

Imagine you have:

```text
luffy
zoro
nami
```

Suppose `luffy` and `zoro` need access to a project, but `nami` does not.

Instead of managing their permissions individually, create a group:

```bash
sudo groupadd developers
```

Add the appropriate users:

```bash
sudo usermod -aG developers luffy
sudo usermod -aG developers zoro
```

Now:

```text
developers
├── luffy
└── zoro
```

You can assign permissions to the group instead of managing each user separately.

Therefore:

> **Groups allow multiple users to share a common access policy.**

---

# 6. Groups and File Permissions

Linux permissions use three main categories:

```text
owner
group
others
```

For example:

```bash
ls -l file.txt
```

might show:

```text
-rw-r----- 1 luffy developers 1234 Sep 25 file.txt
```

The important information is:

```text
owner → luffy
group → developers
```

The permissions are:

```text
-rw-r-----
 │  │  │
 │  │  └── others
 │  └───── group
 └──────── owner
```

More precisely:

```text
owner  → rw-
group  → r--
others → ---
```

Therefore:

* `luffy`, as the owner, can read and write.
* Members of `developers` can read.
* Everyone else has no permissions.

---

# 7. Giving a Directory to a Group

Suppose you have:

```text
/project
```

You can make `developers` the group owner:

```bash
sudo chown root:developers /project
```

Now:

```text
/project
│
├── owner → root
└── group → developers
              ├── luffy
              └── zoro
```

You can then configure its permissions:

```bash
sudo chmod 770 /project
```

The permission structure is:

```text
owner  → rwx
group  → rwx
others → ---
```

So the owner and members of `developers` have access according to those permissions, while others do not.

---

# 8. The `sudo` Group

`sudo` is an important example of group-based access.

`sudo` itself is a command/tool used to execute commands with elevated privileges according to the system's sudo policy.

On Ubuntu, membership in the `sudo` group normally allows a user to use `sudo`.

For example:

```bash
sudo usermod -aG sudo luffy
```

This means:

> Add `luffy` to the `sudo` group.

The relationship becomes:

```text
luffy
  │
  └── member of sudo
              │
              └── sudo policy allows privileged commands
```

Then `luffy` may be able to run:

```bash
sudo apt update
```

depending on the system's sudo configuration.

### Important distinction

> **`sudo` is a tool. The `sudo` group is a group. Sudo policy determines what authorized users can do.**

---

# 9. Primary Group vs Supplementary Groups

A user can belong to multiple groups.

For example:

```text
luffy
│
├── primary group → luffy
│
├── supplementary group → sudo
│
└── supplementary group → developers
```

### Primary group

The user's main group identity.

### Supplementary groups

Additional groups that the user belongs to.

Check all groups:

```bash
id luffy
```

or:

```bash
groups luffy
```

---

# 10. Creating Users and Groups

### Create a group

```bash
sudo groupadd developers
```

### Create a user

```bash
sudo adduser luffy
```

### Add a user to a supplementary group

```bash
sudo usermod -aG developers luffy
```

### Add a user to the sudo group

```bash
sudo usermod -aG sudo luffy
```

### Check a user's groups

```bash
id luffy
```

---

# 11. Understanding `usermod -aG`

This command:

```bash
sudo usermod -aG developers luffy
```

contains:

```text
-a → append
-G → supplementary groups
```

So it means:

> Add `luffy` to the `developers` supplementary group without removing existing supplementary groups.

The `-a` is important.

Prefer:

```bash
sudo usermod -aG developers luffy
```

when you want to **add** a group.

Be careful with:

```bash
sudo usermod -G developers luffy
```

because without `-a`, you can replace the user's existing supplementary group memberships.

---

# 12. Example From Start to Finish

Let's build a small project.

## Step 1 — Create users

```bash
sudo adduser luffy
sudo adduser zoro
sudo adduser nami
```

We now have:

```text
Users
├── luffy
├── zoro
└── nami
```

## Step 2 — Create a group

```bash
sudo groupadd developers
```

Now:

```text
Group
└── developers
```

## Step 3 — Add users to the group

```bash
sudo usermod -aG developers luffy
sudo usermod -aG developers zoro
```

Now:

```text
developers
├── luffy
└── zoro
```

`nami` is not in the group.

## Step 4 — Create a project directory

```bash
sudo mkdir /project
```

## Step 5 — Assign the group

```bash
sudo chown root:developers /project
```

Now:

```text
/project
├── owner → root
└── group → developers
```

## Step 6 — Configure permissions

```bash
sudo chmod 770 /project
```

Conceptually:

```text
/project

root
└── rwx

developers
├── luffy → rwx
└── zoro  → rwx

others
└── nami  → ---
```

We did not need to assign permissions individually to `luffy` and `zoro`.

We assigned permissions to:

```text
developers
```

Because `luffy` and `zoro` belong to that group, the group permissions apply to them.

---

# 13. Important Terminology

## User

An identity/account on the Linux system.

Example:

```text
luffy
```

## Group

A collection of users used to organize access and permissions.

Example:

```text
developers
```

## UID

**User ID** — the numerical identifier of a user.

Example:

```text
uid=1001(luffy)
```

## GID

**Group ID** — the numerical identifier of a group.

Example:

```text
gid=1001(luffy)
```

## Primary group

The main group associated with a user.

## Supplementary group

An additional group that a user belongs to.

## Owner

The user associated with a file or directory.

## Group owner

The group associated with a file or directory.

## Others

Everyone who is neither the file owner nor a member of the relevant group.

---

# 14. The Core Permission Model

Remember this model:

```text
                    USER
                      │
             belongs to groups
                      │
                      ▼
                   GROUP
                      │
                      ▼
              File permissions
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
        OWNER       GROUP       OTHERS
```

When Linux checks access to a file or directory, it determines which permission category applies to the user:

1. Is the user the owner?
2. If not, does the user belong to the relevant group?
3. Otherwise, the `others` permissions apply.

---

# 15. The Most Important Mental Model

Remember these three concepts:

```text
USER
  ↓
identity
```

```text
GROUP
  ↓
collection of users
```

```text
PERMISSIONS
  ↓
what the owner, group, and others can do
```

Together:

```text
                USERS
             /    |    \
          luffy  zoro  nami
             \     /
              \   /
              GROUP
           developers
                 │
                 ▼
             /project
                 │
            permissions
```

So if you see:

```text
luffy
developers
/project
```

think:

```text
luffy        → user
developers   → group
/project     → directory
```

And if:

```text
luffy ∈ developers
```

then permissions assigned to the `developers` group can apply to `luffy`.

---

# 16. Final Summary

The fundamental relationship is:

```text
Users
  ↓
belong to
  ↓
Groups
  ↓
are used when managing
  ↓
Permissions
  ↓
on
  ↓
Files / Directories / Resources
```

The simplest way to remember it:

> **A user is an identity.**
>
> **A group is a collection of users.**
>
> **Permissions determine what the owner, group, and others can do.**

Example:

```text
luffy
   │
   └──── member of ────> developers
                              │
                              └──── permissions ────> /project
```
