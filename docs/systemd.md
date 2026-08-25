# systemd — Linux Summary

## 1. What is systemd?

**systemd** is the **system and service manager** used by most modern Linux distributions.

It is responsible for managing the system's lifecycle, especially during:

* Boot
* Service startup
* Service shutdown
* Service supervision
* Dependency management
* Logging
* Timers
* Mounts
* Sockets
* System targets

A simple mental model:

```text
Linux Kernel
     |
     v
  systemd (PID 1)
     |
     +-- Services
     +-- Targets
     +-- Timers
     +-- Mounts
     +-- Sockets
     +-- Dependencies
     +-- System lifecycle
```

---

# 2. systemd vs systemctl

These are **not the same thing**.

```text
systemd  = the manager
systemctl = the command used to communicate with the manager
```

For example:

```bash
systemctl status ssh
```

means:

> Ask systemd for the current status of the SSH service.

And:

```bash
sudo systemctl restart ssh
```

means:

> Tell systemd to restart the SSH service.

---

# 3. systemd is PID 1

During a normal Linux boot, systemd usually becomes:

```text
PID 1
```

You can verify it with:

```bash
ps -p 1 -o pid,comm,args
```

You may see:

```text
PID COMMAND  COMMAND
1   systemd  /sbin/init
```

PID 1 has a special role because it is the first userspace process started by the kernel.

It becomes the parent, directly or indirectly, of many processes on the system.

---

# 4. Services

A **service** is usually a background program managed by systemd.

Examples:

```text
ssh
docker
nginx
mysql
cron
```

Their systemd unit names normally end with:

```text
.service
```

For example:

```text
ssh.service
docker.service
nginx.service
```

The `.service` suffix can usually be omitted:

```bash
systemctl status ssh
```

is equivalent to:

```bash
systemctl status ssh.service
```

---

# 5. The most important service commands

## Check status

```bash
systemctl status SERVICE
```

Example:

```bash
systemctl status ssh
```

---

## Start a service

```bash
sudo systemctl start SERVICE
```

Example:

```bash
sudo systemctl start nginx
```

This starts the service **now**.

It does not necessarily configure it to start at boot.

---

## Stop a service

```bash
sudo systemctl stop SERVICE
```

Example:

```bash
sudo systemctl stop nginx
```

This stops the service **now**.

---

## Restart a service

```bash
sudo systemctl restart SERVICE
```

Example:

```bash
sudo systemctl restart nginx
```

Conceptually:

```text
running
   |
   v
 stop
   |
   v
 start
   |
   v
running
```

---

## Reload a service

```bash
sudo systemctl reload SERVICE
```

This asks the service to reload its configuration without completely stopping and starting it.

Not every service supports reload.

---

# 6. Enable vs Start

This distinction is extremely important.

## `start`

```bash
sudo systemctl start nginx
```

Means:

> Start nginx now.

## `enable`

```bash
sudo systemctl enable nginx
```

Means:

> Configure nginx to start automatically during boot.

Therefore:

```text
start  = now
enable = boot
```

They are independent concepts.

---

# 7. Enable and start together

Instead of:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

you can use:

```bash
sudo systemctl enable --now nginx
```

This means:

```text
Start nginx now
+
Start nginx automatically at boot
```

---

# 8. Disable vs Stop

Similarly:

```bash
sudo systemctl stop nginx
```

means:

> Stop nginx now.

While:

```bash
sudo systemctl disable nginx
```

means:

> Don't automatically start nginx during boot.

To do both:

```bash
sudo systemctl disable --now nginx
```

This means:

```text
Stop it now
+
Don't start it automatically at boot
```

---

# 9. Active vs Enabled

These are different states.

Check whether a service is currently running:

```bash
systemctl is-active nginx
```

Possible result:

```text
active
```

Check whether it is configured to start at boot:

```bash
systemctl is-enabled nginx
```

Possible result:

```text
enabled
```

You can have:

```text
active + enabled
```

Meaning:

> Running now and configured to start at boot.

Or:

```text
active + disabled
```

Meaning:

> Running now but not configured to start automatically at boot.

Or:

```text
inactive + enabled
```

Meaning:

> Not running now but configured to start during boot.

---

# 10. Listing services

Show currently loaded service units:

```bash
systemctl list-units --type=service
```

You can also use:

```bash
systemctl --type=service
```

Show installed service unit files:

```bash
systemctl list-unit-files --type=service
```

Important distinction:

```text
list-units
    |
    +-- units currently loaded/known to systemd

list-unit-files
    |
    +-- unit files installed on the system
```

---

# 11. What is a systemd unit?

systemd manages more than services.

A **unit** is a resource managed by systemd.

Common unit types include:

