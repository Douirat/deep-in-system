# MySQL on Ubuntu Server

## 1. Goal

The goal is to install a MySQL server on the Ubuntu Server VM and configure it for a later WordPress installation.

The intended architecture is:

```text
Ubuntu Server VM
│
├── WordPress
│      │
│      │ localhost
│      ▼
│   MySQL Server
│      │
│      ▼
│   wordpress database
│      │
│      ├── wp_users
│      ├── wp_posts
│      ├── wp_comments
│      └── ...
│
└── MySQL listens locally
    127.0.0.1:3306
```

The main security idea is:

> Keep MySQL local when remote database access is not required, and give WordPress its own database account with access limited to its own database.

---

# 2. Install MySQL Server

From the normal Ubuntu shell:

```bash
sudo apt install mysql-server
```

### What this does

Installs the MySQL server software on the VM.

`sudo` is required because installing system software modifies protected system directories.

MySQL is a **database server**. Applications communicate with it using SQL.

For example:

```text
Application
    │
    │ SQL query
    ▼
MySQL Server
    │
    ▼
Database
    │
    ▼
Tables
```

---

# 3. Run MySQL security configuration

After installation:

```bash
sudo mysql_secure_installation
```

This is a hardening script that can help configure basic MySQL security.

Depending on the MySQL version and configuration, it can address things such as:

* anonymous users
* remote root access
* test databases
* authentication configuration
* privilege reloading

It is a **security configuration tool**, not the MySQL server itself.

---

# 4. MySQL network binding

MySQL normally uses TCP port:

```text
3306
```

The configuration file discussed was:

```text
/etc/mysql/mysql.conf.d/mysqld.cnf
```

with:

```ini
bind-address = 127.0.0.1
```

## What is `bind-address`?

It tells the MySQL server which local network address/interface it should listen on.

`127.0.0.1` means **localhost**.

So:

```text
127.0.0.1:3306
```

means MySQL is listening on port `3306` through the local machine's loopback interface.

Conceptually:

```text
Same VM
   │
   │ 127.0.0.1
   ▼
MySQL :3306
```

Another machine trying to connect through the server's LAN address would not be able to use that MySQL listener if MySQL is bound only to `127.0.0.1`.

## Important terminology

Don't think:

> "I bind an IP address to the server."

More precisely:

> You configure the MySQL server process to bind/listen on `127.0.0.1`.

---

# 5. Why bind MySQL to localhost?

If WordPress and MySQL are running on the same VM, WordPress does not need MySQL to accept connections from the network.

Instead:

```text
WordPress
    │
    │ local connection
    ▼
127.0.0.1:3306
    │
    ▼
MySQL
```

This reduces MySQL's network exposure.

Without local-only binding, MySQL could potentially listen on another interface such as:

```text
192.168.1.6:3306
```

which would make network connections possible, subject to firewall and MySQL authentication rules.

---

# 6. Linux users vs MySQL users

This is an important distinction.

Linux has users such as:

```text
root
server
luffy
zoro
```

These are **Linux accounts**.

MySQL has its own accounts:

```text
root
wp_user
```

These are **MySQL accounts**.

They are managed by MySQL.

For example:

```bash
sudo mysql
```

is a Linux shell command used to start the MySQL client.

Once inside MySQL:

```text
mysql>
```

you execute SQL commands.

---

# 7. Enter the MySQL client

From Bash:

```bash
sudo mysql
```

The prompt changes from something like:

```text
server@server-host:~$
```

to:

```text
mysql>
```

This means you're now interacting with MySQL.

You saw:

```text
Welcome to the MySQL monitor.
```

and:

```text
Server version: 8.4.11-0ubuntu0.26.04.1 (Ubuntu)
```

This confirmed that the MySQL server was installed and that you successfully connected to it.

The:

```text
Your MySQL connection id is 8
```

is an identifier for your current MySQL connection/session.

It is not your user ID or database ID.

---

# 8. Bash commands vs SQL commands

There are two different environments.

## Ubuntu Bash

Prompt:

```text
server@server-host:~$
```

Commands:

```bash
sudo mysql
ls
pwd
systemctl status mysql
```

## MySQL client

Prompt:

```text
mysql>
```

Commands:

```sql
CREATE USER ...;
CREATE DATABASE ...;
GRANT ...;
SELECT ...;
SHOW DATABASES;
```

Mental model:

```text
Ubuntu Bash
server@server-host:~$
        │
        │ sudo mysql
        ▼
MySQL client
mysql>
        │
        ├── CREATE USER
        ├── CREATE DATABASE
        ├── GRANT
        └── SELECT
```

---

# 9. SQL statements need `;`

You encountered this several times.

You entered:

```sql
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'wordpress'
```

