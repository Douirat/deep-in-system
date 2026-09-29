# 9. WordPress

WordPress is a PHP web application that uses a database, normally MySQL, to store website data.

The basic architecture is:

```text
Browser
   |
   | HTTP
   v
Apache
   |
   | executes PHP
   v
PHP
   |
   v
WordPress
   |
   | SQL
   v
MySQL
   |
   v
wordpress database
```

---

# 9.1 Install the Required Packages

```bash
sudo apt install apache2 php php-mysql libapache2-mod-php php-curl php-gd php-xml php-mbstring
```

This installs several components.

## `sudo`

```bash
sudo
```

Runs the command with administrator privileges.

Installing system packages modifies protected system directories, so root privileges are required.

---

## `apt`

```bash
apt
```

Ubuntu's package manager.

For example:

```bash
sudo apt install apache2
```

means:

> Download and install the Apache package and its dependencies.

---

## `apache2`

```bash
apache2
```

Installs the Apache HTTP server.

Apache receives HTTP requests from browsers.

For example:

```text
Browser
   |
   | GET /
   v
Apache
```

On Ubuntu, Apache's default web root is:

```text
/var/www/html
```

Therefore, files placed there can be served by Apache.

---

## `php`

```bash
php
```

Installs PHP.

WordPress is primarily written in PHP.

PHP is responsible for executing WordPress's server-side code.

For example:

```php
<?php

echo "Hello";
```

PHP executes the code and produces output.

---

## `libapache2-mod-php`

```bash
libapache2-mod-php
```

Connects Apache with PHP.

Apache itself does not understand PHP code.

Conceptually:

```text
Browser
   |
   v
Apache
   |
   | PHP file
   v
PHP
   |
   | generated HTML
   v
Apache
   |
   v
Browser
```

---

## `php-mysql`

```bash
php-mysql
```

Allows PHP applications to communicate with MySQL.

WordPress needs this because WordPress stores its data in MySQL.

```text
WordPress
    |
    | PHP/MySQL connection
    v
MySQL
```

---

## `php-curl`

```bash
php-curl
```

Provides networking functionality to PHP.

PHP applications can use cURL to communicate with external HTTP services and APIs.

---

## `php-gd`

```bash
php-gd
```

Provides image-processing functionality.

WordPress can use this when processing uploaded images and generating image sizes.

For example:

```text
original image
      |
      v
WordPress/PHP
      |
      +----> thumbnail
      +----> medium
      +----> large
```

---

## `php-xml`

```bash
php-xml
```

Provides XML-related functionality to PHP.

WordPress and its plugins can use XML functionality for feeds, integrations, APIs, and other tasks.

---

## `php-mbstring`

```bash
php-mbstring
```

Provides multibyte string support.

This is important when working with text containing characters outside basic ASCII.

Examples:

```text
é
ع
中文
日本語
```

---

# 9.2 Download WordPress

```bash
cd /tmp && wget https://wordpress.org/latest.tar.gz
```

This is actually two commands joined with `&&`.

```bash
cd /tmp
```

changes the current directory to `/tmp`.

Then:

```bash
wget https://wordpress.org/latest.tar.gz
```

downloads the WordPress archive.

The `&&` means:

> Run the second command only if the first command succeeds.

The downloaded file will be:

```text
/tmp/latest.tar.gz
```

---

# 9.3 Why `/tmp`?

`/tmp` is intended for temporary files.

We download and extract WordPress there before moving it to Apache's web directory.

The process is:

```text
/tmp
 |
 +-- latest.tar.gz
 |
 +-- wordpress/
```

Then WordPress is moved to:

```text
/var/www/html
```

---

# 9.4 What is `wget`?

```bash
wget
```

is a command-line program for downloading files.

For example:

```bash
wget https://example.com/file.zip
```

downloads the file from the specified URL.

In this case:

```bash
wget https://wordpress.org/latest.tar.gz
```

downloads WordPress.

---

# 9.5 What is `latest.tar.gz`?

It is a compressed archive.

There are two parts:

```text
.tar
```

and:

```text
.gz
```

`tar` groups files into an archive.

`gzip` compresses the archive.

Conceptually:

```text
WordPress files
      |
      v
     .tar
      |
      v
   gzip compression
      |
      v
latest.tar.gz
```

It is similar in purpose to a `.zip` file.

---

# 9.6 Extract WordPress

```bash
tar -xzf latest.tar.gz
```

Breakdown:

```text
tar
```

Archive utility.

```text
-x
```

Extract.

```text
-z
```

Use gzip compression.

```text
-f
```

The archive filename follows.

Therefore:

```bash
tar -xzf latest.tar.gz
```

means:

> Extract `latest.tar.gz`.

After extraction:

```text
/tmp/
├── latest.tar.gz
└── wordpress/
    ├── index.php
    ├── wp-admin/
    ├── wp-content/
    ├── wp-includes/
    ├── wp-config-sample.php
    └── ...
```

---

# 9.7 WordPress Directory Structure

The extracted WordPress directory contains several important parts.

```text
wordpress/
├── wp-admin/
├── wp-content/
├── wp-includes/
├── index.php
├── wp-config-sample.php
└── ...
```

## `wp-admin`

Contains WordPress administration functionality.

The WordPress administration interface is accessed through:

```text
/wp-admin/
```

For example:

```text
http://YOUR_SERVER_IP/wp-admin/
```

---

## `wp-content`

Contains site-specific content.

Important directories include:

```text
wp-content/
├── plugins/
├── themes/
└── uploads/
```

### `plugins`

Contains installed WordPress plugins.

### `themes`

Contains installed themes.

### `uploads`

Contains uploaded media such as images.

---

## `wp-includes`

Contains WordPress core libraries and functionality.

Normally, these files should not be manually modified.

---

## `index.php`

An important entry point into WordPress.

Conceptually:

```text
Browser
   |
   v
index.php
   |
   v
WordPress
```

---

# 9.8 Move WordPress to Apache's Web Root

```bash
sudo mv wordpress/* /var/www/html/
```

This moves the contents of:

```text
wordpress/
```

into:

```text
/var/www/html/
```

Apache's default document root is:

```text
/var/www/html
```

Therefore, after the command:

```text
/var/www/html/
├── index.php
├── wp-admin/
├── wp-content/
├── wp-includes/
└── ...
```

Now Apache can serve WordPress.

---

# 9.9 Why `wordpress/*`?

There is an important difference between:

```bash
sudo mv wordpress/* /var/www/html/
```

and:

```bash
sudo mv wordpress /var/www/html/
```

The first moves the contents:

```text
wordpress/
   |
   +-- index.php
   +-- wp-admin/
   +-- wp-content/
   +-- wp-includes/

          |
          | move contents
          v

/var/www/html/
```

The second could result in:

```text
/var/www/html/wordpress/
```

which would mean accessing WordPress through:

```text
http://YOUR_SERVER_IP/wordpress/
```

The assignment wants WordPress at the root:

```text
http://YOUR_SERVER_IP/
```

so the contents are moved into `/var/www/html`.

---

# 9.10 Change Ownership

```bash
sudo chown -R www-data:www-data /var/www/html
```

`chown` means:

> Change ownership.

The syntax is:

```text
chown owner:group file
```

Therefore:

```text
www-data:www-data
```

means:

```text
owner = www-data
group = www-data
```

---

# 9.11 What is `www-data`?

Apache commonly runs under the Linux user:

```text
www-data
```

Conceptually:

```text
Apache
   |
   | runs as
   v
www-data
```

Linux file permissions apply to this user.

If WordPress/PHP needs to write files, such as uploaded images, the Linux permissions must allow it.

---

# 9.12 What does `-R` mean?

```text
-R
```

means:

> Recursive.

Without `-R`, ownership would only be changed on the specified directory.

With:

```bash
sudo chown -R www-data:www-data /var/www/html
```

ownership is changed throughout the directory tree.

For example:

```text
/var/www/html/
├── index.php
├── wp-admin/
├── wp-content/
│   ├── plugins/
│   ├── themes/
│   └── uploads/
└── wp-includes/
```

All of these are affected.

---

# 9.13 Why Does Ownership Matter?

Suppose:

```text
/var/www/html/wp-content/uploads
```

belongs to:

```text
root:root
```

while Apache/PHP runs as:

```text
www-data
```

PHP may not be allowed to write there.

That can cause errors when WordPress tries to upload files.

Giving the web server ownership allows WordPress to work with its files.

For a learning environment, this command is common:

```bash
sudo chown -R www-data:www-data /var/www/html
```

In a production environment, permissions are often made more restrictive and carefully scoped.

---

# 9.14 Configure `wp-config.php`

WordPress needs to know how to connect to MySQL.

The important configuration values are:

```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'wp_user');
define('DB_PASSWORD', 'your_password');
define('DB_HOST', 'localhost');
```

These values tell WordPress:

```text
Database name
MySQL username
MySQL password
MySQL server
```

---

# 9.15 `DB_NAME`

```php
define('DB_NAME', 'wordpress');
```

This tells WordPress:

> Use the MySQL database called `wordpress`.

You can verify the database in MySQL:

```sql
SHOW DATABASES;
```

You might see:

```text
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| wordpress          |
+--------------------+
```

---

# 9.16 `DB_USER`

```php
define('DB_USER', 'wp_user');
```

This tells WordPress:

> Connect to MySQL using the MySQL account `wp_user`.

This is a MySQL user, not a Linux user.

Linux users might be:

```text
server
luffy
zoro
www-data
```

MySQL users might be:

```text
root
wp_user
login_user
```

They belong to different authentication systems.

---

# 9.17 Why Should WordPress Not Use MySQL `root`?

Do not configure WordPress like this:

```php
define('DB_USER', 'root');
```

Instead, create a dedicated MySQL account.

For example:

```sql
CREATE USER 'wp_user'@'localhost'
IDENTIFIED BY 'strong_password';

GRANT ALL PRIVILEGES
ON wordpress.*
TO 'wp_user'@'localhost';
```

Now WordPress uses:

```text
WordPress
    |
    | wp_user
    v
wordpress database
```

rather than:

```text
WordPress
    |
    | root
    v
Everything MySQL root can access
```

The principle is:

> An application should use a dedicated account with only the privileges it needs.

---

# 9.18 `DB_PASSWORD`

```php
define('DB_PASSWORD', 'your_password');
```

This is the password belonging to:

```text
wp_user
```

It is not:

* your Linux password
* your SSH password
* your WordPress administrator password

These are separate credentials.

Conceptually:

```text
Linux
    |
    +-- server password

MySQL
    |
    +-- wp_user password

WordPress
    |
    +-- administrator password
```

---

# 9.19 `DB_HOST`

```php
define('DB_HOST', 'localhost');
```

`localhost` means:

> The MySQL server is running on this same machine.

So:

```text
WordPress
    |
    | localhost
    v
MySQL
```

This is appropriate when Apache, PHP, WordPress, and MySQL are all installed on the same server.

---

# 9.20 WordPress Connecting to MySQL

The complete flow becomes:

```text
Browser
   |
   v
Apache
   |
   v
PHP
   |
   v
WordPress
   |
   | DB_HOST = localhost
   | DB_NAME = wordpress
   | DB_USER = wp_user
   | DB_PASSWORD = ********
   v
MySQL
   |
   v
wordpress database
```

WordPress can now read and write website data.

---

# 9.21 Protect `wp-config.php`

`wp-config.php` contains sensitive configuration information.

Therefore, Apache should not allow visitors to request it directly.

Add this to the appropriate Apache virtual host:

```apache
<Files wp-config.php>
    Require all denied
</Files>
```

---

# 9.22 What Does `<Files>` Mean?

```apache
<Files wp-config.php>
```

means:

> Apply this Apache rule to the file named `wp-config.php`.

Then:

```apache
Require all denied
```

means:

> Deny access to everyone.

So:

```text
Browser
   |
   | GET /wp-config.php
   v
Apache
   |
   | access denied
   v
403 Forbidden
```

The visitor should not be able to retrieve the configuration file.

---

# 9.23 Why Protect `wp-config.php`?

The file may contain:

```php
define('DB_USER', 'wp_user');
define('DB_PASSWORD', 'secret');
```

These are database credentials.

If somebody could retrieve the file as plain text, they could obtain those credentials.

Protecting it is therefore a security measure.

---

# 9.24 Apache Virtual Hosts

Apache can host multiple websites.

A virtual host contains configuration for a particular website.

Typical configuration locations are:

```text
/etc/apache2/sites-available/
```

and:

```text
/etc/apache2/sites-enabled/
```

For example:

```text
/etc/apache2/sites-available/000-default.conf
```

may contain:

```apache
<VirtualHost *:80>

    DocumentRoot /var/www/html

    <Files wp-config.php>
        Require all denied
    </Files>

</VirtualHost>
```

The important part for this assignment is:

```apache
<Files wp-config.php>
    Require all denied
</Files>
```

---

# 9.25 Test Apache Configuration

Before reloading Apache, test the configuration:

```bash
sudo apache2ctl configtest
```

A successful configuration should produce:

```text
Syntax OK
```

If Apache reports an error, fix the configuration before reloading it.

---

# 9.26 Reload Apache

After changing Apache configuration:

```bash
sudo systemctl reload apache2
```

Reload means:

> Re-read the configuration without completely stopping Apache.

Compared with:

```bash
sudo systemctl restart apache2
```

a reload is less disruptive.

For configuration changes, use:

```bash
sudo systemctl reload apache2
```

when possible.

---

# 9.27 Complete the WordPress Installation

Open the server's address in a browser:

```text
http://YOUR_SERVER_IP/
```

For example:

```text
http://192.168.1.33/
```

The exact IP depends on your server's current network configuration.

The request follows:

```text
Browser
   |
   | GET /
   v
Apache
   |
   v
/var/www/html/index.php
   |
   v
PHP
   |
   v
WordPress
   |
   v
MySQL
```

