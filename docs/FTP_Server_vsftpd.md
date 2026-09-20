# FTP and vsftpd — Complete Concepts and Practical Guide

## 1. What is FTP?

**FTP (File Transfer Protocol)** is a network protocol used to transfer files between machines.

The important idea is:

```text
FTP client                         FTP server
     |                                  |
     | ------ FTP requests -----------> |
     |                                  |
     | <----- FTP responses ----------- |
     |                                  |
```

The client connects to a remote FTP server and can perform operations such as:

* list files
* download files
* upload files
* change directories
* delete files, if permitted
* rename files, if permitted

FTP is specifically designed around **file transfer**, unlike SSH, which provides a remote shell.

---

# 2. FTP vs SSH

These two protocols can both involve remote machines, but they provide different interfaces.

## SSH

SSH gives you a remote shell:

```bash
ssh nami@192.168.1.6
```

You can potentially execute Linux commands:

```bash
ls
cd
cat
mkdir
rm
sudo ...
```

Conceptually:

```text
ThinkPad
   |
   | SSH
   v
Ubuntu VM
   |
   v
Linux shell
```

---

## FTP

FTP gives you an FTP interface:

```bash
ftp 192.168.1.6
```

You then use FTP commands:

```text
ftp> ls
ftp> get readme.txt
ftp> put file.txt
ftp> pwd
```

Conceptually:

```text
ThinkPad
   |
   | FTP
   v
Ubuntu VM
   |
   v
vsftpd
   |
   v
FTP interface
```

FTP does **not** give you a normal Linux shell.

---

# 3. The architecture of this exercise

The intended setup is:

```text
                    NETWORK
                       |
          +------------+------------+
          |                         |
          v                         v
      ThinkPad                  Ubuntu VM
      FTP client                FTP server
      bendoe                    server-host
                               192.168.1.6
                                    |
                                  vsftpd
                                    |
                                  nami
                                    |
                                 /backup
```

The ThinkPad is the **FTP client**.

The Ubuntu VM is the **FTP server**.

`vsftpd` is the software implementing the FTP server.

`nami` is a Linux account used to authenticate to the FTP server.

`/backup` contains the files exposed through FTP.

---

# 4. What is vsftpd?

`vsftpd` means:

> Very Secure FTP Daemon

It is FTP server software for Linux.

Install it with:

```bash
sudo apt install vsftpd
```

Check its status:

```bash
systemctl status vsftpd
```

A healthy service looks like:

```text
Active: active (running)
```

You can also check whether something is listening on FTP port 21:

```bash
sudo ss -lntp | grep :21
```

FTP normally uses:

```text
TCP 21
```

for the FTP control connection.

---

# 5. Important vsftpd configuration

The configuration file is:

```text
/etc/vsftpd.conf
```

A basic configuration for this exercise:

```ini
anonymous_enable=NO
local_enable=YES
chroot_local_user=YES
write_enable=NO

pasv_min_port=40000
pasv_max_port=50000
```

After changing the configuration:

```bash
sudo systemctl restart vsftpd
```

---

# 6. `anonymous_enable=NO`

```ini
anonymous_enable=NO
```

This disables anonymous FTP access.

Without anonymous authentication, clients need a real local Linux account.

For example:

```text
nami
```

---

# 7. `local_enable=YES`

```ini
local_enable=YES
```

This allows local Linux users to authenticate to vsftpd.

For example:

```text
nami
```

is a Linux user on the Ubuntu server.

vsftpd can authenticate that user using the normal Linux authentication system.

---

# 8. Creating the FTP user

The exercise creates:

```bash
sudo adduser nami --home /backup --no-create-home
```

This creates a Linux user called:

```text
nami
```

and gives it:

```text
/backup
```

as its home directory.

The important distinction is:

> `nami` is a Linux user account on the FTP server.

It is not the FTP server itself.

The server is:

```text
Ubuntu VM + vsftpd
```

The FTP account is:

```text
nami
```

---

# 9. `/backup` as the user's home

The user configuration makes:

```text
nami
```

