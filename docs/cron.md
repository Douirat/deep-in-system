# Automatic monthly updates on Linux Mint: the full guide

The setup has three parts:

1. **The script** does the actual updating.
2. **The scheduler** decides when the script runs.
3. **The log** lets you check what happened afterwards.

Each Linux concept is explained as it comes up.

---

## Part 1: The update script

### Why a script?

A script is a text file of shell commands that run in order. Putting the updates in a script gives you one thing to test by hand and one thing to schedule. Scheduling ten separate commands would be messier.

### Where to put it

By convention, scripts you write for the whole system go in `/usr/local/bin/`. Files there are ones you installed yourself, as opposed to files managed by `apt` (which live in `/usr/bin/`). The directory is also in everyone's `PATH`, so the script can be run by name.

```bash
sudo nano /usr/local/bin/monthly-update.sh
```

- `sudo` runs the command as root (the administrator), which is required to write into `/usr/local/bin/`.
- `nano` is a simple terminal text editor. Save with `Ctrl+O` then `Enter`, and quit with `Ctrl+X`.

### The script

```bash
#!/bin/bash
# Monthly system update for Linux Mint

set -u
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export DEBIAN_FRONTEND=noninteractive

LOG="/var/log/monthly-update.log"

# Must run as root
if [ "$EUID" -ne 0 ]; then
  echo "This script must be run as root (use sudo)." >&2
  exit 1
fi

{
  echo "=============================="
  echo "Update started: $(date)"
  echo "=============================="

  # Safety net: snapshot before changing anything
  if command -v timeshift >/dev/null 2>&1; then
    echo ">> Creating Timeshift snapshot"
    timeshift --create --comments "Before monthly update" --tags M
  fi

  echo ">> APT update & upgrade"
  apt-get update
  apt-get -y \
    -o Dpkg::Options::="--force-confdef" \
    -o Dpkg::Options::="--force-confold" \
    dist-upgrade

  echo ">> Removing unused packages"
  apt-get -y autoremove --purge
  apt-get -y autoclean

  if command -v flatpak >/dev/null 2>&1; then
    echo ">> Flatpak update"
    flatpak update -y --noninteractive
    flatpak uninstall --unused -y --noninteractive
  fi

  if command -v snap >/dev/null 2>&1; then
    echo ">> Snap refresh"
    snap refresh
  fi

  if [ -f /var/run/reboot-required ]; then
    echo ">> REBOOT REQUIRED (kernel or core library updated)"
  fi

  echo "Update finished: $(date)"
} >> "$LOG" 2>&1
```

### Line-by-line explanation

**`#!/bin/bash`** is called the *shebang*. When you run a file, the kernel reads its first line to learn which program should interpret it. Without it, the system wouldn't know the file is a Bash script.

