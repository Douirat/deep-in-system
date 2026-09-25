# Linux `chmod` — Full Explanation

`chmod` is one of the most important Linux commands for understanding **file permissions**.

The name means:

> **change mode**

It changes the **permissions** associated with a file or directory.

The basic syntax is:

```bash
chmod [options] MODE FILE
```

For example:

```bash
chmod 755 script.sh
```

This changes the permissions of `script.sh`.

---

# 1. Why do we need `chmod`?

Linux is a multi-user operating system.

Imagine this file:

```text
/home/luffy/project.txt
```

There may be:

* the owner of the file
* other users in the same group
* completely unrelated users

Linux needs to answer:

> Who is allowed to read this file?

> Who is allowed to modify it?

> Who is allowed to execute it?

That's what permissions control.

---

# 2. The three basic permissions

Linux has three fundamental permissions:

| Permission    | Symbol | Meaning                  |
| ------------- | ------ | ------------------------ |
| Read          | `r`    | Read the contents        |
| Write         | `w`    | Modify the contents      |
| Execute       | `x`    | Execute/use as a program |
| No permission | `-`    | Nothing allowed          |

For example:

```text
rwx
```

means:

```text
read
write
execute
```

while:

```text
r--
```

means:

```text
read only
```

---

# 3. The three categories of users

Permissions are assigned to three categories:

```text
user
group
others
```

They are often abbreviated:

```text
u
g
o
```

## User — `u`

The **owner** of the file.

## Group — `g`

The group associated with the file.

## Others — `o`

Everyone else.

There is also:

```text
a
```

which means:

> all = user + group + others

---

# 4. Reading `ls -l`

Let's look at:

```bash
ls -l
```

You might see:

```text
-rwxr-xr-- 1 luffy developers 1200 Sep 25 script.sh
```

The important part is:

```text
-rwxr-xr--
```

Break it apart:

```text
- rwx r-x r--
  --- --- ---
   u   g   o
```

So:

```text
-     rwx     r-x     r--
│      │        │       │
│      │        │       └── others
│      │        └────────── group
│      └─────────────────── owner
└────────────────────────── file type
```

Therefore:

### Owner

```text
rwx
```

Can:

* read
* write
* execute

### Group

```text
r-x
```

Can:

* read
* execute
* cannot write

### Others

```text
r--
```

Can:

* read
* cannot write
* cannot execute

---

# 5. The first character is not a permission

This is important.

Consider:

```text
-rwxr-xr--
```

The first character:

```text
-
```

describes the **file type**.

Common values:

```text
-    regular file
d    directory
l    symbolic link
```

For example:

```text
drwxr-xr-x
```

is a directory.

Whereas:

```text
-rwxr-xr-x
```

is a regular file.

---

# 6. `chmod` symbolic mode

One way to use `chmod` is with letters.

General syntax:

```bash
chmod [who][operator][permissions] file
```

For example:

```bash
chmod u+x script.sh
```

Means:

```text
u    → user/owner
+    → add
x    → execute
```

So:

> Give the owner execute permission.

---

# 7. The operators

There are three important operators:

```text
+
-
=
```

## `+` — add permission

```bash
chmod u+x script.sh
```

Add execute permission to the owner.

---

## `-` — remove permission

```bash
chmod u-x script.sh
```

Remove execute permission from the owner.

---

## `=` — exactly set permissions

```bash
chmod u=r script.sh
```

The owner's permissions become exactly:

```text
r--
```

Existing `w` and `x` permissions are removed.

---

# 8. Examples of symbolic `chmod`

### Add execute to owner

```bash
chmod u+x script.sh
```

### Remove write from owner

```bash
chmod u-w file.txt
```

### Add read to group

```bash
chmod g+r file.txt
```

### Remove write from group

```bash
chmod g-w file.txt
```

### Add execute to others

```bash
chmod o+x script.sh
```

### Remove all permissions from others

```bash
chmod o-rwx file.txt
```

---

# 9. Using `a`

`a` means everyone.

For example:

```bash
chmod a+x script.sh
```

means:

```text
owner + execute
group + execute
others + execute
```

You can also write:

```bash
chmod +x script.sh
```

In this context, it means adding execute permission for all applicable classes.

---

# 10. Combining permissions

You can combine permissions:

```bash
chmod u+rwx file
```

Owner gets:

```text
rwx
```

Or:

```bash
chmod ug+rw file
```

means:

```text
user    → read + write
group   → read + write
others  → unchanged
```

Another example:

```bash
chmod u=rw,g=r,o= file.txt
```

Result:

```text
-rw-r-----
```

The owner:

```text
rw-
```

Group:

```text
r--
```

Others:

```text
---
```

---

# 11. Numeric permissions

This is the form you'll see very often:

```bash
chmod 755 script.sh
```

Instead of writing:

```bash
chmod u=rwx,g=rx,o=rx script.sh
```

we use numbers.

The values are:

| Permission | Value |
| ---------- | ----: |
| `r`        |   `4` |
| `w`        |   `2` |
| `x`        |   `1` |
| `-`        |   `0` |

You add the values together.

---

# 12. How `7`, `5`, etc. work

For example:

```text
rwx
```

means:

```text
r = 4
w = 2
x = 1

4 + 2 + 1 = 7
```

Therefore:

```text
rwx = 7
```

### `rw-`

```text
r = 4
w = 2
x = 0

4 + 2 = 6
```

Therefore:

```text
rw- = 6
```

### `r-x`

```text
r = 4
w = 0
x = 1

4 + 1 = 5
```

Therefore:

```text
r-x = 5
```

### `r--`

```text
4 + 0 + 0 = 4
```

Therefore:

```text
r-- = 4
```

### `---`

```text
0 + 0 + 0 = 0
```

Therefore:

```text
--- = 0
```

---

# 13. The famous `755`

When you write:

```bash
chmod 755 script.sh
```

you can split it:

```text
755
│││
││└── others
│└─── group
└──── owner
```

Convert each number:

```text
7 = rwx
5 = r-x
5 = r-x
```

So:

```text
755
```

means:

```text
owner   → rwx
group   → r-x
others  → r-x
```

Result:

```text
-rwxr-xr-x
```

---

# 14. Common permission combinations

These are worth memorizing:

| Number | Permissions |
| -----: | ----------- |
|    `0` | `---`       |
|    `1` | `--x`       |
|    `2` | `-w-`       |
|    `3` | `-wx`       |
|    `4` | `r--`       |
|    `5` | `r-x`       |
|    `6` | `rw-`       |
|    `7` | `rwx`       |

Then combine three numbers:

```text
644
```

means:

```text
owner   rw-
group   r--
others  r--
```

Result:

```text
-rw-r--r--
```

This is very common for regular files.

---

# 15. `644`

```bash
chmod 644 file.txt
```

means:

```text
owner   → read + write
group   → read
others  → read
```

So:

```text
-rw-r--r--
```

This is commonly appropriate for normal text/configuration files that don't need to be executable.

---

# 16. `600`

```bash
chmod 600 secret.txt
```

means:

```text
owner   → read + write
group   → nothing
others  → nothing
```

Result:

```text
-rw-------
```

Useful for files that should only be accessible by their owner.

For example, SSH private keys commonly have restrictive permissions.

---

# 17. `700`

```bash
chmod 700 private-directory
```

means:

```text
owner   → rwx
group   → ---
others  → ---
```

Result:

```text
drwx------
```

This means only the owner can access the directory.

---

# 18. `755`

```bash
chmod 755 script.sh
```

means:

```text
owner   → rwx
group   → r-x
others  → r-x
```

This is commonly used for executable scripts and programs.

---

# 19. `777`

```bash
chmod 777 file
```

means:

```text
owner   → rwx
group   → rwx
others  → rwx
```

Everyone can:

* read
* modify
* execute

This is generally **far more permission than necessary**.

Avoid using:

```bash
chmod 777
```

as a generic solution to permission problems.

If you get:

```text
Permission denied
```

don't immediately do:

```bash
chmod 777 file
```

Instead determine **which user needs which permission and why**.

---

# 20. `chmod` on directories

This is where beginners often get confused.

For a **file**:

```text
r → read contents
w → modify contents
x → execute program
```

For a **directory**, the meanings are different.

## Directory `r`

Allows you to list directory entries.

Example:

```bash
ls directory/
```

## Directory `w`

Allows modification of directory entries, such as:

* create files
* delete files
* rename files

But write alone is not normally enough; directory access generally also requires `x`.