associated with:

```text
/backup
```

Conceptually:

```text
nami
 |
 +---- home directory ----> /backup
```

This is why `/backup` becomes the directory associated with the FTP account.

---

# 10. `chroot_local_user=YES`

```ini
chroot_local_user=YES
```

This is an important security concept.

`chroot` changes the filesystem view available to a process.

For the FTP user, `/backup` can become the apparent root of the FTP environment.

The real server filesystem might be:

```text
/
├── bin
├── etc
├── home
├── var
├── backup
└── ...
```

But FTP can present the user with:

```text
/
├── readme.txt
├── data.csv
└── test.html
```

where that FTP `/` corresponds to:

```text
/backup
```

on the real server.

Conceptually:

```text
REAL SERVER

/
├── etc
├── home
├── var
├── backup
│   ├── readme.txt
│   ├── data.csv
│   └── test.html
└── ...


FTP VIEW FOR nami

/
├── readme.txt
├── data.csv
└── test.html
```

So FTP's `/` does not necessarily mean the server's actual `/`.

---

# 11. Why use chroot?

Without restrictions, an FTP user could potentially navigate around the server filesystem.

With:

```ini
chroot_local_user=YES
```

the FTP environment can be restricted to the user's designated area.

In this exercise:

```text
nami
  |
  v
/backup
```

The intended idea is:

> `nami` should only see the FTP-controlled filesystem area.

---

# 12. Setting `/backup` permissions

The exercise uses:

```bash
sudo chown root:root /backup
sudo chmod 755 /backup
```

Check it with:

```bash
ls -ld /backup
```

The permissions:

```text
drwxr-xr-x
```

mean:

```text
d       directory

rwx     owner
r-x     group
r-x     others
```

The owner can:

* read
* write
* enter the directory

Others can:

* read
* enter the directory

but cannot create or delete files.

This is relevant because the exercise sets:

```ini
write_enable=NO
```

---

# 13. `write_enable=NO`

```ini
write_enable=NO
```

This disables FTP write operations.

The user can read/download files but cannot use FTP to upload or modify files.

Conceptually:

```text
FTP client
    |
    | GET
    v
Server
    |
    v
Allowed
```

but:

```text
FTP client
    |
    | PUT
    v
Server
    |
    v
Rejected
```

This creates a **read-only FTP service**.

---

# 14. Creating test files

On the server, create files in:

```text
/backup
```

For example:

```bash
cd /backup

echo "This file is being used to test my FTP server." > readme.txt

cat > data.csv <<EOF
id,name,role
1,luffy,admin
2,zoro,user
3,nami,user
EOF

cat > test.html <<EOF
<!DOCTYPE html>
<html>
<head>
    <title>FTP Test</title>
</head>
<body>
    <h1>FTP Server Test</h1>
    <p>This file was transferred through FTP.</p>
</body>
</html>
EOF
```

Check:

```bash
ls -lh /backup
```

You should see files such as:

```text
data.csv
readme.txt
test.html
```

---

# 15. The most important distinction: local vs remote

This was the main source of confusion during the FTP test.

When you run:

```bash
ftp 192.168.1.6
```

there are **two machines** involved:

```text
LOCAL MACHINE                         REMOTE MACHINE

ThinkPad                              Ubuntu VM
bendoe                                server-host
                                      192.168.1.6
     |                                      |
     |             FTP connection            |
     +-------------------------------------> |
```

The FTP client runs on the **local machine**.

vsftpd runs on the **remote machine**.

---

# 16. `pwd` vs `lpwd`

Inside the FTP client:

```text
ftp> pwd
```

means:

> Show the current **remote** directory.

Example:

```text
Remote directory: /backup
```

Whereas:

```text
ftp> lpwd
```

means:

> Show the current **local** directory.

Example:

```text
Local directory: /home/bendoe/backup
```

These are completely different machines.

---

# 17. `cd` vs `lcd`

Similarly:

```text
ftp> cd /backup
```

changes the **remote** directory.

While:

```text
ftp> lcd /home/bendoe/backup
```

