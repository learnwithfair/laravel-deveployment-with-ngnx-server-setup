# Laravel Production Deployment Documentation

[![Youtube][youtube-shield]][youtube-url]
[![Facebook][facebook-shield]][facebook-url]
[![Instagram][instagram-shield]][instagram-url]
[![LinkedIn][linkedin-shield]][linkedin-url]

Thanks for visiting my GitHub account!


**Environment:** Ubuntu + Nginx + PHP 8.3 + MySQL + Supervisor + Scheduler

---

## Prerequisites

* Ubuntu Server (22.04 LTS or newer recommended)
* SSH access with `sudo` privileges
* Domain name pointing to the server (optional but recommended)
* Basic knowledge of Linux command line

---

## Step 1: Install and Configure Nginx

Nginx is used as the web server to serve the Laravel application.

```bash
ssh username@IPAddress
sudo apt update
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

Verify installation:

```bash
nginx -v
```

---

## Step 2: Install PHP 8.3 and Required Extensions

Laravel requires PHP and several extensions to operate correctly.

```bash
sudo apt install -y php8.3 \
php8.3-common \
php8.3-opcache \
php8.3-cli \
php8.3-gd \
php8.3-curl \
php8.3-mysql \
php8.3-mbstring \
php8.3-zip \
php8.3-xml \
php8.3-intl \
php8.3-bcmath \
php8.3-soap \
php8.3-fpm \
php8.3-imagick \
php8.3-ldap
```

Verify PHP version:

```bash
php -v
```

---

## Step 3: Install and Configure MySQL

```bash
sudo apt install -y mysql-server
sudo systemctl start mysql
sudo systemctl enable mysql
```

---

## Step 4: Create Database and User

Log into MySQL:

```bash
sudo mysql
```

Run the following commands:

```sql
CREATE DATABASE aero_select;
CREATE USER 'admin'@'localhost' IDENTIFIED BY 'DatabasePassowrd';
GRANT ALL PRIVILEGES ON database_name.* TO 'admin'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## Step: Install nano (editor)
```bash
sudo apt update
sudo apt install nano -y
nano --version
```

---

## Step 5: Install Supervisor (Queue Management)

Supervisor keeps Laravel queue workers running continuously.

```bash
sudo apt install -y supervisor
sudo systemctl start supervisor
sudo systemctl enable supervisor
```

---

## Step 6: Install Git

```bash
sudo apt install -y git
```

---

## Step 7: Clone the Project

```bash
cd /var/www
sudo git clone https://github.com/learnwithfair/christophlombar.git
```

Set ownership:

```bash
sudo chown -R www-data /var/www/christophlombar
```

---

## Step 8: Configure Environment Variables

```bash
cd /var/www/christophlombar
cp .env.example .env
sudo nano .env
```

Set database values:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=
DB_USERNAME=admin
DB_PASSWORD=
```

Set application URL:

```env
APP_URL=https://cockpit.aeroselect.io
```

---

## Step 9: Install Dependencies and App Key

```bash
composer install --no-dev --optimize-autoloader
php artisan key:generate
```

---

## Step 10: Set File and Directory Permissions

Laravel requires write access to specific directories.

```bash
 cd /var/www/christophlombar
 # Set the web server as the owner
 sudo chown -R www-data:www-data .
 # Add your user to group
 sudo usermod -a -G www-data user
 # Set file(s) permission
 sudo find . -type f -exec chmod 644 {} \;
 # Set folder(s) permission
 sudo find . -type d -exec chmod 755 {} \;
 # Set cache directory permission
 sudo chgrp -R www-data storage bootstrap/cache
 sudo chmod -R ug+rwx storage bootstrap/cache
```

For uploads:

```bash
mkdir -p public/uploads
sudo chown -R www-data public/uploads
sudo chmod -R 775 public/uploads
```

---

## Step 11: Configure Nginx Virtual Host

Create configuration file:

```bash
sudo nano /etc/nginx/sites-available/christophlombar
```

Example configuration:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com;  # Replace with your domain
    root /var/www/christophlombar/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}

```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/christophlombar /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

## Step 12: Database Migration and Cache Optimization

```bash
php artisan migrate --force
php artisan optimize:clear
php artisan config:cache
```

---

## Step 13: Configure Supervisor Queue Worker

Create the worker.log file
```bash
touch /var/www/christophlombar/storage/logs/worker.log
```

Set correct permissions
```bash
sudo chown -R www-data /var/www/christophlombar/storage
sudo chmod -R 775 /var/www/christophlombar/storage
sudo chmod 664 /var/www/christophlombar/storage/logs/worker.log
```

Create Supervisor config:

```bash
sudo nano /etc/supervisor/conf.d/christophlombar-worker.conf
```

```ini
[program:christophlombar-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/christophlombar/artisan queue:work --sleep=3 --tries=3
autostart=true
autorestart=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/christophlombar/storage/logs/worker.log
stopwaitsecs=3600
```

Apply changes:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start christophlombar-worker:*
```

---

## Step 14: Configure Laravel Scheduler (Cron)

Laravel Scheduler requires a system cron job.

```bash
crontab -e
```

Add:

```cron
* * * * * cd /var/www/christophlombar && php artisan schedule:run >> /dev/null 2>&1
```

This enables scheduled tasks such as:

* Email processing
* Subscription checks
* Custom commands

---

## Step 15: Restart Queue Workers (When Code Changes)

Whenever you deploy new code or change jobs:

```bash
php artisan queue:restart
```

Or via Supervisor:

```bash
sudo supervisorctl restart christophlombar-worker:*
```

---

## Step 16: Secure the Application with SSL (Certbot)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d cockpit.aeroselect.io
```

Test auto-renewal:

```bash
sudo certbot renew --dry-run
```

---

## Conclusion

You now have a fully production-ready Laravel deployment with:

* Nginx web server
* PHP 8.3 with required extensions
* MySQL database
* Background queue processing via Supervisor
* Task scheduling via Laravel Scheduler + Cron
* Secure HTTPS with Let’s Encrypt
* Proper file permissions and logging

This setup is suitable for scalable, maintainable, and secure Laravel applications in production.


## Follow Me

[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/github.svg' alt='github' height='40'>](https://github.com/learnwithfair) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/facebook.svg' alt='facebook' height='40'>](https://www.facebook.com/learnwithfair/) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/instagram.svg' alt='instagram' height='40'>](https://www.instagram.com/learnwithfair/) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/twitter.svg' alt='twitter' height='40'>](https://www.twiter.com/learnwithfair/) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/youtube.svg' alt='YouTube' height='40'>](https://www.youtube.com/@learnwithfair)

 <!-- MARKDOWN LINKS & IMAGES  -->

[youtube-shield]: https://img.shields.io/badge/-Youtube-black.svg?style=flat-square&logo=youtube&color=555&logoColor=white
[youtube-url]: https://youtube.com/@learnwithfair
[facebook-shield]: https://img.shields.io/badge/-Facebook-black.svg?style=flat-square&logo=facebook&color=555&logoColor=white
[facebook-url]: https://facebook.com/learnwithfair
[instagram-shield]: https://img.shields.io/badge/-Instagram-black.svg?style=flat-square&logo=instagram&color=555&logoColor=white
[instagram-url]: https://instagram.com/learnwithfair
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=flat-square&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/company/learnwithfair

#learnwithfair #rahtulrabbi #rahatul-rabbi #learn-with-fair