## Directory `x`

Means you can **traverse/access** the directory.

For example:

```bash
cd directory/
```

requires execute permission on the directory.

This distinction is extremely important.

---

# 21. Directory example

Suppose:

```text
/home/project
```

has:

```text
drwxr-x---
```

That means:

```text
owner   → rwx
group   → r-x
others  → ---
```

Owner can:

* list
* enter
* create/delete files
* modify directory entries

Group can:

* list
* enter

Group cannot modify directory entries.

Others cannot access/traverse it.

---

# 22. Why `x` on directories is special

Suppose:

```text
directory/
└── secret.txt
```

The file might have:

```text
-rw-r--r--
```

but if you don't have `x` permission on the directory, you may still be unable to access:

```text
directory/secret.txt
```

because you cannot traverse the directory.

Think of directory `x` as:

> permission to pass through the directory.

---

# 23. `chmod` and `chown` are different

This is extremely important.

`chmod` changes:

> **permissions**

`chown` changes:

> **ownership**

Example:

```bash
chown luffy file.txt
```

changes the owner.

While:

```bash
chmod 600 file.txt
```

changes permissions.

You can also change owner and group:

```bash
sudo chown luffy:developers file.txt
```

Now:

```text
owner = luffy
group = developers
```

But `chmod` still determines what that owner/group/others are allowed to do.

---

# 24. Example: understanding a complete file

Suppose:

```bash
ls -l script.sh
```

returns:

```text
-rwxr-x--- 1 luffy developers 1200 Sep 25 script.sh
```

Let's analyze it.

### File type

```text
-
```

Regular file.

### Owner

```text
luffy
```

### Group

```text
developers
```

### Permissions

```text
rwx r-x ---
```

Therefore:

```text
luffy       → read/write/execute
developers  → read/execute
everyone    → nothing
```

Equivalent numeric representation:

```text
750
```

So:

```bash
chmod 750 script.sh
```

would produce those permissions.

---

# 25. Changing one permission

Suppose:

```text
-rw-r--r--
```

You want the owner to execute it.

Use:

```bash
chmod u+x file
```

Result:

```text
-rwxr--r--
```

Notice that only the owner's permissions changed.

---

# 26. Removing one permission

Suppose:

```text
-rwxr-xr-x
```

You don't want others to execute it.

Use:

```bash
chmod o-x file
```

Result:

```text
-rwxr-xr--
```

---

# 27. Setting exact permissions

Suppose you want:

```text
owner   rw
group   r
others  nothing
```

You could use:

```bash
chmod u=rw,g=r,o= file
```

or:

```bash
chmod 640 file
```

The numeric version is shorter.

---

# 28. Recursive `chmod`

You can use:

```bash
chmod -R 755 directory/
```

`-R` means:

> recursive

It applies the change to the directory and everything underneath it.

For example:

```text
project/
├── file1
├── file2
└── src/
    ├── a
    └── b
```

Running:

```bash
chmod -R 755 project/
```

affects all of these.

## Be careful

Recursive permission changes can cause serious problems.

For example:

```bash
chmod -R 777 /
```

would be an extremely dangerous command.

Don't use recursive `chmod` unless you understand exactly what objects you're modifying.

---

# 29. A better recursive approach

Files and directories often need **different permissions**.

For example, you may want:

```text
directories → 755
files       → 644
```

Rather than blindly doing:

```bash
chmod -R 755 project/
```

you can use:

```bash
find project/ -type d -exec chmod 755 {} \;
find project/ -type f -exec chmod 644 {} \;
```

This gives:

```text
directories → rwxr-xr-x
files       → rw-r--r--
```

For a web project, however, the correct values depend on the application and service account, so don't apply this blindly.

---

# 30. `chmod` and `sudo`

You can only change permissions when you have sufficient authority.

For example:

```bash
chmod 600 myfile
```

works if you own `myfile`.

But if:

```text
-rw-r--r-- root root file.txt
```

and you're an ordinary user, you generally cannot change its permissions.

You may need:

```bash
sudo chmod 600 file.txt
```

because `root` owns the file.

---

# 31. `chmod` does not change ownership

For example:

```bash
sudo chmod 600 file.txt
```

does **not** change:

```text
owner
group
```

It only changes:

```text
permissions
```