**`set -u`** makes Bash treat use of an undefined variable as an error instead of silently substituting an empty string. That catches typos such as `$LOGG`. (Many scripts also use `set -e` to stop at the first failure. It is left out on purpose here, so that a failing Flatpak update doesn't prevent the APT cleanup from running.)

**`export PATH=...`** matters for automation. `PATH` is the list of directories the shell searches when you type a command name. Cron and systemd start jobs with a very small `PATH`, much smaller than your terminal's. Setting it explicitly avoids the classic bug of a script that works when you run it by hand but fails from cron with "command not found".

**`export DEBIAN_FRONTEND=noninteractive`** is an environment variable that tells `apt`/`dpkg` never to open interactive prompts (like blue configuration dialogs). A scheduled job has no keyboard attached, so any prompt would hang it forever.

**`LOG="/var/log/monthly-update.log"`** is a variable. `/var/log/` is the standard directory for logs. You use `"$LOG"` later, and the quotes protect against spaces in paths.

**`if [ "$EUID" -ne 0 ]`** checks the effective user ID. Root always has ID 0, so `-ne 0` ("not equal to zero") means you're not root. The script then prints an error to `stderr` (`>&2`) and stops with `exit 1`. Exit codes matter in automation: `0` means success and anything else means failure.

**`{ ... } >> "$LOG" 2>&1`** groups all the commands between the braces and redirects their combined output:

- `>>` *appends* to a file (a single `>` would overwrite it each time).
- `2>&1` means "send stderr (file descriptor 2) to the same place as stdout (file descriptor 1)". Without it, error messages would be lost, because cron has no terminal to show them.

**`$(date)`** is *command substitution*. The shell runs `date` and pastes its output in place.

**`command -v timeshift >/dev/null 2>&1`** checks whether a program exists. `command -v` prints the program's path if found. The `>/dev/null 2>&1` part throws all output away, since only the exit code matters to the `if`. `/dev/null` is a special "black hole" file.

**Timeshift** takes a system snapshot you can roll back to if an update breaks something. `--tags M` labels it as a monthly snapshot so Timeshift's retention rules handle it correctly.

**`apt-get update`** refreshes the list of available packages, and it upgrades nothing. **`apt-get dist-upgrade`** then installs the new versions. Unlike plain `upgrade`, it may add or remove packages when a new version needs it (kernels, for example). This is the safe choice on Mint.

- `-y` answers "yes" to confirmations.
- The two `-o Dpkg::Options::=...` flags tell `dpkg`, if a package wants to replace a config file you've edited, to keep your version (`--force-confold`) and use defaults only for untouched files (`--force-confdef`). This prevents a prompt that would block the job.

**`autoremove --purge`** deletes packages that were installed as dependencies and are no longer needed, including their config files. **`autoclean`** deletes old downloaded `.deb` files that can no longer be downloaded, so disk space doesn't slowly fill up.

**Flatpak and Snap** are separate package systems with their own updaters. `apt` doesn't touch them, so they get their own steps, guarded by `command -v` in case they're not installed.

**`/var/run/reboot-required`** is a flag file that Ubuntu-family systems create when an update (like a new kernel) needs a restart. The script only *reports* this. Rebooting automatically could destroy unsaved work.

### Make it executable

```bash
sudo chmod +x /usr/local/bin/monthly-update.sh
```

Linux files have permission bits for read, write and execute. A new file isn't executable by default. `chmod +x` adds the execute bit so the file can run as a program. You can inspect permissions with `ls -l /usr/local/bin/monthly-update.sh` (you'll see an `x` in the permission string).

### Test it by hand first

Always run an automated script manually before trusting the scheduler:

```bash
sudo /usr/local/bin/monthly-update.sh
tail -50 /var/log/monthly-update.log
```

`tail -50` shows the last 50 lines of the file. Add `-f` (`tail -f`) to watch a log live as it grows.

---

## Part 2: The scheduler

You have two good options. **Pick one, not both**, or the update will run twice.

### Concept: why "cron" alone isn't enough

**Cron** is a background service (daemon) that runs commands at fixed times. Its weakness is that if the computer is off at the scheduled time, the job is simply skipped. For a desktop that isn't on 24/7, that's a problem.

Two tools fix this by remembering missed runs:

- **anacron** works with cron: "run this job once per period, whenever the machine is on".
- **systemd timers** are the modern replacement, with a `Persistent=true` setting that does the same.

### Option A: `/etc/cron.monthly/` (simplest)

Debian and Ubuntu-based systems (including Mint) have folders like `/etc/cron.daily/`, `/etc/cron.weekly/` and `/etc/cron.monthly/`. Any executable file placed there is run once per period. On a desktop, anacron handles them, so a missed month is made up after the next boot.

```bash
sudo ln -s /usr/local/bin/monthly-update.sh /etc/cron.monthly/monthly-update
```

- `ln -s` creates a *symbolic link* (a shortcut). The real script stays in one place, and the link points to it. If you edit the script, the scheduled version updates too.
- **The link name has no `.sh`** because the tool that runs these folders (`run-parts`) ignores names containing dots. This is a very common gotcha.

Verify that it will be picked up:

```bash
run-parts --test /etc/cron.monthly
```

`--test` lists what *would* run without running anything. You should see your job listed.

Make sure anacron is installed and its config is valid:

```bash
sudo apt install anacron
sudo anacron -T
```

`-T` tests the configuration file (`/etc/anacrontab`) for syntax errors.

**How anacron tracks runs:** it stores a timestamp per period in `/var/spool/anacron/`. Once the last run is more than a month old, the next boot triggers the job (after a short delay). You can see this with `cat /var/spool/anacron/cron.monthly` after the first run.

**Laptop caveat:** on Debian-based systems anacron skips jobs while on battery, so on a laptop the job waits until you're plugged in.

### Option B: a systemd timer (best for learning)

systemd is the init system on Mint. It manages services, and a **timer** is a unit that starts a **service** on a schedule. You need two files.

If you already made the symlink in Option A, remove it first:

```bash
sudo rm /etc/cron.monthly/monthly-update
```