changes the **local** directory.

Remember:

```text
cd  = remote
lcd = local
```

---

# 18. `ls`

Inside FTP:

```text
ftp> ls
```

lists the contents of the **remote FTP server**.

For example:

```text
data.csv
readme.txt
test.html
```

It does NOT list the local ThinkPad directory.

---

# 19. `get`

The FTP command:

```text
ftp> get readme.txt
```

means:

> Download `readme.txt` from the remote FTP server to the local machine.

Direction:

```text
REMOTE
/backup/readme.txt
       |
       | GET
       v
LOCAL
/home/bendoe/backup/readme.txt
```

So:

```text
get = REMOTE → LOCAL
```

---

# 20. `put`

The opposite operation is:

```text
ftp> put somefile.txt
```

This means:

> Upload a local file to the remote FTP server.

Direction:

```text
LOCAL
/home/bendoe/somefile.txt
       |
       | PUT
       v
REMOTE
/backup/somefile.txt
```

So:

```text
put = LOCAL → REMOTE
```

However, in this exercise:

```ini
write_enable=NO
```

means uploads are disabled.

---

# 21. Why `cat` doesn't work inside FTP

We tried:

```text
ftp> cat data.csv
```

and received:

```text
?Invalid command.
```

That's because:

```bash
cat
```

is a **Linux shell command**, not an FTP command.

Inside FTP, use:

```text
ftp> get data.csv
```

Then leave FTP:

```text
ftp> bye
```

and inspect the downloaded file from the Linux shell:

```bash
cat data.csv
```

The distinction is:

```text
Linux shell:

$ cat file.txt


FTP client:

ftp> get file.txt
```

---

# 22. Why `get` initially failed

We encountered:

```text
ftp> get readme.txt

local: readme.txt remote: readme.txt
ftp: Can't access `readme.txt': Permission denied
```

The important thing was that FTP was trying to create the **local destination file**.

`get` does not just read the remote file and display it.

It tries to create a local copy.

The destination is determined by the FTP client's current **local directory**.

---

# 23. The two `/backup` directories

This was another major source of confusion.

There can be:

```text
SERVER:
/backup
```

and:

```text
THINKPAD:
/home/bendoe/backup
```

They are completely unrelated directories because they belong to different machines.

For example:

```text
SERVER VM

/backup
├── data.csv
├── readme.txt
└── test.html
```

while:

```text
THINKPAD

/home/bendoe/backup
└── readme.txt
```

could contain a downloaded copy.

The FTP transfer connects them:

```text
SERVER                              THINKPAD

/backup/readme.txt
       |
       | FTP GET
       v
                                   /home/bendoe/backup/readme.txt
```

---

# 24. Why creating `/backup` locally was confusing

On the ThinkPad we had:

```text
/home/bendoe/backup
```

owned by:

```text
bendoe:bendoe
```

with:

```text
drwxrwxr-x
```

That directory is writable by `bendoe`.

But this:

```text
/backup
```

is a completely different path.

Linux absolute paths start from `/`.

Therefore:

```text
/backup
```

is NOT:

```text
/home/bendoe/backup
```

They are different directories.

The correct local path for the ThinkPad was:

```text
/home/bendoe/backup
```

or simply:

```text
~/backup
```

---

# 25. How to test local permissions

Before blaming FTP, test whether the local user can create a file:

```bash
touch ~/backup/test.txt
```

If that works:

```bash
rm ~/backup/test.txt
```

then `bendoe` has permission to write there.

You can inspect ownership with:

```bash
ls -ld ~/backup
```

Example:

```text
drwxrwxr-x 2 bendoe bendoe ... backup
```

This means `bendoe` owns the directory and can write to it.

---

# 26. The critical mistake: connecting the server to itself

At one point the shell was:

```text
server@server-host:/backup$
```

and:

```bash
hostname -I
```

returned:

```text
192.168.1.6
```

Then FTP was run against:

```bash
ftp 192.168.1.6
```

That means:

```text
Ubuntu VM
192.168.1.6
   |
   | FTP
   v