If everything is configured correctly, WordPress will show its setup page.

---

# 9.28 Create the WordPress Administrator

During installation, WordPress asks you to create an administrator account.

This account belongs to WordPress.

It is not a Linux account.

It is also not a MySQL account.

For example:

```text
Linux user:
server

MySQL user:
wp_user

WordPress user:
admin
```

These are three separate systems.

---

# 9.29 WordPress Administrator vs MySQL User

This distinction is important.

The MySQL user:

```text
wp_user
```

allows WordPress to communicate with MySQL.

The WordPress administrator:

```text
admin
```

allows a human to manage the WordPress website.

Conceptually:

```text
Human
   |
   | WordPress username/password
   v
WordPress
   |
   | MySQL username/password
   v
MySQL
```

---

# 9.30 Create a Test Post

After logging into WordPress, create a test post.

For example:

```text
Title:
My First Test Post

Content:
WordPress installation is working.
```

Publish it.

This is more than just a visual test.

Creating a post tests several layers of the system.

---

# 9.31 What Happens When You Create a Post?

Conceptually:

```text
Browser
   |
   | HTTP request
   v
Apache
   |
   v
PHP
   |
   v
WordPress
   |
   | SQL
   v
MySQL
   |
   v
wordpress database
```

WordPress stores information about the post in its database.

When you later open the post:

```text
Browser
   |
   v
Apache
   |
   v
PHP
   |
   v
WordPress
   |
   | SQL SELECT
   v
MySQL
   |
   v
Post data
```

WordPress uses the retrieved data to generate the page.

---

# 9.32 Why a Test Post Is Useful

A successful test post demonstrates that multiple components are working together:

```text
Apache
   ✓
PHP
   ✓
WordPress
   ✓
MySQL connection
   ✓
Database permissions
   ✓
WordPress database
   ✓
WordPress administrator
   ✓
```

So the test post is effectively an end-to-end test.

---

# 9.33 Complete Installation

The complete process is:

## Step 1 — Install Apache, PHP, and PHP extensions

```bash
sudo apt install apache2 php php-mysql libapache2-mod-php php-curl php-gd php-xml php-mbstring
```

---

## Step 2 — Download WordPress

```bash
cd /tmp && wget https://wordpress.org/latest.tar.gz
```

---

## Step 3 — Extract WordPress

```bash
tar -xzf latest.tar.gz
```

---

## Step 4 — Move WordPress into Apache's web root

```bash
sudo mv wordpress/* /var/www/html/
```

---

## Step 5 — Give Apache ownership

```bash
sudo chown -R www-data:www-data /var/www/html
```

---

## Step 6 — Configure `wp-config.php`

Example:

```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'wp_user');
define('DB_PASSWORD', 'your_password');
define('DB_HOST', 'localhost');
```

Do not use MySQL `root`.

---

## Step 7 — Protect `wp-config.php`

In the Apache virtual host:

```apache
<Files wp-config.php>
    Require all denied
</Files>
```

---

## Step 8 — Test Apache configuration

```bash
sudo apache2ctl configtest
```

Expected:

```text
Syntax OK
```

---

## Step 9 — Reload Apache

```bash
sudo systemctl reload apache2
```

---

## Step 10 — Open WordPress

```text
http://YOUR_SERVER_IP/
```

---

## Step 11 — Create the WordPress administrator

Complete the setup through the browser.

---

## Step 12 — Create a test post

Publish a test post and verify that it can be viewed.

---

# 9.34 Final Architecture

After everything is installed, the server looks conceptually like this:

```text
                         CLIENT
                           |
                           | HTTP :80
                           v
                  +-------------------+
                  |      Apache       |
                  |     www-data      |
                  +-------------------+
                           |
                           | executes PHP
                           v
                  +-------------------+
                  |       PHP         |
                  +-------------------+
                           |
                           v
                  +-------------------+
                  |     WordPress     |
                  | /var/www/html     |
                  +-------------------+
                           |
                           | MySQL connection
                           | wp_user
                           v
                  +-------------------+
                  |       MySQL       |
                  +-------------------+
                           |
                           v
                  +-------------------+
                  |  wordpress DB     |
                  |                   |
                  | users             |
                  | posts             |
                  | pages             |
                  | settings          |
                  | comments          |
                  | metadata          |
                  +-------------------+
```

The key idea is:

```text
Apache
  ↓
receives HTTP requests

PHP
  ↓
executes server-side PHP code

WordPress
  ↓
provides the website/application logic

MySQL
  ↓
stores the website's persistent data
```

WordPress is therefore not simply "a collection of HTML files."

It is a PHP application that runs on the server, communicates with MySQL, generates web pages dynamically, and uses Apache to communicate with the browser.
