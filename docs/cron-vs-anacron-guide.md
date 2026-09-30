# Automating scripts on Linux: cron vs anacron

This guide covers how to automate a script with **cron**, how **anacron** works, and, most importantly, **when to use which**. The examples were tested on Linux Mint (Debian/Ubuntu family), but the concepts apply to most Linux systems.

---

## 1. The core difference in one paragraph

**cron** runs a job at an **exact time** ("every day at 02:00"). If the computer is off at that moment, the run is **skipped**, and cron never makes it up.

**anacron** runs a job **once per period** ("once a month"), at roughly any time. If the computer was off, anacron **catches up** the next time it is on.

| | cron | anacron |
|---|---|---|
| Schedule style | Exact clock time | "Once every N days" |
| Missed run (PC was off) | **Skipped** | **Runs after next boot** |
| Smallest interval | 1 minute | 1 day |
| Needs the machine always on | Yes, to be reliable | No |
| Who can use it | Every user (own crontab) | Root, system-wide (by default) |
| Remembers past runs | No | Yes (timestamp files) |
| Best for | Servers, precise timing, frequent jobs | Laptops and desktops, daily/weekly/monthly maintenance |
| Config file | `crontab -e`, `/etc/crontab`, `/etc/cron.d/` | `/etc/anacrontab` |

A simple way to remember it:

- **cron asks:** "What time is it? Is it time to run?"
- **anacron asks:** "When did this last run? Has it been long enough?"

---

## 2. cron: automating a script step by step

### 2.1 What cron is

cron is a background service (a *daemon*) that wakes up every minute, reads its schedule tables, and starts any job whose time has come. Check that it's running:

```bash
systemctl status cron
```

### 2.2 Write the script

Put your script somewhere sensible. For system-wide scripts, `/usr/local/bin/` is the convention. For personal ones, `~/bin/` or `~/scripts/` is fine.

Example: a script that backs up a project folder.

```bash
nano ~/scripts/backup.sh
```

```bash
#!/bin/bash
# Backup ~/projects into a dated archive

set -u
export PATH=/usr/local/bin:/usr/bin:/bin

SRC="$HOME/projects"
DEST="$HOME/backups"
LOG="$HOME/backups/backup.log"

mkdir -p "$DEST"

{
  echo "Backup started: $(date)"
  tar -czf "$DEST/projects-$(date +%F).tar.gz" -C "$HOME" projects
  echo "Backup finished: $(date)"
} >> "$LOG" 2>&1
```

Line by line:

- `#!/bin/bash` is the *shebang*. It tells the kernel which interpreter to use.
- `set -u` treats undefined variables as errors, which catches typos.
- `export PATH=...` sets the command search path explicitly. **This is important under cron** (see section 2.5).
- `mkdir -p` creates the folder if it doesn't exist and doesn't complain if it does.
- `$(date +%F)` is *command substitution*. `date +%F` prints `2026-09-30`, so each archive gets a unique name.
- `tar -czf` creates (`c`) a gzip-compressed (`z`) archive into a file (`f`). `-C "$HOME"` changes into that directory first, so the archive stores relative paths.
- `{ ... } >> "$LOG" 2>&1` groups the commands and appends both normal output and errors to a log. Under cron there is no terminal to show output, so **a log file is your only window** into what happened.

Make it executable, then test it manually:

```bash
chmod +x ~/scripts/backup.sh
~/scripts/backup.sh
cat ~/backups/backup.log
```

**Golden rule of automation: never schedule a script you haven't run by hand first.**

### 2.3 Edit your crontab

```bash
crontab -e
```