without:

```text
;
```

MySQL changed the prompt to:

```text
->
```

The `->` means:

> MySQL thinks the SQL statement is not finished yet.

You can finish it by entering:

```text
;
```

or cancel the incomplete statement with:

```text
Ctrl + C
```

You did both correctly during the session.

For example:

```sql
CREATE DATABASE wordpress;
```

The semicolon tells the MySQL client that the statement is complete.

---

# 10. Create a MySQL user

You successfully executed:

```sql
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'wordpress';
```

This creates the MySQL account:

```text
wp_user@localhost
```

The account has two important components:

```text
username + host
```

Therefore:

```text
'wp_user'@'localhost'
```

means:

```text
username = wp_user
host     = localhost
```

This is a MySQL account, not a Linux user.

---

# 11. MySQL account hosts

MySQL accounts can be associated with different hosts.

For example:

```text
'wp_user'@'localhost'
'wp_user'@'192.168.1.10'
'wp_user'@'%'
```

These can represent different MySQL accounts.

The important examples are:

```text
'wp_user'@'localhost'
```

means the account is associated with local connections.

And:

```text
'wp_user'@'%'
```

uses `%` as a wildcard for the host, meaning any host.

---

# 12. Create the WordPress database

The intended command is:

```sql
CREATE DATABASE wordpress;
```

A MySQL server can contain multiple databases:

```text
MySQL Server
│
├── wordpress
├── school
├── shop
└── ...
```

A database contains tables.

For example:

```text
wordpress
│
├── wp_users
├── wp_posts
├── wp_comments
└── wp_options
```

WordPress will later use the `wordpress` database.

---

# 13. Your typo

You initially attempted:

```sql
CREATE DATABASE wordpress
```

but forgot the semicolon and cancelled the statement with:

```text
Ctrl + C
```

That did not create the database.

Then you accidentally entered:

```sql
CREATE DATABASE wodpress;
```

Notice:

```text
wodpress
```

instead of:

```text
wordpress
```

So you created the wrong database name.

Your state became:

```text
Database:
wodpress
```

but your privilege command referred to:

```text
wordpress.*
```

These names do not match.

To correct it:

```sql
DROP DATABASE wodpress;
CREATE DATABASE wordpress;
```

Be careful with `DROP DATABASE`: it permanently deletes that database and its contents.

Since `wodpress` was just accidentally created and empty in this exercise, removing it is appropriate.

---

# 14. `GRANT`

The command:

```sql
GRANT ALL PRIVILEGES
ON wordpress.*
TO 'wp_user'@'localhost';
```

means:

> Give the MySQL account `wp_user` connecting from `localhost` broad privileges on everything inside the `wordpress` database.

Break it into pieces:

```text
GRANT
```

means:

```text
Give permissions
```

Then:

```text
ALL PRIVILEGES
```

means:

```text
Give broad/all available privileges
```

Then:

```text
ON wordpress.*
```

means:

```text
On everything inside the wordpress database
```

Finally:

```text
TO 'wp_user'@'localhost'
```

means:

```text
Give those permissions to wp_user@localhost
```

---

# 15. Understanding `wordpress.*`

This is one of the most important parts.

The general pattern is:

```text
database.object
```

So:

```text
wordpress.*
```

means:

```text
database = wordpress
object   = everything
```

For example:

```text
wordpress.*
    │
    ├── wp_users
    ├── wp_posts
    ├── wp_comments
    └── wp_options
```

It does NOT mean everything in the entire MySQL server.

Imagine:

```text
MySQL Server
│
├── wordpress
│   ├── wp_users
│   └── wp_posts
│
├── school
│   ├── students
│   └── teachers
│
└── shop
    ├── products
    └── orders
```

This:

```sql
GRANT ALL PRIVILEGES
ON wordpress.*
TO 'wp_user'@'localhost';
```

gives:

```text
wp_user
   │
   ▼
wordpress
   ├── wp_users       allowed
   └── wp_posts       allowed
```

but does not grant the same privileges on:

```text
school
shop
```

---

# 16. Why not give WordPress MySQL root?

You don't want your application to normally use:

```text
root
```

as its database account.

Instead:

```text
WordPress
    │
    ▼
wp_user
    │
    ▼
wordpress.*
```

This follows the:

> Principle of least privilege

The application gets the permissions it needs without unnecessarily giving it administrative access to every database.

---

# 17. `ALL PRIVILEGES` does not mean "everything"

Compare:

```sql
GRANT ALL PRIVILEGES
ON wordpress.*
TO 'wp_user'@'localhost';
```

with:

```sql
GRANT ALL PRIVILEGES
ON *.*
TO 'wp_user'@'localhost';
```

The first:

```text
wordpress.*
```

means:

```text
everything inside wordpress
```

The second:

```text
*.*
```

means:

```text
all databases
+
all objects
```

So the scope is determined by what comes after `ON`.

---

# 18. Examples of specific privileges

Instead of:

```sql
GRANT ALL PRIVILEGES
```

you could grant individual privileges.

For example:

```sql
GRANT SELECT
ON wordpress.*
TO 'wp_user'@'localhost';
```

would give read permission.

Other common privileges include:

```text
SELECT  → read data
INSERT  → add data
UPDATE  → modify data
DELETE  → delete data
CREATE  → create objects
ALTER   → modify object structures
DROP    → remove objects
```

For the WordPress setup in this exercise, the guide uses:

```sql
GRANT ALL PRIVILEGES
ON wordpress.*
TO 'wp_user'@'localhost';
```

---

# 19. `FLUSH PRIVILEGES`

You executed:

```sql
FLUSH PRIVILEGES;
```

This tells MySQL to reload privilege information.

With modern MySQL, commands such as:

```sql
CREATE USER
GRANT
```

already update the privilege system directly.

Therefore, after those commands, `FLUSH PRIVILEGES` is generally unnecessary.

You may still encounter it in tutorials, especially older ones.

---

# 20. Checking the root MySQL account

You executed:

```sql
SELECT host, user
FROM mysql.user
WHERE user='root';
```

This asks:

> Show the host and username for MySQL accounts whose username is `root`.

Your result was:

```text
+-----------+------+
| host      | user |
+-----------+------+
| localhost | root |
+-----------+------+
```

So you have:

```text
root@localhost
```

and the query did not show:

```text
root@%
```

---

# 21. What `%` means

In a MySQL account such as:

```text
'root'@'%'
```

the `%` represents a wildcard for the host.

Conceptually:

```text
root@localhost
      │
      └── local host

root@%
      │
      └── any host
```

The guide asks you to check that root doesn't have an any-host account such as:

```text
root@%
```

because remote root authentication increases the exposure of the administrative account.

---

# 22. Your current setup

Based on the commands you showed, you successfully created:

```text
wp_user@localhost
```

You also accidentally created:

```text
wodpress
```

instead of:

```text
wordpress
```

And you granted:

```text
wp_user@localhost
    │
    ▼
ALL PRIVILEGES
    │
    ▼
wordpress.*
```

Your root check showed:

```text
root@localhost
```

The intended final state should be:

```text
MySQL Server
│
├── root@localhost
│
└── wp_user@localhost
       │
       │ privileges
       ▼
   wordpress.*
       │
       ├── WordPress tables
       ├── wp_users
       ├── wp_posts
       ├── wp_comments
       └── ...
```

---

# 23. Commands to finish/correct the setup

First check the databases:

```sql
SHOW DATABASES;
```

If you see the accidental:

```text
wodpress
```

you can remove it:

```sql
DROP DATABASE wodpress;
```

Then create the correctly named database:

```sql
CREATE DATABASE wordpress;
```

Then ensure the WordPress user has privileges:

```sql
GRANT ALL PRIVILEGES
ON wordpress.*
TO 'wp_user'@'localhost';
```

Check the privileges:

```sql
SHOW GRANTS FOR 'wp_user'@'localhost';
```

Check the database:

```sql
SHOW DATABASES;
```

Check the root account:

```sql
SELECT host, user
FROM mysql.user
WHERE user='root';
```

---

# 24. The complete mental model

The most important thing is to understand that there are several different layers.

```text
Ubuntu Server
│
│
├── Linux users
│      ├── root
│      ├── server
│      ├── luffy
│      └── zoro
│
│
└── MySQL Server
       │
       ├── MySQL accounts
       │      ├── root@localhost
       │      └── wp_user@localhost
       │
       └── Databases
              ├── wordpress
              │      ├── tables
              │      ├── rows
              │      └── ...
              │
              └── other databases
```

And the WordPress relationship is:

```text
WordPress
    │
    │ uses credentials
    ▼
wp_user@localhost
    │
    │ has privileges
    ▼
wordpress.*
    │
    ▼
WordPress data
```

Meanwhile, the network configuration determines whether MySQL can be reached from outside:

```text
bind-address = 127.0.0.1
                 │
                 ▼
          MySQL listens locally
```

So there are **two different security concepts**:

### Network access

```text
bind-address = 127.0.0.1
```

Controls:

> Where can the MySQL server listen for network connections?

### Database authorization

```sql
'wp_user'@'localhost'
```

and:

```sql
GRANT ALL PRIVILEGES ON wordpress.*
```

Control:

> Which MySQL account can connect, from where, and what can that account do?

These are related, but they are not the same thing.