If you need to change ownership:

```bash
sudo chown luffy file.txt
```

---

# 32. Special permissions

There are three additional permission mechanisms worth learning:

```text
setuid
setgid
sticky bit
```

They use the execute position in a special way.

---

# 33. Setuid

Numeric value:

```text
4000
```

Example:

```bash
chmod 4755 program
```

You may see:

```text
-rwsr-xr-x
```

The `s` appears where the owner's `x` normally appears.

For an executable, setuid means it runs with the **file owner's effective privileges**.

This is security-sensitive and should not be added casually.

---

# 34. Setgid

Numeric value:

```text
2000
```

Example:

```bash
chmod 2755 directory
```

On a directory, setgid causes newly created files/subdirectories to inherit the directory's group in typical Linux behavior.

This is useful for shared project directories.

For example:

```text
project/
    owner: luffy
    group: developers
```

With setgid, new files created inside can inherit:

```text
developers
```

as their group.

---

# 35. Sticky bit

Numeric value:

```text
1000
```

Common example:

```bash
ls -ld /tmp
```

You will often see something like:

```text
drwxrwxrwt
```

The final:

```text
t
```

is the sticky bit.

For a shared writable directory, it prevents ordinary users from deleting or renaming files belonging to other users, subject to the normal ownership/root rules.

This is why `/tmp` can be writable by many users without allowing everyone to delete everyone else's files.

---

# 36. Four-digit chmod

You may therefore see:

```bash
chmod 4755 file
```

or:

```bash
chmod 2755 directory
```

or:

```bash
chmod 1777 directory
```

The first digit represents special permissions:

```text
4 → setuid
2 → setgid
1 → sticky
```

The remaining three digits are:

```text
owner
group
others
```

So:

```text
4755
│
└── special: setuid

755
├── owner = rwx
├── group = r-x
└── others = r-x
```

---

# 37. Checking permissions

The most common command:

```bash
ls -l
```

For a specific file:

```bash
ls -l file.txt
```

For a directory itself rather than its contents:

```bash
ls -ld directory/
```

You can also use:

```bash
stat file.txt
```

which gives more detailed information.

For example:

```bash
stat file.txt
```

can show:

```text
Access: (0644/-rw-r--r--)
Uid: (1000/luffy)
Gid: (1000/developers)
```

This is useful when debugging permissions.

---

# 38. `chmod` versus `umask`

Another important concept is `umask`.

`chmod` changes permissions **after a file exists**.

`umask` influences the permissions used when new files/directories are created.

For example:

```bash
umask
```

might return:

```text
0022
```

This helps explain why a newly created regular file might normally start around:

```text
644
```

and a newly created directory around:

```text
755
```

The exact resulting permissions depend on the program creating the object and the permission bits it requests.

Think of it as:

```text
umask → controls default permission restrictions
chmod  → changes permissions explicitly
```

---

# 39. A useful mental model

When you see:

```text
-rwxr-x---
```

don't try to memorize it as one giant string.

Always split it:

```text
- | rwx | r-x | ---
    u     g     o
```

Then translate:

```text
rwx → 7
r-x → 5
--- → 0
```

Therefore:

```text
750
```

---

# 40. The permission table you should memorize

This table is the most useful one:

| Symbol | Numeric | Meaning                |
| ------ | ------: | ---------------------- |
| `---`  |     `0` | nothing                |
| `--x`  |     `1` | execute                |
| `-w-`  |     `2` | write                  |
| `-wx`  |     `3` | write + execute        |
| `r--`  |     `4` | read                   |
| `r-x`  |     `5` | read + execute         |
| `rw-`  |     `6` | read + write           |
| `rwx`  |     `7` | read + write + execute |

Then:

```text
XYZ
│││
││└── others
│└─── group
└──── owner
```

---

# 41. Common permissions to recognize

You should be able to immediately understand these:

## `644`

```text
-rw-r--r--
```

```text
owner   → rw-
group   → r--
others  → r--
```

## `600`

```text
-rw-------
```

```text
owner   → rw-
group   → ---
others  → ---
```

## `755`

```text
-rwxr-xr-x
```

```text
owner   → rwx
group   → r-x
others  → r-x
```

## `700`

```text
-rwx------
```

```text
owner   → rwx
group   → ---
others  → ---
```