- This opens **your personal** schedule table (one per user). The first time, it may ask which editor to use. Choose `nano`.
- `crontab -l` lists your entries. `crontab -r` deletes **all** of them (careful, there's no confirmation).
- Jobs in your crontab run as **you**, with your permissions.
- For jobs needing root (like system updates), use `sudo crontab -e`, which edits **root's** crontab.

### 2.4 The cron syntax

Each line has five time fields, then the command:

```
┌───────────── minute        (0-59)
│ ┌─────────── hour          (0-23)
│ │ ┌───────── day of month  (1-31)
│ │ │ ┌─────── month         (1-12)
│ │ │ │ ┌───── day of week   (0-7, both 0 and 7 = Sunday)
│ │ │ │ │
* * * * *  command to run
```

Special characters:

| Symbol | Meaning | Example |
|---|---|---|
| `*` | every value | `* * * * *` = every minute |
| `,` | a list | `0 8,18 * * *` = at 08:00 and 18:00 |
| `-` | a range | `0 9 * * 1-5` = 09:00, Monday to Friday |
| `/` | a step | `*/15 * * * *` = every 15 minutes |

Common examples:

```
# Every day at 02:30
30 2 * * * /home/bendoe/scripts/backup.sh

# Every Sunday at 03:00
0 3 * * 0 /home/bendoe/scripts/backup.sh

# 1st of every month at 10:00
0 10 1 * * /home/bendoe/scripts/backup.sh

# Every 15 minutes
*/15 * * * * /home/bendoe/scripts/check.sh
```

Shortcut keywords also exist:

| Keyword | Equivalent |
|---|---|
| `@reboot` | once, at boot |
| `@hourly` | `0 * * * *` |
| `@daily` | `0 0 * * *` |
| `@weekly` | `0 0 * * 0` |
| `@monthly` | `0 0 1 * *` |

You can validate a schedule visually at [crontab.guru](https://crontab.guru).

Add the backup job with this line in `crontab -e`:

```
30 2 * * * /home/bendoe/scripts/backup.sh
```

Use the **absolute path** (`/home/bendoe/...`). Don't use `~` or relative paths.

### 2.5 The cron environment (where most bugs come from)

cron does **not** run your job the way your terminal does. It starts a bare-bones environment:

- **A minimal `PATH`** (usually just `/usr/bin:/bin`), so commands in `/usr/local/bin` or tools like `docker` may not be found.
- **`/bin/sh` instead of Bash** by default. Bash-only features can fail in a crontab line.
- **No terminal**, so anything that prompts for input will hang or fail.
- **A different working directory** (your home), so always use absolute paths.
- **None of your `.bashrc` settings** (aliases, environment variables, nvm/sdkman paths).

The classic symptom: *"it works in my terminal but not in cron."* The fixes:

1. Use absolute paths everywhere.
2. Set `PATH` inside the script (as done above).
3. Set variables at the top of the crontab if needed:

```
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=""

30 2 * * * /home/bendoe/scripts/backup.sh
```

`MAILTO=""` stops cron from trying to email output to you (local mail is rarely set up).

### 2.6 A special gotcha: the `%` character

In a crontab line, an unescaped `%` means "newline" and breaks the command. So this is wrong:

```
0 1 * * * tar -czf /backup/proj-$(date +%F).tar.gz /projects
```

Either escape it (`date +\%F`) or, better, put the logic in a script file, as we did. That's another reason to keep crontab lines short and call a script.

### 2.7 Redirecting output inside the crontab line

If you don't want to log inside the script, redirect in the line itself:

```
30 2 * * * /home/bendoe/scripts/backup.sh >> /home/bendoe/backups/cron.log 2>&1
```

- `>>` appends (a single `>` would overwrite).
- `2>&1` sends error output (stream 2) to the same place as normal output (stream 1).

### 2.8 Verify and debug cron

Did cron even start the job?

```bash
grep CRON /var/log/syslog | tail -20
# or on systemd systems:
journalctl -u cron --since today
```

You'll see lines like `CMD (/home/bendoe/scripts/backup.sh)`. That proves cron *launched* it. Whether the script itself succeeded is what **your log file** tells you.

Debugging checklist when a job doesn't work:

1. Does the script run by hand with the full path?
2. Is it executable (`ls -l` shows `x`)?
3. Does it use absolute paths and a set `PATH`?
4. Is the log file being written? Read it.
5. Is the `cron` service running?
6. Did you edit the crontab of the right user (yours vs root)?

### 2.9 Other places cron reads jobs from

| Location | Notes |
|---|---|
| `crontab -e` | Per-user table. No user field. |
| `/etc/crontab` | System table. **Has an extra user field** after the time fields. |
| `/etc/cron.d/` | Drop-in files, same format as `/etc/crontab` (with user field). Good for packages and tidy setups. |
| `/etc/cron.hourly/`, `daily/`, `weekly/`, `monthly/` | Drop in an **executable script** (no dot in the name) and it runs at that interval. |

Example of the system format, where `root` is the user field:

```
30 2 * * * root /usr/local/bin/monthly-update.sh
```

### 2.10 The limitation

Say you scheduled `30 2 * * *` and your laptop is asleep or powered off at 02:30. **That backup is skipped for the day**, and cron doesn't try again later. For a server that's always on, this is fine. For a laptop, it's a real problem, and it's exactly what anacron solves.

---

## 3. anacron: automating "once per period" jobs

### 3.1 The idea

anacron was designed for machines that are **not on 24/7**. Instead of asking "is it 02:30?", it asks "has this job run in the last N days?" and if not, it runs it.

Each job has a **timestamp file** in `/var/spool/anacron/`. When anacron runs, it compares today's date with the timestamp:

- Older than the job's period? **Run it, then update the timestamp.**
- Recent enough? **Skip it.**

You can see this bookkeeping yourself:

```bash
ls -l /var/spool/anacron/
cat /var/spool/anacron/cron.monthly     # a date like 20260930
```

### 3.2 The anacrontab format

The config file is `/etc/anacrontab`. On Mint it looks like this:

```
SHELL=/bin/sh
HOME=/root
LOGNAME=root

# period  delay  job-id        command
1         5      cron.daily    run-parts --report /etc/cron.daily
7         10     cron.weekly   run-parts --report /etc/cron.weekly
@monthly  15     cron.monthly  run-parts --report /etc/cron.monthly
```

Each job line has four fields:

| Field | Meaning |
|---|---|
| **period** | How often, **in days** (`1`, `7`, `30`), or `@daily`, `@weekly`, `@monthly`, `@yearly` |
| **delay** | Minutes to wait after anacron starts before running the job. Spreads out load so boot isn't flooded. |
| **job-id** | A unique name. It's also the **name of the timestamp file**. No spaces. |
| **command** | The command to run. |

So the `cron.monthly` line means: "once a month, 15 minutes after anacron starts, run every script in `/etc/cron.monthly/`."

### 3.3 Two ways to use anacron

**Way 1: drop a script in `/etc/cron.daily/`, `weekly/` or `monthly/`** (the easy way, and what we used for the system updates)

```bash
sudo ln -s /usr/local/bin/monthly-update.sh /etc/cron.monthly/monthly-update
```

- Executable file, **no dot in the name** (the tool that runs the folder, `run-parts`, ignores names like `something.sh`).
- The stock `anacrontab` lines above already trigger these folders, so nothing else to configure.
- List what would run without running it: `run-parts --test /etc/cron.monthly`.

**Way 2: add your own line to `/etc/anacrontab`** (for a custom period)

```bash
sudo nano /etc/anacrontab
```

Add at the bottom:

```
# Backup every 3 days, 20 minutes after anacron starts
3   20   my-backup   /usr/local/bin/backup.sh
```

Now `backup.sh` runs whenever 3 or more days have passed since its last run, even if the laptop was off on the "right" day. Only whole days are supported; you can't say "every 6 hours."

### 3.4 Handy anacrontab settings

You can add variables at the top of `/etc/anacrontab`:

```
START_HOURS_RANGE=8-22     # only start jobs between 08:00 and 22:00
RANDOM_DELAY=30            # add up to 30 random minutes to each delay
```

`START_HOURS_RANGE` is useful so a heavy job doesn't kick off in the middle of the night on a machine that happens to be on.

### 3.5 Useful anacron commands

```bash
sudo anacron -T      # test the config file for syntax errors
sudo anacron -u      # update timestamps to "now" without running jobs
sudo anacron -f -n   # force ALL jobs to run right now (ignores timestamps and delays)
```

Careful with `-f -n`, because it runs **every** job in the anacrontab immediately, including daily and weekly cleanups. To test just your own script, run it directly instead.

### 3.6 When does anacron itself start?

anacron doesn't run continuously. It is started:

- **At boot** (shortly after the system comes up), by a systemd service or timer.
- **Periodically by cron** (a small entry in `/etc/cron.d/anacron`), so it also fires on machines that stay on for days.

On Debian and Ubuntu-based systems those triggers check that the machine is on **AC power**. On a **laptop running on battery, anacron jobs are skipped** until you plug in. It's a deliberate battery-saving choice, so a monthly update on a laptop waits for the charger.

### 3.7 The limitations

- **Granularity is one day.** No hourly or every-15-minute jobs.
- **No exact time.** You can't say "at 02:30." It runs when anacron gets to it.
- **System-wide by default.** Jobs run as root, and normal users can't easily add their own.
- **Battery caveat** on laptops (see above).

---

## 4. cron and anacron together

They aren't rivals, and on most desktop Linux systems they work as a team. The folders `/etc/cron.daily`, `weekly` and `monthly` are triggered by **both**:

- On an always-on server, **cron** runs them at fixed times (from `/etc/crontab`).
- On a desktop or laptop, **anacron** runs them, catching up after downtime.

The `0anacron` file you saw in `/etc/cron.monthly/` is the glue. It runs first and refreshes anacron's timestamp, so the same job isn't run twice by both tools.

**Decision guide:**

| Your situation | Use |
|---|---|
| Must run at an exact time (e.g. 02:30, weekdays only) | **cron** |
| Runs more than once a day (every 5 minutes, hourly) | **cron** |
| Machine is a laptop or desktop that gets turned off | **anacron** |
| Maintenance like updates, cleanup, backups, and it's fine to run "sometime this month" | **anacron** |
| A per-user job with no root access | **cron** (`crontab -e`) |
| Run something once at every boot | **cron** (`@reboot`) |

---

## 5. A third option: systemd timers

Modern Linux also has **systemd timers**, which combine the strengths of both. `OnCalendar=` gives exact times like cron, and `Persistent=true` gives catch-up like anacron:

```ini
[Timer]
OnCalendar=monthly
Persistent=true
```

They also give you centralized logs (`journalctl`) and don't have the battery restriction. The trade-off is that they need two files (a `.service` and a `.timer`) instead of one line. You already have one running for your system updates.

| | cron | anacron | systemd timer |
|---|---|---|---|
| Exact times | Yes | No | Yes |
| Catches up missed runs | No | Yes | Yes (`Persistent=true`) |
| Setup effort | One line | One line / one file | Two files |
| Built-in logging | No | Minimal | Yes (`journalctl`) |
| Skips on battery | No | Yes (Debian family) | No |

---

## 6. Quick reference

```bash
# cron
crontab -e                          # edit your schedule
crontab -l                          # list your schedule
sudo crontab -e                     # edit root's schedule
systemctl status cron               # is the service running?
journalctl -u cron --since today    # did cron start my job?

# anacron
cat /etc/anacrontab                 # view its schedule
ls -l /var/spool/anacron/           # last-run timestamps
run-parts --test /etc/cron.monthly  # what would the monthly folder run?
sudo anacron -T                     # validate config

# general
chmod +x script.sh                  # make a script executable
tail -f /path/to/log                # watch a log live
```

**The five rules of reliable automation:**

1. Test the script by hand before scheduling it.
2. Use absolute paths, and set `PATH` inside the script.
3. Log everything to a file (`>> log 2>&1`).
4. Never rely on interactive prompts.
5. After scheduling, **verify** the job ran (check the log the next day).
