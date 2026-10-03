# WordPress Database Backup with Cron — Audit Manual

## 1. Objective

The objective is to automate a daily backup of the WordPress MySQL database.

The complete process is:

```text
MySQL database
      ↓
mysqldump
      ↓
temporary .sql file
      ↓
tar + gzip
      ↓
.tar.gz backup archive
      ↓
temporary .sql file deleted
      ↓
backup log updated
```

Cron is then used to execute the backup script automatically every day.

---

# 2. Configuration Used

| Item             | Value                      |
| ---------------- | -------------------------- |
| Database         | `wordpress`                |
| MySQL user       | `wordpress_user`           |
| Backup directory | `/backup`                  |
| Backup script    | `/usr/local/bin/backup.sh` |
| Backup log       | `/var/log/backup.log`      |
| Cron schedule    | Every day at 02:00         |

---

# 3. Verify the MySQL Database

First verify that the MySQL user can access the WordPress database.

```bash
mysql -u wordpress_user -p wordpress
```

Enter the MySQL password.

If authentication succeeds, you should see:

```text
mysql>
```

Exit MySQL:

```sql
exit;
```

### Meaning

```text
mysql -u wordpress_user -p wordpress
      │   │               │
      │   │               └── Database name
      │   └────────────────── MySQL username
      └────────────────────── MySQL client
```

`-p` tells MySQL to ask for the password.

---

# 4. `mysqldump`

`mysqldump` creates a logical SQL backup of a MySQL database.

Example:

```bash
mysqldump --no-tablespaces -u wordpress_user -p'12345678' wordpress > /backup/wordpress-2026-10-01.sql
```

### Breakdown

```text
mysqldump
```

Program used to export the database.

```text
--no-tablespaces
```

Prevents tablespace information from being dumped.

This avoids requiring the `PROCESS` privilege in this setup.

```text
-u wordpress_user
```

Specifies the MySQL user.

```text
-p'12345678'
```

Supplies the password directly.

There is intentionally no space between `-p` and the password.

```text
wordpress
```

The database being backed up.

```text
> /backup/wordpress-2026-10-01.sql
```

Redirects the SQL output into a file.

---

# 5. The Date Variable

The script uses:

```bash
DATE=$(date +%F)
```

`date +%F` produces:

```text
YYYY-MM-DD
```

For example:

```text
2026-10-01
```

Therefore:

```bash
DATE=$(date +%F)
```

creates:

```text
DATE=2026-10-01
```

Then:

```bash
$DATE
```

can be used in filenames.

For example:

```bash
/backup/wordpress-$DATE.sql
```

becomes:

```text
/backup/wordpress-2026-10-01.sql
```

---

# 6. Create the Backup Script

Create the script:

```bash
sudo nano /usr/local/bin/backup.sh
```

Put this inside:

```bash
#!/usr/bin/bash

DATE=$(date +%F)

mysqldump --no-tablespaces -u wordpress_user -p'12345678' wordpress > /backup/wordpress-$DATE.sql

tar -czf /backup/wordpress_backup-$DATE.tar.gz /backup/wordpress-$DATE.sql

rm /backup/wordpress-$DATE.sql

echo "wordpress backup was created on $DATE" >> /var/log/backup.log
```

Save with:

```text
Ctrl + O
Enter
Ctrl + X
```

---

# 7. Explanation of the Script

## Line 1

```bash
#!/usr/bin/bash
```

This is the shebang.

It tells Linux to execute the script using Bash.

---

## Line 2

```bash
DATE=$(date +%F)
```

Gets today's date and stores it in the variable `DATE`.

---

## Line 3

```bash
mysqldump --no-tablespaces -u wordpress_user -p'12345678' wordpress > /backup/wordpress-$DATE.sql
```

Creates the SQL backup.

For example:

```text
/backup/wordpress-2026-10-01.sql
```

The `>` operator redirects the output into the file.

---

## Line 4

```bash
tar -czf /backup/wordpress_backup-$DATE.tar.gz /backup/wordpress-$DATE.sql
```

Creates the compressed backup archive.

For example:

```text
/backup/wordpress_backup-2026-10-01.tar.gz
```

### Tar options

```text
-c    create an archive
-z    use gzip compression
-f    specify the archive filename
```

Therefore:

```bash
tar -czf
```

means:

> Create a gzip-compressed tar archive.

---

# 8. What Is `.tar.gz`?

A `.tar.gz` file combines two concepts.

### `.tar`

An archive containing one or more files.

### `.gz`

Gzip compression.

Therefore:

```text
wordpress_backup-2026-10-01.tar.gz
```

means:

> A tar archive compressed using gzip.

---

# 9. Delete the Temporary SQL File

The script contains:

```bash
rm /backup/wordpress-$DATE.sql
```

The SQL file is temporary.

After it has been placed inside the archive, it is deleted.