Ubuntu VM
192.168.1.6
```

The FTP client and FTP server were on the **same machine**.

That is technically possible, but it was not the architecture intended for this exercise.

It also explains why the local FTP client could not see:

```text
/home/bendoe/backup
```

because `bendoe` is the user on the ThinkPad, not necessarily a user/directory on the VM.

---

# 27. Correct architecture

The correct test is:

```text
                    LAN
                     |
        +------------+------------+
        |                         |
        v                         v
   THINKPAD                    UBUNTU VM
   bendoe                     server-host
   FTP client                 192.168.1.6
        |                         |
        | FTP                     |
        +------------------------>|
                                  |
                                vsftpd
                                  |
                                nami
                                  |
                               /backup
                                  |
                     +------------+------------+
                     |            |            |
                  readme.txt   data.csv    test.html
```

---

# 28. Correct testing procedure

## On the server

Make sure vsftpd is running:

```bash
sudo systemctl status vsftpd
```

Make sure the files exist:

```bash
ls -l /backup
```

Expected:

```text
data.csv
readme.txt
test.html
```

---

## Leave the server

If you connected through SSH:

```bash
exit
```

You should return to the ThinkPad:

```text
bendoe@bendoe-ThinkPad-T590
```

Check:

```bash
hostname
```

You should see the ThinkPad hostname, not:

```text
server-host
```

---

## Start FTP from the ThinkPad

```bash
ftp 192.168.1.6
```

Authenticate:

```text
Name: nami
Password: ********
```

Expected:

```text
230 Login successful.
```

---

## Check the remote directory

```text
ftp> pwd
ftp> ls
```

You should see:

```text
data.csv
readme.txt
test.html
```

---

## Select the local download directory

```text
ftp> lcd /home/bendoe/backup
```

Verify:

```text
ftp> lpwd
```

Expected:

```text
Local directory: /home/bendoe/backup
```

---

## Download

```text
ftp> get readme.txt
```

Then:

```text
ftp> get data.csv
```

and:

```text
ftp> get test.html
```

Exit:

```text
ftp> bye
```

Then on the ThinkPad:

```bash
ls -l ~/backup
```

You should see:

```text
data.csv
readme.txt
test.html
```

Inspect:

```bash
cat ~/backup/readme.txt
cat ~/backup/data.csv
cat ~/backup/test.html
```

---

# 29. Useful FTP commands

| FTP command   | Meaning                              |
| ------------- | ------------------------------------ |
| `open IP`     | Connect to an FTP server             |
| `user nami`   | Authenticate as a user               |
| `pwd`         | Show remote directory                |
| `lpwd`        | Show local directory                 |
| `cd DIR`      | Change remote directory              |
| `lcd DIR`     | Change local directory               |
| `ls`          | List remote files                    |
| `get FILE`    | Download remote file                 |
| `put FILE`    | Upload local file                    |
| `mget FILES`  | Download multiple files              |
| `mput FILES`  | Upload multiple files                |
| `delete FILE` | Delete remote file if permitted      |
| `mkdir DIR`   | Create remote directory if permitted |
| `binary`      | Use binary transfer mode             |
| `ascii`       | Use ASCII transfer mode              |
| `bye`         | Exit FTP                             |
| `quit`        | Exit FTP                             |
| `help`        | Show FTP commands                    |

---

# 30. FTP transfer direction

Always remember:

```text
get:

REMOTE ---------------> LOCAL
       download


put:

LOCAL -----------------> REMOTE
       upload
```

This is one of the most important concepts.

---

# 31. Binary mode

The FTP client displayed:

```text
Using binary mode to transfer files.
```

Binary mode transfers the raw bytes of a file.

This is appropriate for files such as:

* images
* PDFs
* archives
* executables
* HTML
* CSV
* arbitrary binary data

For general file transfers, binary mode is usually the safe default.

---

# 32. FTP control connection vs data connection

FTP is unusual because it uses separate connections for control and data.

The **control connection** normally uses:

```text
TCP 21
```

It carries commands such as:

```text
USER
PASS
LIST
RETR
STOR
QUIT
```

The actual file data/listing uses a separate **data connection**.

Conceptually:

```text
FTP client                    FTP server

    |                             |
    |------ Control :21 --------->|
    |                             |
    |------ Data connection ----->|
    |                             |
