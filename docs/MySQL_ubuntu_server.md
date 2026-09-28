# MySQL Server Security and WordPress Database Setup

## 1. Installing MySQL

Install the MySQL server:

```bash
sudo apt install mysql-server
```

After installation, run the security configuration:

```bash
sudo mysql_secure_installation
```

This removes or disables unnecessary/default MySQL features that can create security risks.

---

# 2. `mysql_secure_installation`

Run:

```bash
sudo mysql_secure_installation
```

Recommended choices for a typical Ubuntu server:

| Question                      | Recommendation | Reason                                      |
| ----------------------------- | -------------- | ------------------------------------------- |
| Validate password component?  | `Y`            | Enforces stronger passwords                 |
| Password validation policy    | `2` / MEDIUM   | Good balance between security and usability |
| Remove anonymous users?       | `Y`            | Anonymous accounts are unnecessary          |
| Disallow root login remotely? | `Y`            | Prevents remote MySQL root authentication   |
| Remove test database?         | `Y`            | Removes an unnecessary database             |
| Reload privilege tables?      | `Y`            | Applies the changes immediately             |

## Important: MySQL root authentication

On Ubuntu installations, MySQL `root` may authenticate using the Unix socket rather than a MySQL password.

You can test local root access with:

```bash
sudo mysql
```

If you see:

```text
mysql>
```

you successfully entered MySQL as root.

You can leave MySQL with:

```sql
exit;
```

---

# 3. MySQL Network Binding

MySQL normally listens for TCP connections on port:

```text
3306
```

The configuration file is:

```text
/etc/mysql/mysql.conf.d/mysqld.cnf
```

To edit it:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Set:

```ini
bind-address = 127.0.0.1
```

---

# 4. What Does `127.0.0.1` Mean?

`127.0.0.1` is the server's **loopback address**.

It means:

> This computer itself.

So:

```ini
bind-address = 127.0.0.1
```

tells MySQL:

> Only accept network connections through the server's local loopback interface.

Conceptually:

```text
                 Ubuntu Server
              ┌─────────────────┐
              │                 │
              │     MySQL       │
              │      :3306      │
              │        ▲        │
              │        │        │
              │   127.0.0.1     │
              └────────┼────────┘
                       │
                Local programs
```

For example:

```text
WordPress
    │
    │ 127.0.0.1:3306
    ▼
  MySQL
```

But another computer cannot directly connect through the server's network interface:

```text
Other computer
      │
      │ 192.168.1.33:3306
      X
    MySQL
```

assuming MySQL is bound only to `127.0.0.1`.

---

# 5. Why Bind MySQL to Localhost?

If WordPress and MySQL are on the same server, WordPress does not need MySQL to be accessible from other computers.

For example:

```text
Internet / LAN
      │
      X
      │
   MySQL :3306
      │
      │ localhost only
      ▼
WordPress
```

This reduces the number of ways an external machine can interact with MySQL.

The principle is:

> If a service does not need remote network access, don't expose it unnecessarily.

---

# 6. MySQL Users Are Username + Host

One of the most important MySQL concepts is that a MySQL account is not simply a username.

It is effectively:

```text
username + host
```

For example:

```sql
'wp_user'@'localhost'
```

has two components:

```text
'wp_user'     '@'     'localhost'
    │                    │
    │                    └── Where the user may connect from
    │
    └─────────────────────── Username
```

Therefore:

```sql
'wp_user'@'localhost'
```

means:

> The MySQL user `wp_user` can authenticate when connecting from the local machine.

---

# 7. Different Hosts Create Different MySQL Accounts

These are different MySQL accounts:

```text
'wp_user'@'localhost'
'wp_user'@'192.168.1.10'
'wp_user'@'%'
```

Even though they all use:

```text
wp_user
```

as the username, their allowed connection sources are different.

For example:

```text
'wp_user'@'localhost'
```

means:

```text
Only the local server
```

while:

```text
'wp_user'@'%'
```

means:

```text
Any host
```

---

# 8. What Does `%` Mean?

In a MySQL host specification:

```text
%
```

is a wildcard.