The final backup directory therefore contains the compressed backup rather than both files.

---

# 10. Backup Log

The script ends with:

```bash
echo "wordpress backup was created on $DATE" >> /var/log/backup.log
```

`echo` produces text.

The operator:

```text
>>
```

means:

> Append to the file.

It does not overwrite previous log entries.

Example:

```text
wordpress backup was created on 2026-10-01
wordpress backup was created on 2026-10-02
wordpress backup was created on 2026-10-03
```

---

# 11. Make the Script Executable

Run:

```bash
sudo chmod +x /usr/local/bin/backup.sh
```

`chmod` changes file permissions.

`+x` adds executable permission.

Check:

```bash
ls -l /usr/local/bin/backup.sh
```

You should see `x` permissions, for example:

```text
-rwxr-xr-x
```

---

# 12. Test the Script Manually

Before configuring cron, run:

```bash
sudo /usr/local/bin/backup.sh
```

You may see:

```text
mysqldump: [Warning] Using a password on the command line interface can be insecure.
tar: Removing leading `/' from member names
```

These messages are not necessarily errors.

## MySQL warning

```text
Using a password on the command line interface can be insecure.
```

This happens because the password is directly included in the command.

The backup can still execute successfully.

For a production system, credentials should be stored using a protected configuration rather than directly in the script.

## Tar message

```text
tar: Removing leading `/' from member names
```

This happens because the source path is absolute:

```text
/backup/wordpress-2026-10-01.sql
```

Tar removes the leading `/` when storing the path inside the archive.

This is normal.

---

# 13. Check the Backup

Run:

```bash
sudo ls -lh /backup/
```

You should see something similar to:

```text
wordpress_backup-2026-10-01.tar.gz
```

The temporary `.sql` file should normally not be present because the script deletes it.

---

# 14. Verify the Archive Contents

This is an important audit step.

Run:

```bash
sudo tar -tzf /backup/wordpress_backup-2026-10-01.tar.gz
```

Expected result:

```text
backup/wordpress-2026-10-01.sql
```

This confirms that the SQL dump is actually inside the archive.

### Options

```text
-t    list archive contents
-z    decompress gzip
-f    specify archive file
```

---

# 15. Check the Backup Log

Run:

```bash
sudo cat /var/log/backup.log
```

Expected:

```text
wordpress backup was created on 2026-10-01
```

The log records executions of the script.

Important:

The current simple script does not verify every previous command before writing the success message. A production-quality script should check command failures before declaring success.

---

# 16. Configure Cron

Cron is the Linux scheduler used to automatically execute commands at specified times.

Open the root crontab:

```bash
sudo crontab -e
```

Add:

```cron
0 2 * * * /usr/local/bin/backup.sh
```

Save and exit.

---

# 17. Understand the Cron Expression

The cron line is:

```cron
0 2 * * * /usr/local/bin/backup.sh
```

The five scheduling fields are:

```text
┌──────── minute
│ ┌────── hour
│ │ ┌──── day of month
│ │ │ ┌── month
│ │ │ │ ┌ day of week
│ │ │ │ │
0 2 * * * /usr/local/bin/backup.sh
```

Meaning:

```text
0     = minute 0
2     = hour 2
*     = every day of the month
*     = every month
*     = every day of the week
```

Therefore:

> Run the backup script every day at 02:00.

---

# 18. Verify the Cron Job

Run:

```bash
sudo crontab -l
```

Expected:

```cron
0 2 * * * /usr/local/bin/backup.sh
```

This confirms that the job has been installed in root's crontab.

---

# 19. Verify the Cron Service

Check that the cron service is running:

```bash
sudo systemctl status cron --no-pager
```

Or use:

```bash
systemctl is-active cron
```

Expected:

```text
active
```

---

# 20. Complete Audit Procedure

Use the following commands in this order.

## Step 1 — Verify MySQL access

```bash
mysql -u wordpress_user -p wordpress
```

Then:

```sql
exit;
```

---

## Step 2 — Display the backup script

```bash
sudo cat /usr/local/bin/backup.sh
```

---

## Step 3 — Check script permissions

```bash
ls -l /usr/local/bin/backup.sh
```

---

## Step 4 — Run the backup manually

```bash
sudo /usr/local/bin/backup.sh
```

---

## Step 5 — Check the backup directory

```bash
sudo ls -lh /backup/
```

---

## Step 6 — Inspect the archive

Replace the date if necessary:

```bash
sudo tar -tzf /backup/wordpress_backup-2026-10-01.tar.gz
```

Expected:

```text
backup/wordpress-2026-10-01.sql
```

---

## Step 7 — Check the backup log

```bash
sudo cat /var/log/backup.log
```

---

## Step 8 — Check the cron job

```bash
sudo crontab -l
```

Expected:

```cron
0 2 * * * /usr/local/bin/backup.sh
```

---

## Step 9 — Check the cron service

```bash
systemctl is-active cron
```

Expected:

```text
active
```

---

# 21. Final Architecture

The complete system is:

```text
                    CRON
                     │
                     │ Every day at 02:00
                     ▼
          /usr/local/bin/backup.sh
                     │
                     ▼
                 mysqldump
                     │
                     ▼
          wordpress-DATE.sql
                     │
                     ▼
                tar + gzip
                     │
                     ▼
       wordpress_backup-DATE.tar.gz
                     │
                     ▼
             Remove temporary SQL
                     │
                     ▼
             /var/log/backup.log
```

---

# 22. Important Commands to Remember

## MySQL login

```bash
mysql -u wordpress_user -p wordpress
```

## Database dump

```bash
mysqldump --no-tablespaces -u wordpress_user -p'PASSWORD' wordpress > /backup/wordpress-$DATE.sql
```

## Create compressed archive

```bash
tar -czf archive.tar.gz file.sql
```

## List archive contents

```bash
tar -tzf archive.tar.gz
```

## Delete a file

```bash
rm file
```

## Make a script executable

```bash
chmod +x script.sh
```

## Display a file

```bash
cat file
```

## List files with sizes

```bash
ls -lh /backup/
```

## Edit root's cron jobs

```bash
sudo crontab -e
```

## Display root's cron jobs

```bash
sudo crontab -l
```

## Check cron service

```bash
systemctl is-active cron
```

---

# 23. Common Mistakes

## Mistake 1 — Incorrect date syntax

Wrong:

```bash
DATE=(date +%F)
```

Correct:

```bash
DATE=$(date +%F)
```

`$(...)` is command substitution.

---

## Mistake 2 — Incorrect password syntax

When supplying the password directly to `-p`, do not put a space between `-p` and the password.

Correct:

```bash
-p'12345678'
```

However, putting passwords directly into commands is insecure.

---

## Mistake 3 — Incorrect filename

Be consistent.

Correct:

```text
wordpress-2026-10-01.sql
```

and:

```text
wordpress_backup-2026-10-01.tar.gz
```

A typo such as:

```text
worpress
```

creates a different filename.

---

## Mistake 4 — Incorrect cron expression

Wrong:

```cron
2 * * * /usr/local/bin/backup.sh
```

Correct:

```cron
0 2 * * * /usr/local/bin/backup.sh
```

Cron requires five scheduling fields before the command.

---

## Mistake 5 — Using `$DATE` manually without defining it

Inside the script:

```bash
DATE=$(date +%F)
```

creates the variable.

But if you open a new terminal and type:

```bash
rm /backup/wordpress-$DATE.sql
```

`DATE` may not exist in that shell.

The result could become:

```text
/backup/wordpress-.sql
```

The script itself should handle the temporary SQL file.

---

# 24. What to Explain During the Audit

Be able to explain these points:

### `mysqldump`

Creates a logical SQL backup of the MySQL database.

### `DATE=$(date +%F)`

Gets the current date and stores it in the `DATE` variable.

### `>`

Redirects command output into a file.

### `tar -czf`

Creates a gzip-compressed archive.

### `rm`

Deletes the temporary SQL file after it has been archived.

### `>>`

Appends information to the backup log without overwriting previous entries.

### `chmod +x`

Makes the backup script executable.

### Cron

Schedules commands to run automatically.

### `0 2 * * *`

Means:

> Every day at 02:00.

### `tar -tzf`

Lists the contents of a compressed tar archive and allows verification that the SQL dump exists inside it.

### `crontab -l`

Displays the installed cron jobs.

### `systemctl is-active cron`

Checks whether the cron service is running.

---

# 25. Important Limitation of the Current Script

The current script is sufficient for demonstrating the exercise, but it can be improved.

Current structure:

```bash
mysqldump ...
tar ...
rm ...
echo "wordpress backup was created ..."
```

If `mysqldump` fails, the script can potentially continue to the next commands.

A production-quality backup script should:

1. Detect command failures.
2. Stop when a critical operation fails.
3. Only log success after the complete backup succeeds.
4. Protect the database credentials.
5. Verify that the archive was created successfully.
6. Optionally keep multiple backup generations and remove old backups according to a retention policy.

These are improvements beyond the basic exercise.

---

# 26. Final Expected Result

At the end of the exercise, you should have:

```text
/usr/local/bin/backup.sh
```

containing the backup logic,

```text
/backup/wordpress_backup-YYYY-MM-DD.tar.gz
```

containing the SQL database dump,

```text
/var/log/backup.log
```

containing backup log entries,

and root's crontab containing:

```cron
0 2 * * * /usr/local/bin/backup.sh
```

The cron service should be:

```text
active
```

This completes the basic automated WordPress database backup exercise.