```

---

# 33. Passive mode

The FTP client showed:

```text
229 Entering Extended Passive Mode (|||40804|)
```

This means the server selected a port for the FTP **data connection**.

The port was:

```text
40804
```

The configuration:

```ini
pasv_min_port=40000
pasv_max_port=50000
```

restricts passive FTP data ports to:

```text
40000-50000
```

This is useful for firewall configuration.

For example, if UFW is protecting the server, the FTP passive range may need to be allowed:

```bash
sudo ufw allow 40000:50000/tcp
```

The exact firewall configuration depends on the network and assignment.

---

# 34. Understanding the FTP listing

A listing such as:

```text
-rw-r--r--    1 0 0 51 Sep 20 14:13 data.csv
```

can be interpreted as:

```text
-rw-r--r--
│
└── file type + permissions
```

Then:

```text
1
```

is the link count.

```text
0
```

is the numeric owner UID.

```text
0
```

is the numeric group ID.

```text
51
```

is the file size in bytes.

Then comes the modification date/time and filename.

---

# 35. Why files showed UID/GID 0

The listing showed:

```text
1 0 0
```

UID `0` and GID `0` normally correspond to:

```text
root
root
```

So the files were owned by root.

For example:

```text
-rw-r--r-- 1 root root ... readme.txt
```

means:

```text
owner: root
group: root
```

and:

```text
rw-
```

means root can read and write.

```text
r--
```

means others can only read.

Therefore `nami` can read the file but cannot modify it.

---

# 36. Why `ls` worked but `get` failed

These are different operations.

When:

```text
ftp> ls
```

works, the FTP server successfully sends a directory listing.

When:

```text
ftp> get readme.txt
```

is executed, FTP must:

1. read the remote file
2. establish the FTP data transfer
3. create/open the local destination file
4. write the downloaded bytes locally

A failure in step 3 can produce:

```text
Permission denied
```

even though the remote file itself is readable.

Therefore:

> A successful `ls` does not automatically mean the local destination for `get` is writable.

---

# 37. Debugging `get`

If:

```text
ftp> get readme.txt
```

fails, check:

```text
ftp> lpwd
```

Then exit FTP and check that local directory:

```bash
ls -ld /path/from/lpwd
```

Test whether your local user can write:

```bash
touch /path/from/lpwd/test.txt
```

If it fails:

```text
Permission denied
```

the problem is local filesystem permissions.

If the directory doesn't exist:

```text
No such file or directory
```

the local path is wrong.

---

# 38. Checking which machine you are on

This became important during debugging.

Run:

```bash
whoami
```

to see the current user.

Run:

```bash
hostname
```

to see the machine.

Run:

```bash
hostname -I
```

to see IP addresses.

For example, on the server:

```text
whoami
server

hostname
server-host

hostname -I
192.168.1.6
```

That tells you:

```text
You are on the VM.
```

On the ThinkPad you should instead see something like:

```text
whoami
bendoe

hostname
bendoe-ThinkPad-T590
```

---

# 39. Checking the local environment from inside FTP

Inside the FTP client:

```text
ftp> !pwd
```

runs the local shell's `pwd`.

Similarly:

```text
ftp> !whoami
ftp> !hostname
```

run those commands locally.

This is useful when you are unsure which machine the FTP client is running on.

Remember:

```text
ftp> pwd
```

= remote

while:

```text
ftp> !pwd
```

= local shell

and:

```text
ftp> lpwd
```

= local FTP directory.

---

# 40. The `530 Login incorrect` error

We also encountered:

```text
530 Login incorrect.
```

when trying:

```text
Name: bendoe
```

The FTP configuration was built around the user:

```text
nami
```

Therefore the intended test is:

```text
Name: nami
Password: ...
```

If you want `bendoe` to authenticate through FTP, `bendoe` must also exist as an appropriate local account on the **FTP server**, and the account must satisfy the server's authentication/configuration requirements.

Your ThinkPad username does not automatically become an FTP user on the VM.

---

# 41. Local users exist per machine

This is another important Linux concept.

You can have:

```text
THINKPAD