**The service file**, which describes *what* to run:

```bash
sudo nano /etc/systemd/system/monthly-update.service
```

```ini
[Unit]
Description=Monthly system update
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monthly-update.sh
```

- `/etc/systemd/system/` is where your own unit files live.
- `Wants=` and `After=network-online.target` ensure the update doesn't start before you have internet, which matters when a missed job fires right at boot.
- `Type=oneshot` means the program runs once and exits, as opposed to a long-running daemon.
- `ExecStart=` is the command to run.

**The timer file**, which describes *when*:

```bash
sudo nano /etc/systemd/system/monthly-update.timer
```

```ini
[Unit]
Description=Run monthly system update

[Timer]
OnCalendar=monthly
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

- The timer's name must match the service's (`monthly-update`) so systemd links them.
- `OnCalendar=monthly` means the 1st of each month at 00:00. You can write precise schedules too, like `OnCalendar=*-*-01 10:00:00`.
- `Persistent=true` stores the last run time on disk and runs the job at boot if a run was missed. This is the "computer was off" solution.
- `RandomizedDelaySec=10min` adds a random delay so the job doesn't start at the exact same instant as other things.
- `WantedBy=timers.target` makes the timer start automatically at boot when enabled.

Activate it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now monthly-update.timer
```

- `daemon-reload` tells systemd to re-read unit files from disk, which you must do after creating or editing them.
- `enable` makes it start at every boot, and `--now` also starts it immediately.

**Useful commands to inspect it:**

```bash
systemctl list-timers monthly-update.timer     # when it last ran and will next run
systemctl status monthly-update.service        # result of the last run
journalctl -u monthly-update.service           # full logs (journald)
sudo systemctl start monthly-update.service    # run the update right now, via systemd
```

`journalctl` reads systemd's central log, so with this option you have two logs: journald and your `/var/log/monthly-update.log`.

### Bonus: classic cron syntax (worth knowing)

If you ever want an exact schedule with plain cron, edit root's crontab:

```bash
sudo crontab -e
```

`crontab -e` opens your personal schedule table (`-l` lists it). Each line has five time fields and then a command:

```
┌───────── minute (0-59)
│ ┌─────── hour (0-23)
│ │ ┌───── day of month (1-31)
│ │ │ ┌─── month (1-12)
│ │ │ │ ┌─ day of week (0-7, Sunday = 0 or 7)
│ │ │ │ │
0 10 1 * * /usr/local/bin/monthly-update.sh
```

That means "at 10:00 on the 1st of every month". `*` means "every value". Two other examples: `*/15 * * * *` runs every 15 minutes, and `0 3 * * 0` runs at 03:00 every Sunday. Remember that this form has the skip-if-off weakness, so for your use case A or B is the better fit.

---

## Part 3: Maintenance and habits

**Check the log occasionally:**

```bash
less /var/log/monthly-update.log
```

`less` lets you scroll a file (`q` to quit, `/word` to search). Look for "REBOOT REQUIRED" and for errors.

**Keep the log from growing forever** with *logrotate*, the standard log-management tool:

```bash
sudo nano /etc/logrotate.d/monthly-update
```

```
/var/log/monthly-update.log {
    rotate 6
    monthly
    compress
    missingok
    notifempty
}
```

This keeps 6 compressed old logs and deletes older ones.

**What isn't covered.** The script updates what the system's package managers know about. It does *not* update:

- AppImages, or software installed by hand.
- Language tooling you installed yourself (`npm -g`, `go install`, `cargo install`, `pip --user`).
- Docker images. You could add `docker image prune -f` to clean unused ones, but avoid auto-pulling new versions, since that can break your projects.
- Mint upgrades to a new major release (22 to 23). That's a deliberate, manual process.

---

## Summary checklist

1. Create `/usr/local/bin/monthly-update.sh` and paste the script.
2. `sudo chmod +x /usr/local/bin/monthly-update.sh`
3. Test it: `sudo /usr/local/bin/monthly-update.sh`, then read the log.
4. Schedule it with **either** Option A (symlink into `/etc/cron.monthly/`) **or** Option B (systemd service and timer).
5. Verify with `run-parts --test /etc/cron.monthly` or `systemctl list-timers`.
6. Once in a while, glance at the log and reboot if it says so.

**Exercise:** since the goal is to learn automation, try extending the script yourself. Add a line that emails you the log, or one that runs `docker image prune -f`.