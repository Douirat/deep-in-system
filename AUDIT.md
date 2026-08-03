#### General

##### Check Repository Content

Files that must be present in the repository:

- deep-in-system.sha1
- README.md

###### Are the required files present?

##### Verify that the virtual machine being audited matches the submitted one:

```console
user:~$ sha1sum deep-in-system.ova > deep-in-system-toaudit.sha1
user:~$ diff deep-in-system.sha1  deep-in-system-toaudit.sha1 ; echo $?
0
user:~$
```

###### Is the SHA1 checksum of the audited VM identical to the submitted one?

##### Check the Virtual machine aliases

###### The virtual machine is clean of any alias that may affect the results of the audit commands?

#### The Virtual Machine Part:

##### Check the Linux distribution

To get information about the OS release:

```console
user:~$ cat /etc/os-release
PRETTY_NAME="Ubuntu <...> LTS"
NAME="Ubuntu"
VERSION_ID="<...>"
VERSION="<...> LTS <...>"
VERSION_CODENAME=<...>
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=<...>
user:~$
```

##### Check if ubuntu is a server and not a desktop:

```console
user:~$ dpkg -l ubuntu-desktop
dpkg-query: no packages found matching ubuntu-desktop
user:~$
```

You can check the versions of the ubuntu server from here: [Get Ubuntu Server](https://ubuntu.com/download/server)

###### Is the installed operating system Ubuntu Server (latest LTS)?

###### Is the system a server installation and not a desktop version?

##### Check the VM disk and partitions

Check the VM disk and partitions with this command:

```console
user:~$ lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT /dev/sda
NAME   FSTYPE SIZE MOUNTPOINT
sda            30G
├─sda<...>          1M
├─sda<...> swap     4G [SWAP]
├─sda<...> ext4    15G /
├─sda<...> ext4     5G /home
└─sda<...> ext4     6G /backup
user:~$
```

- The VM disk size must be 30GB.

- VM disk must be divided into these partitions:
  "swap:" 4G
  "/": 15G
  "/home": 5G
  "/backup": 6G

###### Is the VM Disk size correct?

> Minor size differences are acceptable (maximum error: ±0.5G).

###### Are the VM disk partitions correctly configured?

##### Check Hostname and User name

To check the hostname:

```console
user:~$ hostname
<username>-host
user:~$
```

To check the user name and groups:

```console
user:~$ id
uid=<...>({username}) gid=<...>({username}) groups=<...>({username}),<...>(sudo),<...>
user:~$
```

###### Is the hostname in the format "{username}-host"?

###### Does the learner use a non-root user?

###### Does the username contain the learner's login?

###### Is the user part of the `sudo` group?

###### Can the learner explain what the `sudo group` is and its role in Linux?

#### The Network & Security Part:

##### Check the VM IP address

The learner must show the file that was modified to set a static IP address.

###### Can the learner show and explain the configuration file used to set a static IP address?

###### Can the learner explain what a `netmask` is?

##### Check if the IP address is static with this command:

```console
user:~$ ip a | grep dynamic
user:~$
```

###### Is there no network interface using dynamic IP assignment?

##### Check if the internet works fine with the static IP address:

```console
user:~$ ping -c 5 google.com
```

###### Can the VM connect to the internet properly?

###### Can the learner explain why a static IP address is important for a web server?

##### Check the SSH configuration

The learner must show the file that was modified to secure the SSH server.

###### Can the learner show and explain the SSH configuration file?

###### Is root login disabled (`PermitRootLogin no`)?

###### Is the SSH port set to `2222`?

##### Try to connect from outside the VM

```console
outsideTheVM:~$ ssh {username}@{machine-ip} -p 2222
{username}@{machine-ip}'s password:
Welcome to Ubuntu <......>
InsideTheVM:~$ hostname
{username}-host
InsideTheVM:~$
```

###### Can the learner connect via SSH successfully?

###### Can the learner explain what an `SSH server` is and its role?

##### Check the firewall

If `ufw` is used:

```console
user:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW       Anywhere
<...>
Apache                     ALLOW       Anywhere
<...>
user:~$
```

Otherwise, the learner must demonstrate the firewall in use.

###### Is the firewall enabled?

##### Ask the learner to justify why each open port is open

###### Can the learner justify every open port?

###### Can the learner explain what a `firewall` is and its role on a server?

#### User Management Part:

##### Check `luffy` user

The learner should connect to the machine with the "luffy" user by using his SSH key:

- Private key: The learner's generated SSH key

###### Can the learner log in as `luffy` using a private key and no password?

##### Check the groups of `luffy` user:

```console
luffy:~$ groups luffy
luffy : luffy sudo
luffy:~$
```

##### Check the home directory of `luffy` user:

```console
luffy:~$ echo ~
/home/luffy
luffy:~$ echo $HOME
/home/luffy
luffy:~$
```

###### Is `luffy` part of the `sudo` group?

###### Is the home directory `/home/luffy`?

##### Check `zoro` user

The learner should connect to the machine with the "zoro" user by using a password.

###### Can the learner log in as `zoro` using password authentication?

##### Try to execute a command with sudo:

```console
zoro:$ sudo cat /etc/shadow
zoro is not in the sudoers file.  This incident will be reported.
zoro:~$
```

##### Check the groups of `zoro` user:

```console
zoro:~$ groups zoro
zoro : zoro
zoro:~$
```

##### Check the home directory of `zoro` user:

```console
zoro:~$ echo ~
/home/zoro
zoro:~$ echo $HOME
/home/zoro
zoro:~$
```

###### Is `zoro` unable to execute sudo commands?

###### Is `zoro` not part of the sudo group?

###### Is the home directory `/home/zoro`?

#### Live User Creation

Within **10 minutes**, the learner must:

- Create a user named `kratos`
- Generate an SSH key during the exam
- Configure key-based authentication
- Add the user to the `sudo` group

The learner must then demonstrate:

- Successful SSH login with the private key
- Ability to execute a sudo command

> If the learner fails this exam, the project is failed.

###### Can the learner generate an SSH key?

###### Can the learner create a user?

###### Can the learner assign the public key correctly?

###### Can the learner add the user to the sudo group?

###### Can `kratos` connect via SSH using the private key?

###### Can `kratos` execute sudo commands?

#### Services Part:

##### Check `nami` user:

##### By using SSH create a file inside `/backup`:

```console
$ sudo touch /backup/audit-check
```

##### Try to connect as `nami` via FTP:

```console
user:~$ ftp {vm-ip}
Connected to {vm-ip}.
<...>
Name ({vm-ip}:{username}): nami
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
<...>
<...> audit-check
<...>
226 Directory send OK.
ftp> get audit-check
<...>
226 Transfer complete.
ftp>
```

###### Can the learner connect via FTP using the `nami` user?

###### Does the FTP user `nami` fail when attempting to upload a file (operation denied)?

###### Is the FTP user `nami` restricted to read-only access on the allowed directories?

###### Does the `audit-check` file appear in the FTP directory?

###### Can the file be downloaded?

##### Check `anonymous` user:

##### Try to connect with an `anonymous` user and a blank password:

```console
user:~$ ftp {vm-ip}
Connected to {vm-ip}.
<...>
Name ({vm-ip}:{username}): anonymous
331 Please specify the password.
Password:
530 Login incorrect.
ftp: Login failed
ftp>
```

###### Is `anonymous` FTP access denied?

###### Can the learner explain what an `FTP server` is and its role?

#### The Database Part:

###### Is the MySQL port closed to external access?

###### Is remote MySQL root login disabled (Host for root in mysql.user is limited to localhost)?

###### Is a dedicated, non-root MySQL user configured for WordPress?

###### Does wp-config.php use a non-root database user in the DB_USER setting?

#### WordPress Part:

##### From a browser, open:

```
http://{vm-ip}/
```

(HTTPS is acceptable if SSL is configured.)

###### Is WordPress installed and functional?

##### Ask the learner to log in with the admin user and post something.

###### Can the learner log in as an admin and create content?

##### Try to access to `http://{vm-ip}/wp-config.php`

###### Is the configuration file inaccessible from the browser?

#### Backup Part:

##### Check the cronjob:

###### Can the learner show the configured cron job?

###### Is there a cron job scheduled to run daily at 00:00 (`0 0 * * *`)?

###### Does the cron job create a WordPress database backup in `/backup`?

##### Check the FTP system functionality:

##### Before testing:

- Remove all backup files from `/backup`
- Remove `/var/log/backup.log`

##### Change cron schedule to:

```
* * * * *
```

##### After 1 minute, check the FTP Server with the `nami` user:

```console
user:~$ ftp {vm-ip}
Connected to {vm-ip}.
<...>
Name ({vm-ip}:{username}): nami
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
<...>
<...> {wordpress-backupfile}
<...>
226 Directory send OK.
ftp> get audit-check
<...>
226 Transfer complete.
ftp>
```

###### Does a backup file with today’s date appear via FTP?

##### Check the backup logs file:

```console
user:~$ cat /var/log/backup.log
<...>wordpress backup created!, date: <...>
user:~$
```

###### Does the log file exist and include a success message with timestamp?

###### Can the learner explain what a `cron job` is and its role?

###### Can the learner explain why `backups` are important?

#### Check the documentation:

###### Is the README.md file containing the clarification of all the knowledge learned and the steps passed by the learner to set up the server?

###### Does `README.md` clearly document:

- All setup steps
- Commands used
- Concepts learned
- Installed and configured services?

#### Bonus

##### Ask the learner if he implemented any features that may be considered a bonus.

###### + Did the learner implement any bonus features? (You need to be the judge here, is it a bonus feature or not?)