It means approximately:

> Any host.

Therefore:

```text
'root'@'%'
```

means:

> A MySQL `root` account that can authenticate from any host, subject to the server's other network and authentication controls.

This is why checking for:

```text
root@%
```

is useful.

---

# 9. Remote Root Login

You generally do not want MySQL root authentication available remotely.

The recommended configuration is:

```text
'root'@'localhost'
```

rather than:

```text
'root'@'%'
```

With:

```text
'root'@'localhost'
```

root administration is restricted to the local server.

Conceptually:

```text
Ubuntu Server
      │
      ▼
'root'@'localhost'
      │
      ▼
    MySQL
```

Instead of:

```text
Remote computer
      │
      ▼
'root'@'%'
      │
      ▼
    MySQL
```

The second arrangement unnecessarily exposes a highly privileged account to remote authentication attempts.

---

# 10. Check How Root Is Configured

Inside MySQL, run:

```sql
SELECT host, user
FROM mysql.user
WHERE user='root';
```

You may see:

```text
+-----------+------+
| host      | user |
+-----------+------+
| localhost | root |
+-----------+------+
```

The important part is:

```text
localhost
```

You are checking that you don't have an unnecessary:

```text
%
```

entry for root.

For example, this would indicate a root account associated with any host:

```text
+------+------+
| host | user |
+------+------+
| %    | root |
+------+------+
```

---

# 11. Why Create a Separate WordPress User?

Do **not** normally configure WordPress to use MySQL `root`.

Instead, create a dedicated MySQL account:

```text
wp_user
```

The idea is **least privilege**.

Instead of:

```text
WordPress
    │
    ▼
MySQL root
    │
    ▼
Everything
```

use:

```text
WordPress
    │
    ▼
wp_user
    │
    ▼
wordpress database
```

The WordPress application only needs access to its own database.

---

# 12. Create the WordPress Database

Inside MySQL:

```sql
CREATE DATABASE wordpress;
```

This creates a database named:

```text
wordpress
```

Conceptually:

```text
MySQL
  │
  └── wordpress
```

---

# 13. Create the WordPress User

Create the account:

```sql
CREATE USER 'wp_user'@'localhost'
IDENTIFIED BY 'your_password';
```

This creates:

```text
username:
    wp_user

allowed host:
    localhost

password:
    your_password
```

The important part is:

```sql
'wp_user'@'localhost'
```

which restricts the account to local connections.

Use a strong password instead of the example password.

---

# 14. Grant Permissions

Now give the WordPress user permissions on its database:

```sql
GRANT ALL PRIVILEGES ON wordpress.*
TO 'wp_user'@'localhost';
```

The important part is:

```text
wordpress.*
```

Break it down:

```text
wordpress.*
│         │
│         └── all objects/tables in the database
│
└──────────── database
```

So this means:

> Give `wp_user` all privileges on objects within the `wordpress` database.

It does **not** mean:

> Give `wp_user` unrestricted control over every database on the MySQL server.

---

# 15. Apply the Privilege Changes

Run:

```sql
FLUSH PRIVILEGES;
```

This tells MySQL to reload its privilege information.

Then the account and permissions are ready to use.

---

# 16. Complete WordPress Database Setup

The complete sequence is:

```sql
CREATE DATABASE wordpress;

CREATE USER 'wp_user'@'localhost'
IDENTIFIED BY 'your_password';

GRANT ALL PRIVILEGES ON wordpress.*
TO 'wp_user'@'localhost';

FLUSH PRIVILEGES;
```

You can then verify the user:

```sql
SELECT user, host
FROM mysql.user
WHERE user='wp_user';
```

Expected result:

```text
wp_user | localhost
```

---

# 17. The Security Model

Your setup should look approximately like this:

```text
                     Ubuntu Server
              ┌────────────────────────┐
              │                        │
              │       WordPress        │
              │           │            │
              │           │            │
              │           ▼            │
              │     127.0.0.1:3306     │
              │           │            │
              │           ▼            │
              │         MySQL          │
              │                        │
              │     ┌────────────┐     │
              │     │ wordpress  │     │
              │     │ database   │     │
              │     └─────▲──────┘     │
              │           │            │
              │        wp_user         │
              │      @localhost        │
              │                        │
              └────────────────────────┘
```

