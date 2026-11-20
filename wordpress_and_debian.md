> [!NOTE]
> **🇬🇧 Note on the use of AI:** AI tools such as ChatGPT, Gemini, and Copilot were used to help develop this material.  
> **🇫🇮 Huomio tekoälyn käytöstä:** Tämän materiaalin laatimisessa on hyödynnetty tekoälytyökaluja, kuten ChatGPT:tä, Geminiä ja Copilotia.  

# **Installing WordPress on Debian Server with MariaDB**

## **🎯 Objective**

Install and configure a working WordPress website on Debian Linux using the following components:

- **Debian Linux** – Operating system
- **Nginx** – Web server
- **PHP-FPM** – Executes WordPress PHP code
- **MariaDB** – Stores WordPress data
- **WordPress** – Content management system (CMS)

The WordPress source code archive (`wordpress.tar.gz`) has already been downloaded to your home directory.

After completing this guide, you should be able to:

- Install and manage the required services on Debian.
- Create a MariaDB database and a dedicated database user.
- Extract and configure WordPress.
- Configure Nginx and PHP-FPM to serve WordPress.
- Access and manage WordPress through a web browser.
- Verify the installation and troubleshoot common problems.

## **System architecture**

**Web browser**

http\://SERVER_IP

**Nginx**

Web server · TCP port 80

**PHP-FPM + WordPress**

PHP application processing

**MariaDB**

Local database · TCP port 3306

## **1. Update Debian and install packages**

Update the package index and installed packages:

```
sudo apt update
sudo apt upgrade -y
```

Install Nginx, MariaDB, PHP-FPM, and the PHP extensions required by WordPress:

```
sudo apt install -y \
  nginx mariadb-server mariadb-client \
  php php-fpm php-mysql php-curl \
  php-gd php-mbstring php-xml \
  php-zip php-intl
```

The exact PHP package names may vary between Debian releases. Check the installed PHP version:

```
php -v
```

## **2. Configure MariaDB**

Start MariaDB and enable automatic startup:

```
sudo systemctl enable --now mariadb
```

Verify that MariaDB is running:

```
sudo systemctl status mariadb
```

Run the initial security configuration:

```
sudo mariadb-secure-installation
```

Follow the prompts to remove anonymous users, disable remote root login, and remove the test database. On Debian, MariaDB root commonly uses Unix socket authentication, so a separate database root password may not be necessary.

### **Create the WordPress database**

Open the MariaDB command-line interface:

```
sudo mariadb
```

Execute the following SQL commands:

```
CREATE DATABASE wordpress
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'wpuser'@'localhost'
  IDENTIFIED BY 'ChangeThisStrongPassword!';

GRANT ALL PRIVILEGES ON wordpress.*
  TO 'wpuser'@'localhost';

EXIT;
```

Replace the example password with a strong, unique password.

Test the database connection:

```
mariadb -u wpuser -p wordpress
```

Enter the password and verify that the connection succeeds. You can also check the databases:

```
SHOW DATABASES;
EXIT;
```

## **3. Extract and install WordPress**

The WordPress archive is already located in your home directory.

Check the file:

```
ls -lh ~/wordpress*.tar.gz
```

Extract the archive:

```
tar -xzvf ~/wordpress*.tar.gz
```

Verify the extracted directory:

```
ls -l ~/wordpress
```

Create a directory for the website:

```
sudo mkdir -p /var/www/wordpress
```

Copy the WordPress files:

```
sudo cp -a ~/wordpress/. /var/www/wordpress/
```

Set the correct ownership and permissions:

```
sudo chown -R www-data:www-data /var/www/wordpress

sudo find /var/www/wordpress -type d -exec chmod 755 {} \;

sudo find /var/www/wordpress -type f -exec chmod 644 {} \;
```

On Debian, Nginx and PHP-FPM commonly use the `www-data` account, whereas Alpine commonly uses `nginx`.

The Nginx user will now have access to the WordPress files.

