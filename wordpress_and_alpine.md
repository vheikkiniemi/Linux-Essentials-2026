> [!NOTE]
> **🇬🇧 Note on the use of AI:** AI tools such as ChatGPT, Gemini, and Copilot were used to help develop this material.  
> **🇫🇮 Huomio tekoälyn käytöstä:** Tämän materiaalin laatimisessa on hyödynnetty tekoälytyökaluja, kuten ChatGPT:tä, Geminiä ja Copilotia.  

# **Installing WordPress on Alpine Linux with MariaDB**

## **🎯 Objective**

Install and configure a working WordPress website on Alpine Linux using the following components:

- **Alpine Linux** – Operating system
- **Nginx** – Web server
- **PHP-FPM** – Executes WordPress PHP code
- **MariaDB** – Stores WordPress data
- **WordPress** – Content management system (CMS)

The WordPress source code archive (`wordpress.tar.gz`) has already been downloaded to your home directory.

After completing this guide, you should be able to:

- Install and manage the required services on Alpine.
- Create a MariaDB database and a dedicated database user.
- Extract and configure WordPress.
- Configure Nginx and PHP-FPM to serve WordPress.
- Access and manage WordPress through a web browser.
- Verify the installation and troubleshoot common problems.

## **System architecture**

**Web browser**

http\://SERVER_IP

**Nginx**

HTTP server · Port 80

**PHP-FPM + WordPress**

PHP application processing

**MariaDB**

Local database · Port 3306

## **1. Update Alpine and install packages**

Update the package index and installed packages:

```
sudo apk update
sudo apk upgrade
```

Install Nginx, MariaDB, PHP-FPM, and the PHP extensions required by WordPress:

```
sudo apk add nginx mariadb mariadb-client \
  php php-fpm php-mysqli php-pdo_mysql \
  php-curl php-gd php-mbstring \
  php-xml php-zip php-intl \
  php-openssl php-phar php-session \
  php-fileinfo php-dom php-simplexml \
  php-xmlreader php-xmlwriter
```

The exact PHP package names may vary between Alpine releases. Check the installed PHP version:

```
php -v
```

## **2. Configure MariaDB**

Initialize the database:

```
sudo /etc/init.d/mariadb setup
```

Start MariaDB and enable automatic startup:

```
sudo rc-service mariadb start
sudo rc-update add mariadb default
```

Run the initial security configuration:

```
sudo mariadb-secure-installation
```

Follow the prompts to remove anonymous users and the test database and to disable remote root login.

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
sudo chown -R nginx:nginx /var/www/wordpress

sudo find /var/www/wordpress -type d -exec chmod 755 {} \;

sudo find /var/www/wordpress -type f -exec chmod 644 {} \;
```

On Debian, Nginx and PHP-FPM commonly use the `www-data` account, whereas Alpine commonly uses `nginx`.

The Nginx user will now have access to the WordPress files.

## **4. Configure PHP-FPM**

Find the PHP-FPM service and its configuration:

```
ls /etc/init.d/php-fpm*
ls /etc/php*/php-fpm.d/www.conf
```

The service and configuration paths depend on the installed PHP version.

Open the pool configuration file, replacing `XX` with the installed PHP version number:

```
sudo nano /etc/phpXX/php-fpm.d/www.conf
```

For this exercise, configure PHP-FPM to listen on a local TCP port:

```
user = nginx
group = nginx

listen = 127.0.0.1:9000
```

Make sure that there is only one active `listen` directive.

Start PHP-FPM and enable it at boot. Replace `php-fpmXX` with the actual service name shown earlier:

```
sudo rc-service php-fpmXX start
sudo rc-update add php-fpmXX default
```

Verify that PHP-FPM is listening:

```
sudo netstat -tln | grep 9000
```

## **5. Configure Nginx**

Create a WordPress configuration file:

```
sudo nano /etc/nginx/http.d/wordpress.conf
```

Add the following configuration:

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
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME
            $document_root$fastcgi_script_name;
    }

    location ~ /\. {
        deny all;
    }
}
```

Check whether another default server configuration exists:

```
ls /etc/nginx/http.d/
```

If `default.conf` exists and also listens on port 80 as the default server, disable it to avoid conflicts:

```
sudo mv /etc/nginx/http.d/default.conf \
  /etc/nginx/http.d/default.conf.disabled
```

Test the Nginx configuration:

```
sudo nginx -t
```

Start Nginx and enable it at boot:

```
sudo rc-service nginx start
sudo rc-update add nginx default
```

## **6. Configure WordPress database access**

Go to the WordPress installation directory:

```
cd /var/www/wordpress
```

Create the configuration file:

```
sudo cp wp-config-sample.php wp-config.php
```

Edit it:

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
sudo chown nginx:nginx /var/www/wordpress/wp-config.php
sudo chmod 600 /var/www/wordpress/wp-config.php
```

## **7. Complete the installation in a browser**

Find the server's IP address:

```
hostname -I
ip -4 addr
```

Open a web browser on your Debian Desktop or another workstation and navigate to:

```
http://SERVER_IP
```

[How to install WordPress on localhost (7 easy steps) | by Themeum | Medium](https://images.openai.com/static-rsc-4/nhrY8wpQbOjqQX9gNVJjKSDgTCgjLslnP8JvSHFUDgRlJhpa2b2trDIGZb3JtAsc194sGhXwJ-mCf11jZ-0Q3MxC2lMZ20X2NV9pB40nYtiJ-UBWvs-QgFynznAdE6vxcx8Qen5S7AQxuRbuo60NoLMlvO12BYdjgxfUEIyQ2KY?purpose=inline)

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
sudo rc-service nginx status
sudo rc-service mariadb status
sudo rc-service php-fpmXX status
```

Check listening network ports:

```
sudo netstat -tln
```

Expected connections:

| Service | Port | Purpose              |
| ------- | ---- | -------------------- |
| Nginx   | 80   | HTTP                 |
| PHP-FPM | 9000 | Local PHP processing |
| MariaDB | 3306 | Local database       |

PHP-FPM should listen only on `127.0.0.1:9000`. MariaDB should also remain inaccessible from external networks.

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

Check PHP-FPM:

```
sudo rc-service php-fpmXX status
```

MariaDB logs:

```
sudo rc-service mariadb status
```

Common problems:

| Problem                                | Possible cause                                            |
| -------------------------------------- | --------------------------------------------------------- |
| 502 Bad Gateway                        | PHP-FPM is stopped or Nginx uses the wrong PHP-FPM port   |
| Error establishing database connection | Incorrect database credentials or MariaDB is stopped      |
| 403 Forbidden                          | Incorrect file permissions or Nginx configuration         |
| PHP source code displayed              | PHP processing is not configured correctly                |
| Browser cannot connect                 | Network configuration, firewall, or Nginx service problem |

```
sudo nginx -t
```

Restart the relevant services after correcting configuration errors:

```
sudo rc-service nginx restart
sudo rc-service php-fpmXX restart
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

- Nginx, PHP-FPM and MariaDB are installed and running.
- MariaDB contains the WordPress database and a dedicated database user.
- WordPress files are extracted into /var/www/wordpress.
- Nginx forwards PHP requests to PHP-FPM.
- WordPress connects successfully to MariaDB.
- The WordPress website opens from another computer.
- The WordPress administration interface is accessible.

**Security note:** This configuration is intended for an isolated classroom or laboratory network. Before publishing WordPress on the Internet, configure HTTPS, restrict network access, review filesystem permissions, and establish a backup and update strategy. In particular, protect `wp-config.php`, keep MariaDB local, and avoid using HTTP for administrator logins over untrusted networks.