```text
.service   -> services
.target    -> groups/system states
.timer     -> scheduled tasks
.socket    -> sockets
.mount     -> mount points
.device    -> devices
.path      -> filesystem path monitoring
```

Examples:

```text
ssh.service
multi-user.target
backup.timer
```

Therefore:

```text
systemd
   |
   +-- service units
   +-- target units
   +-- timer units
   +-- socket units
   +-- mount units
   +-- device units
```

---

# 12. Targets

A **target** is a way for systemd to group units and represent a particular system state.

Common targets:

```text
multi-user.target
graphical.target
```

Check the default target:

```bash
systemctl get-default
```

A desktop Linux system commonly uses:

```text
graphical.target
```

A server may commonly use:

```text
multi-user.target
```

List targets:

```bash
systemctl list-units --type=target
```

---

# 13. Unit files

A unit file describes how systemd should manage a unit.

Common locations include:

```text
/etc/systemd/system/
```

and:

```text
/usr/lib/systemd/system/
```

On some distributions:

```text
/lib/systemd/system/
```

is also used.

Generally:

```text
/etc/systemd/system/
    |
    +-- administrator/custom unit configuration

/usr/lib/systemd/system/
    |
    +-- package-provided unit files
```

Custom services are commonly created in:

```text
/etc/systemd/system/
```

---

# 14. Example service file

Example:

```ini
[Unit]
Description=My Java Server
After=network.target

[Service]
User=myuser
ExecStart=/usr/bin/java -jar /opt/myserver/server.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

# 15. `[Unit]`

Example:

```ini
[Unit]
Description=My Java Server
After=network.target
```

### `Description`

Human-readable description:

```ini
Description=My Java Server
```

### `After`

Controls startup ordering:

```ini
After=network.target
```

It means the service should be started after `network.target` in the ordering relationship.

Important:

```text
After=      -> ordering
Requires=  -> dependency
Wants=     -> weaker dependency
```

These concepts are related but not identical.

---

# 16. `[Service]`

This section describes how the actual program should run.

Example:

```ini
[Service]
User=myuser
ExecStart=/usr/bin/java -jar /opt/myserver/server.jar
Restart=on-failure
```

### `User`

Defines which Linux user runs the service:

```ini
User=myuser
```

This is useful for security because applications should generally not run as root unnecessarily.

### `ExecStart`

Defines the command systemd starts:

```ini
ExecStart=/usr/bin/java -jar /opt/myserver/server.jar
```

### `Restart`

Controls automatic restart behavior:

```ini
Restart=on-failure
```

If the application crashes, systemd can restart it.

---

# 17. `[Install]`

Example:

```ini
[Install]
WantedBy=multi-user.target
```

This is used when enabling the service.

For example:

```bash
sudo systemctl enable myserver
```

systemd uses the `[Install]` information to determine how the service should be connected to the boot process.

---

# 18. Creating a custom service

Create the unit file:

```bash
sudo nano /etc/systemd/system/myserver.service
```

Example:

```ini
[Unit]
Description=My Java Server
After=network.target

[Service]
User=myuser
ExecStart=/usr/bin/java -jar /opt/myserver/server.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Then tell systemd to reread its unit files:

```bash
sudo systemctl daemon-reload
```

Enable and start it:

```bash
sudo systemctl enable --now myserver
```

Check it:

```bash
systemctl status myserver
```

---

# 19. `daemon-reload`

When you create or modify a unit file:

```bash
sudo systemctl daemon-reload
```

tells systemd:

> Reread the unit files because their configuration may have changed.

It does **not** restart all services.

Typical workflow:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myserver
```

Use `daemon-reload` when the **unit file itself** has changed.

---

# 20. Logs with `journalctl`

systemd is closely associated with the systemd journal.

The command:

```bash
journalctl
```

is used to inspect logs.

For a specific service:

```bash
sudo journalctl -u myserver
```

The `-u` means:

```text
-u = unit
```

For example:

```bash
sudo journalctl -u ssh
```

---

# 21. Follow logs live

```bash
sudo journalctl -u myserver -f
```

`-f` means follow the log output.

This is similar to:

```bash
tail -f
```

but for the systemd journal.

---

# 22. Show the last 50 entries

```bash
sudo journalctl -u myserver -n 50
```

Meaning:

```text
-u myserver -> logs for myserver
-n 50       -> last 50 entries
```

You can combine it with `-f`:

```bash
sudo journalctl -u myserver -n 50 -f
```

---

# 23. Boot logs

Current boot:

```bash
sudo journalctl -b
```

Previous boot:

```bash
sudo journalctl -b -1
```

This is useful when debugging startup problems.

---

# 24. `systemctl cat`

Show the unit file systemd is using:

```bash
systemctl cat myserver
```

Example:

```bash
systemctl cat ssh
```

This is useful for inspecting service configuration.

---

# 25. `systemctl show`

Show detailed systemd properties:

```bash
systemctl show myserver
```

You may see properties such as:

```text
ActiveState=active
SubState=running
User=myuser
ExecMainPID=1234
Restart=on-failure
```

`status` is easier for humans.

`show` is useful when you need detailed properties or scripting.

---

# 26. Dependencies

Show dependencies of a unit:

```bash
systemctl list-dependencies myserver
```

For a target:

```bash
systemctl list-dependencies multi-user.target
```

This helps you understand how systemd builds the system from individual units and their relationships.

---

# 27. Masking a service

`mask` is stronger than `disable`.

```bash
sudo systemctl mask nginx
```

A masked service is prevented from being started normally through systemd.

Think:

```text
disable
    |
    +-- don't automatically start at boot