bendoe
```

and:

```text
VM

server
nami
zoro
luffy
```

These are separate Linux systems.

The existence of:

```text
bendoe
```

on the ThinkPad does not mean:

```text
bendoe
```

exists on the VM.

Similarly:

```text
/home/bendoe
```

on the ThinkPad does not imply that:

```text
/home/bendoe
```

exists on the VM.

---

# 42. Why FTP is useful

Suppose a server has:

```text
/backup
├── report.pdf
├── database.csv
├── image.png
└── archive.tar.gz
```

A remote user might only need to retrieve these files.

Giving them SSH access would provide a much broader interface:

```text
SSH
 |
 +-- shell
 +-- commands
 +-- filesystem
 +-- processes
 +-- potentially sudo
```

FTP can instead provide a restricted file-transfer interface:

```text
FTP
 |
 +-- list files
 +-- download files
 +-- possibly upload files
 +-- restricted directory
```

Therefore FTP can separate:

> **file transfer access**

from:

> **general shell access**

---

# 43. Security considerations

FTP itself is an old protocol and traditional FTP does not encrypt credentials or file contents.

For real production environments, consider secure alternatives such as:

* SFTP over SSH
* FTPS

For this assignment, however, `vsftpd` is useful because it teaches:

* network services
* Linux users
* authentication
* permissions
* chroot
* ports
* passive FTP
* firewall configuration
* client/server architecture
* file transfer

---

# 44. Final mental model

Keep these roles separate:

```text
┌──────────────────────────────────────────────┐
│                 THINKPAD                     │
│                                              │
│  bendoe                                      │
│  FTP client                                  │
│                                              │
│  ~/backup/                                   │
│      ↑                                       │
│      │ get                                   │
└──────┼───────────────────────────────────────┘
       │
       │ FTP
       │
       │ 192.168.1.6
       ↓
┌──────────────────────────────────────────────┐
│                UBUNTU VM                     │
│                                              │
│  server-host                                 │
│  192.168.1.6                                 │
│                                              │
│  vsftpd                                      │
│      │                                       │
│      │ authenticates                         │
│      ↓                                       │
│     nami                                     │
│      │                                       │
│      │ restricted FTP view                   │
│      ↓                                       │
│   /backup/                                   │
│      ├── readme.txt                          │
│      ├── data.csv                            │
│      └── test.html                           │
│                                              │
└──────────────────────────────────────────────┘
```

The complete `get` operation is:

```text
ThinkPad                         Ubuntu VM
bendoe                           192.168.1.6

/home/bendoe/backup              /backup
        |                             |
        |                             |
        |        FTP GET              |
        | <---------------------------|
        |                             |
        |                       readme.txt
        |                             |
        v                             |
readme.txt                           |
```

More precisely:

```text
REMOTE SERVER
/backup/readme.txt
       |
       | get
       v
LOCAL MACHINE
/home/bendoe/backup/readme.txt
```

---

# 45. The five commands to remember

For basic FTP file transfer, remember these first:

```text
ls
```

> What files are on the remote server?

```text
pwd
```

> Where am I on the remote server?

```text
lpwd
```

> Where will downloaded files go locally?

```text
get file
```

> Download remote → local.

```text
put file
```

> Upload local → remote.

And the two directory commands:

```text
cd DIR
```

> Change remote directory.

```text
lcd DIR
```

> Change local directory.

The central concept is:

```text
                 FTP CLIENT
              ThinkPad/bendoe
                    |
                    | FTP
                    v
                 vsftpd
              Ubuntu/server
                    |
                  nami
                    |
                 /backup
```

**The FTP client and FTP server should be treated as two different roles, even though Linux allows you to run both on the same machine.**
