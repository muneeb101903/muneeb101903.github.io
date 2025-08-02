---
title: Debian WordPress Deployment
date: 2025-08-02
description: This guide will help deploy WordPress on Debian with Caddy
categories: ['Tutorials']
draft: false # Change to true to not render the post in on the website
---

<h3>**What Is WordPress**</h3>

WordPress is a free and open-source content management system (CMS) used to create and manage websites.

What You Can Do with WordPress:
  -Create blogs, personal websites, portfolios
  -Build business sites, online stores (with WooCommerce)
  -Publish articles, manage media, users, and comments
  -Use themes to change appearance
  -Use plugins to add functionality (e.g. contact forms, SEO, backups)

<h3>**Why Should You use WordPress**</h3>

1. No Coding Needed (But It's Developer-Friendly)
2. It's Free and Open Source
3. Thousands of Themes and Plugins
4. Built for SEO
5. You Can Build Anything
6. Scalable & Flexible
7. Regular Updates & Security
8. Global Community

<h3>**The Steps**</h3>

In order to deploy WordPress through Caddy on Debian you are going to need to do the following:

Instal PHP and Required Extensions
<pre>sudo apt update
sudo apt install php php-fpm php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip unzip curl -y
</pre>

Install Docker 
<pre>sudo apt-get install docker
sudo systemctl enable docker --now
</pre>

Start MariaDB in a container
<pre>
docker run -d \
  --name wordpress-db \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wpuser \
  -e MYSQL_PASSWORD=wppassword \
  -p 3306:3306 \
  mariadb:latest
</pre>

Install Caddy Web Server
<pre>
sudo apt install -y debian-keyring debian-archive-keyring curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
</pre>

Download and Configure WordPress
<pre>
cd /var/www/
sudo curl -O https://wordpress.org/latest.tar.gz
sudo tar -xvzf latest.tar.gz
sudo chown -R www-data:www-data wordpress
sudo chmod -R 755 wordpress
</pre>

Copy Sample config
<pre>cd wordpress
cp wp-config-sample.php wp-config.php
</pre>

Edit wp-config.php
<pre>
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'wppassword' );
define( 'DB_HOST', '127.0.0.1:3306' ); // Docker container exposes this
</pre>

Configure Caddy and Edit /etc/caddy/Caddyfile
<pre>
:80 {
    root * /var/www/wordpress
    php_fastcgi unix//run/php/php-fpm.sock
    encode gzip
    file_server
}
</pre>