mask
    |
    +-- prevent normal starting
```

Undo it with:

```bash
sudo systemctl unmask nginx
```

---

# 28. Restart vs Reload

### Restart

```bash
sudo systemctl restart nginx
```

Usually:

```text
stop
 ↓
start
```

### Reload

```bash
sudo systemctl reload nginx
```

Usually:

```text
keep process running
        ↓
reload configuration
```

Reload is only available when the service supports it.

---

# 29. `systemctl` and processes

Do not confuse a **systemd service** with a **process**.

For example:

```text
systemd
   |
   +-- ssh.service
          |
          +-- sshd process
```

You can inspect the service:

```bash
systemctl status ssh
```

You can inspect processes:

```bash
ps aux
```

And you can inspect listening network sockets:

```bash
ss -tulpn
```

These commands answer different questions.

```text
systemctl
    -> Is the service managed/running?

ps
    -> What processes exist?

ss
    -> What network sockets/ports are listening?
```

---

# 30. A practical debugging workflow

Suppose you have:

```text
myserver.service
```

Start it:

```bash
sudo systemctl start myserver
```

Check it:

```bash
systemctl status myserver
```

If it failed:

```bash
sudo journalctl -u myserver
```

Or:

```bash
sudo journalctl -u myserver -n 100
```

If your server should listen on port `8080`:

```bash
ss -tulpn | grep 8080
```

The general workflow is:

```text
Start service
     |
     v
Check status
     |
     +---- running? ----> check application
     |
     +---- failed?
             |
             v
       journalctl
             |
             v
       identify error
             |
             v
          fix it
             |
             v
    daemon-reload if unit changed
             |
             v
          restart
```

---

# 31. System-level commands

systemctl can also control the system itself.

Reboot:

```bash
sudo systemctl reboot
```

Power off:

```bash
sudo systemctl poweroff
```

---

# 32. Important commands to memorize

## Service management

```bash
systemctl status SERVICE
sudo systemctl start SERVICE
sudo systemctl stop SERVICE
sudo systemctl restart SERVICE
sudo systemctl reload SERVICE
```

## Boot configuration

```bash
sudo systemctl enable SERVICE
sudo systemctl disable SERVICE
sudo systemctl enable --now SERVICE
sudo systemctl disable --now SERVICE
```

## State checking

```bash
systemctl is-active SERVICE
systemctl is-enabled SERVICE
```

## Listing

```bash
systemctl list-units --type=service
systemctl list-unit-files --type=service
```

## Unit inspection

```bash
systemctl cat SERVICE
systemctl show SERVICE
systemctl list-dependencies SERVICE
```

## Unit configuration

```bash
sudo systemctl daemon-reload
```

## Logs

```bash
journalctl -u SERVICE
journalctl -u SERVICE -n 50
journalctl -u SERVICE -f
journalctl -b
journalctl -b -1
```

## Service blocking

```bash
sudo systemctl mask SERVICE
sudo systemctl unmask SERVICE
```

## System

```bash
sudo systemctl reboot
sudo systemctl poweroff
```

---

# 33. The core mental model

Remember this:

```text
                    Linux
                      |
                      v
                  systemd
                   PID 1
                      |
        +-------------+-------------+
        |             |             |
     services       targets       timers
        |
   +----+----+----+
   |         |    |
  ssh      docker nginx
```

And:

```text
systemctl
    |
    +-- communicates with systemd
    |
    +-- starts services
    +-- stops services
    +-- restarts services
    +-- enables services at boot
    +-- disables services at boot
    +-- inspects units
    +-- manages system state
```

The simplest definition to remember is:

> **systemd is the system and service manager; systemctl is the command-line tool used to control and inspect systemd.**

And the most important distinction:

```text
start  = run it now
stop   = stop it now

enable = run it automatically at boot
disable = don't run it automatically at boot
```