## `750`

```text
-rwxr-x---
```

```text
owner   → rwx
group   → r-x
others  → ---
```

## `777`

```text
-rwxrwxrwx
```

Everyone has everything.

Use with caution.

---

# 42. SSH example

You may have:

```bash
sudo mkdir -p /home/luffy/.ssh
```

Then:

```bash
sudo chmod 700 /home/luffy/.ssh
```

This makes the directory:

```text
drwx------
```

So the owner can:

```text
read
write
traverse
```

while other users cannot access it.

Then:

```bash
sudo chmod 600 /home/luffy/.ssh/authorized_keys
```

would make the file:

```text
-rw-------
```

Meaning:

```text
luffy → read/write
everyone else → nothing
```

This is a common restrictive permission setup for SSH key material.

---

# 43. `chmod` does not bypass the entire permission system

Suppose you have:

```text
/home
└── luffy
    └── secret.txt
```

Even if:

```text
secret.txt
```

has:

```text
644
```

a user still needs appropriate permissions on the **parent directories** to reach the file.

For example:

```text
/home
/home/luffy
/home/luffy/secret.txt
```

Linux checks the path components as well.

This is one reason permission debugging can be more complicated than simply looking at the final file.

---

# 44. `chmod` is not the whole Linux security model

Linux access control can involve more than traditional Unix permissions.

You may eventually encounter:

```text
chmod
chown
chgrp
umask
ACLs
setuid
setgid
sticky bit
SELinux
AppArmor
capabilities
```

For Linux administration, a useful progression is:

```text
users
   ↓
groups
   ↓
owner/group/others
   ↓
rwx
   ↓
chmod
   ↓
chown/chgrp
   ↓
umask
   ↓
special permissions
   ↓
ACL
```

That gives you a solid foundation before moving into more advanced access-control systems.

---

# 45. Commands worth practicing

Create a practice directory:

```bash
mkdir ~/chmod-practice
cd ~/chmod-practice
```

Create a file:

```bash
touch file.txt
```

Check it:

```bash
ls -l file.txt
```

Set:

```bash
chmod 644 file.txt
```

Check:

```bash
ls -l file.txt
```

Then:

```bash
chmod 600 file.txt
ls -l file.txt
```

Then:

```bash
chmod 640 file.txt
ls -l file.txt
```

Then:

```bash
chmod 755 file.txt
ls -l file.txt
```

Now practice symbolic mode:

```bash
chmod u-x file.txt
chmod g+w file.txt
chmod o-r file.txt
```

And inspect after each command:

```bash
ls -l file.txt
```

The important part is to **predict the permission string before running the command**, then check whether you were correct.

---

# 46. The core idea

If you remember only one diagram, remember this:

```text
                 FILE
                  │
        ┌─────────┼─────────┐
        │         │         │
      OWNER      GROUP     OTHERS
        │         │         │
       rwx       rwx       rwx
```

Numerically:

```text
r = 4
w = 2
x = 1
```

So:

```text
rwx = 4 + 2 + 1 = 7
rw- = 4 + 2     = 6
r-x = 4 + 1     = 5
r-- = 4         = 4
-w- = 2
--x = 1
--- = 0
```

Then:

```text
chmod 750 file
      │││
      ││└── others
      │└─── group
      └──── owner
```

Therefore:

```text
750
 │
 ├── 7 → rwx → owner
 ├── 5 → r-x → group
 └── 0 → --- → others
```

That is the foundation of `chmod`.

---

# 47. Recommended learning order

After mastering `chmod`, continue with:

```text
1. Users
2. Groups
3. File ownership
4. chmod
5. chown
6. chgrp
7. umask
8. File permissions vs directory permissions
9. setuid
10. setgid
11. sticky bit
12. ACLs
13. SELinux/AppArmor
14. Linux capabilities
```

The most important relationship to understand is:

```text
USER
  │
  ├── owns ────────────────┐
  │                        │
GROUP                      FILE
  │                        │
  └── associated with ─────┤
                           │
                    PERMISSIONS
                           │
                  ┌────────┼────────┐
                  │        │        │
                 USER     GROUP    OTHERS
                  │        │        │
                 rwx      rwx      rwx
```

Once this model is clear, commands such as `chmod`, `chown`, and `chgrp` become much easier to understand.