External machines:

```text
Other computer
      │
      │
      X
      │
 MySQL :3306
```

because MySQL is configured to listen only on:

```text
127.0.0.1
```

---

# 18. Multiple Layers of Protection

Your configuration provides several separate protections.

## Layer 1 — Network binding

```ini
bind-address = 127.0.0.1
```

MySQL isn't listening for normal network connections on the server's external IP.

---

## Layer 2 — Root restriction

```text
'root'@'localhost'
```

Root is not intended to be remotely authenticated.

---

## Layer 3 — Dedicated application account

```text
'wp_user'@'localhost'
```

WordPress doesn't need to use root.

---

## Layer 4 — Database-level permissions

```sql
GRANT ALL PRIVILEGES ON wordpress.*
TO 'wp_user'@'localhost';
```

The application account's permissions are scoped to the WordPress database.

---

# 19. Why Not Give WordPress Root?

Imagine MySQL contains:

```text
MySQL
├── wordpress
├── another_database
├── mysql
├── sys
└── other databases
```

If WordPress uses:

```text
root
```

then a compromise of WordPress could potentially give an attacker access to MySQL administrative capabilities.

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

The application is given only the database it needs.

This follows the security principle:

> **Least privilege:** give a program only the permissions it actually needs.

---

# 20. Useful Verification Commands

## Check MySQL service

From the Linux shell:

```bash
sudo systemctl status mysql
```

---

## Check MySQL listening ports

```bash
sudo ss -lntp | grep 3306
```

If MySQL is bound to localhost, you should see something similar to:

```text
127.0.0.1:3306
```

rather than:

```text
0.0.0.0:3306
```

or:

```text
*:3306
```

---

## Check MySQL users

Inside MySQL:

```sql
SELECT user, host
FROM mysql.user;
```

---

## Check root specifically

```sql
SELECT host, user
FROM mysql.user
WHERE user='root';
```

---

## Check the WordPress user

```sql
SELECT host, user
FROM mysql.user
WHERE user='wp_user';
```

---

# 21. The Most Important Concepts

Remember these four ideas:

### 1. `127.0.0.1`

Means:

```text
This machine itself
```

---

### 2. `localhost`

In a MySQL account:

```sql
'wp_user'@'localhost'
```

means:

```text
wp_user can authenticate from the local host.
```

---

### 3. `%`

Means:

```text
Any host
```

So:

```sql
'root'@'%'
```

represents a root account associated with any host.

---

### 4. `wordpress.*`

Means:

```text
All objects/tables inside the wordpress database
```

So:

```sql
GRANT ALL PRIVILEGES ON wordpress.*
TO 'wp_user'@'localhost';
```

means:

```text
wp_user
   │
   └── localhost only
          │
          └── all privileges
                  │
                  └── wordpress database only
```

---

# 22. Recommended Final Configuration

For your Ubuntu server, the intended configuration is:

```ini
# /etc/mysql/mysql.conf.d/mysqld.cnf

bind-address = 127.0.0.1
```

And in MySQL:

```sql
CREATE DATABASE wordpress;

CREATE USER 'wp_user'@'localhost'
IDENTIFIED BY 'STRONG_PASSWORD';

GRANT ALL PRIVILEGES ON wordpress.*
TO 'wp_user'@'localhost';

FLUSH PRIVILEGES;
```

Then verify:

```sql
SELECT host, user
FROM mysql.user
WHERE user='root';
```

and:

```sql
SELECT host, user
FROM mysql.user
WHERE user='wp_user';
```

The basic architecture is:

```text
                    SERVER
                       │
              ┌────────┴────────┐
              │                 │
          WordPress          MySQL
              │                 │
              └──── localhost ──┘
                                │
                         wp_user@localhost
                                │
                                ▼
                         wordpress.*
```

The overall goal is simple:

> **MySQL is local, root is local, and WordPress gets its own dedicated account with access to its own database.**