## **4. Configure PHP-FPM**

Find the installed PHP-FPM configuration:

```
ls /etc/php/*/fpm/pool.d/www.conf
```

The default Debian PHP-FPM configuration usually uses a Unix socket instead of a TCP port.

For example:

```
/run/php/php8.4-fpm.sock
```

Check the actual socket configuration:

```
grep '^listen =' /etc/php/*/fpm/pool.d/www.conf
```

Example output:

```
listen = /run/php/php8.4-fpm.sock
```

We will use this Unix socket in the Nginx configuration.

Start PHP-FPM and enable it at boot:

```
sudo systemctl enable --now phpPHP_VERSION-fpm
```

For example, with PHP 8.4:

```
sudo systemctl enable --now php8.4-fpm
```

Verify the service:

```
sudo systemctl status php8.4-fpm
```

Use your actual PHP version in these commands.

## **5. Configure Nginx**

Create a WordPress site configuration:

```
sudo nano /etc/nginx/sites-available/wordpress
```

Add the following configuration. Replace the PHP-FPM socket path with the one identified in Step 4.

```
server {
    listen 80;
    server_name _;

    root /var/www/wordpress;
    index index.php index.html;

    client_max_body_size 32M;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        try_files $uri =404;

        include fastcgi_params;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;

        fastcgi_param SCRIPT_FILENAME
            $document_root$fastcgi_script_name;
    }

    location ~ /\. {
        deny all;
    }

    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }
}
```

Enable the WordPress site:

```
sudo ln -s /etc/nginx/sites-available/wordpress \
  /etc/nginx/sites-enabled/wordpress
```

Disable the default site to avoid conflicts:

```
sudo rm /etc/nginx/sites-enabled/default
```

Test the Nginx configuration:

```
sudo nginx -t
```

Start Nginx and enable automatic startup:

```
sudo systemctl enable --now nginx
sudo systemctl reload nginx
```

## **6. Configure WordPress database access**

Go to the WordPress installation directory:

```
cd /var/www/wordpress
```

Create the WordPress configuration file:

```
sudo cp wp-config-sample.php wp-config.php
```

Edit the configuration:

```
sudo nano wp-config.php
```

Find and modify the following settings:

```
define('DB_NAME', 'wordpress');
define('DB_USER', 'wpuser');
define('DB_PASSWORD', 'ChangeThisStrongPassword!');
define('DB_HOST', 'localhost');
define('DB_CHARSET', 'utf8mb4');
```

Use the same database password that you configured in MariaDB.

### **Configure authentication keys and salts**

Replace the default WordPress authentication keys and salts with unique random values. You can generate them using the official WordPress service:

[WordPress secret-key generator](https://api.wordpress.org/secret-key/1.1/salt/)

Copy the generated definitions into `wp-config.php`, replacing the existing placeholder definitions.

Finally, protect the configuration file:

```
sudo chown root:www-data /var/www/wordpress/wp-config.php
sudo chmod 640 /var/www/wordpress/wp-config.php
```

This allows PHP-FPM to read the database configuration without allowing the web server account to modify it.

## **7. Complete the installation in a browser**

Find the Debian Server's IP address:

```
hostname -I
ip -4 addr
```

Open a web browser on your Debian Desktop or another workstation and navigate to:

```
http://SERVER_IP
```

[Setting Up Your Local Development Environment for WordPress – WordPress.com News](https://images.openai.com/static-rsc-4/mxS-jvHZ2P7K21Dh13sbLZhYOZkYdKnF6E5UskgQvnj_gLTdfG9ukOeSdLCpFADcYxC4XolPliPWVniO-wGODfPE6mQtrHgwlSSQmAMiS5RdwBWoV7b8ZqheAu1dLERd6x4AHYDlRsYRGtimEYJuxnaMz_NmMnPdwiaDVZYDScQ?purpose=inline)

Follow the WordPress installation wizard:

1. Select your preferred language.
2. Enter the website title.
3. Create a WordPress administrator account.
4. Set a strong administrator password.
5. Enter an email address.
6. Complete the installation and log in.

The WordPress administration interface is available at:

```
http://SERVER_IP/wp-admin/
```

## **8. Verify the installation**

Check the status of the services:

```
sudo systemctl status nginx
sudo systemctl status mariadb
sudo systemctl status php8.4-fpm
```

Use your installed PHP-FPM service name.

Check listening network ports:

```
sudo ss -tlnp
```

Expected connections:

| Service | Port or socket          | Purpose              |
| ------- | ----------------------- | -------------------- |
| Nginx   | TCP 80                  | HTTP                 |
| PHP-FPM | Unix socket             | Local PHP processing |
| MariaDB | TCP 3306 or Unix socket | Local database       |

MariaDB should not be exposed to external networks.

Test the web server:

```
curl -I http://localhost
```

You should receive an HTTP response from Nginx.

Test database connectivity:

```
mariadb -u wpuser -p wordpress
```

Once connected, check the tables:

```
SHOW TABLES;
EXIT;
```

After completing the WordPress installation, the database should contain WordPress tables such as `wp_posts`, `wp_users`, and `wp_options` (assuming the default table prefix).

## **9. Troubleshooting**

If WordPress does not work, investigate the service logs.

Nginx error log:

```
sudo tail -n 50 /var/log/nginx/error.log
```

PHP-FPM logs:

```
sudo journalctl -u php8.4-fpm -n 50
```

MariaDB logs:

```
sudo journalctl -u mariadb -n 50
```

Replace the PHP-FPM version as needed.

Common problems:

| Problem                                     | Possible cause                                            |
| ------------------------------------------- | --------------------------------------------------------- |
| 502 Bad Gateway                             | PHP-FPM is stopped or Nginx uses the wrong socket path    |
| Error establishing database connection      | Incorrect database credentials or MariaDB is stopped      |
| 403 Forbidden                               | Incorrect file permissions or Nginx configuration         |
| PHP source code displayed                   | PHP processing is not configured correctly                |
| Browser cannot connect                      | Network configuration, firewall, or Nginx service problem |
| WordPress installation page does not appear | Incorrect document root or missing WordPress files        |

If you need to check the Nginx configuration:

```
sudo nginx -t
```

Restart the relevant services after correcting configuration errors:

```
sudo systemctl restart nginx
sudo systemctl restart php8.4-fpm
```

## **10. Debian and Alpine: Key differences**

Both systems use the same WordPress application and MariaDB database, but their administration differs.

| Feature                  | Debian                        | Alpine                   |
| ------------------------ | ----------------------------- | ------------------------ |
| Package manager          | `apt`                         | `apk`                    |
| Service manager          | `systemctl`                   | `rc-service`             |
| Enable services          | `systemctl enable`            | `rc-update add`          |
| Web server user          | `www-data`                    | `nginx`                  |
| Nginx site configuration | `/etc/nginx/sites-available/` | `/etc/nginx/http.d/`     |
| PHP-FPM connection       | Unix socket by default        | Depends on configuration |
| Database                 | MariaDB                       | MariaDB                  |

## **✅ Final checklist**

**Installation verification**

- Nginx, PHP-FPM, and MariaDB are installed and running.
- MariaDB contains the WordPress database and a dedicated database user.
- WordPress files are extracted into /var/www/wordpress.
- Nginx forwards PHP requests to PHP-FPM through a Unix socket.
- WordPress connects successfully to MariaDB.
- The WordPress website opens from another computer.
- The WordPress administration interface is accessible.
- All three services start automatically after reboot.

**Security note:** This configuration is intended for an isolated classroom or laboratory network. Before publishing WordPress on the Internet, configure HTTPS, restrict network access, review filesystem permissions, and establish a backup and update strategy. In particular, protect `wp-config.php`, keep MariaDB local, and avoid using HTTP for administrator logins over untrusted networks.