---
layout: post
title: "Running Nginx and Apache with Let's Encrypt: From Install to Auto-Renewing TLS"
date: Oct-13-2012
categories: [Linux, Networking]
tags: [linux, nginx, apache, tls, letsencrypt, certbot]
image: /assets/img/nginx-apache-tls-padlock.jpg
---

You can check when a certificate expires from the command line, but someone still has to issue it, configure the server to use it, and renew it before it does expire. Nginx and Apache are the two web servers you'll meet on almost every Linux box, and Certbot turned "getting a TLS certificate" from a paperwork process into a single command.

## Choosing Between Nginx and Apache

Both serve HTTP and HTTPS perfectly well; the practical differences show up in configuration style and how they handle concurrent connections. Nginx uses an event-driven model that scales well under heavy concurrent load with a lean config syntax; Apache's `.htaccess` support and module ecosystem remain unmatched for certain legacy and CMS-driven setups.

```bash
# Nginx
sudo apt install nginx    # Debian/Ubuntu
sudo dnf install nginx    # RHEL/CentOS/Fedora

# Apache
sudo apt install apache2       # Debian/Ubuntu
sudo dnf install httpd         # RHEL/CentOS/Fedora

# Confirm either is running
sudo systemctl status nginx
sudo systemctl status apache2
```

*Tip:* Running both on the same host means a port conflict on 80/443. Pick one for a given site, or put one in front of the other as a reverse proxy if you genuinely need both.

## Nginx Server Block Basics

Nginx organizes virtual hosts as "server blocks," typically one file per site under `sites-available`, symlinked into `sites-enabled` when active.

```nginx
# /etc/nginx/sites-available/example.com
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/example.com;
    index index.html;
}
```

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/

# Reload after changes
sudo nginx -t && sudo systemctl reload nginx
```

*Tip:* Nginx uses `listen` plus `server_name` to decide which block handles a request, not file location. Two blocks with the same `server_name` will silently conflict; `nginx -t` catches most but not all of these issues.

## Apache Virtual Host Basics

Apache's equivalent concept is the virtual host, enabled and disabled with dedicated helper commands rather than manual symlinks.

```apache
# /etc/apache2/sites-available/example.com.conf
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example.com
</VirtualHost>
```

```bash
# Enable the site and required modules
sudo a2ensite example.com.conf
sudo a2enmod rewrite ssl

# Reload after changes
sudo systemctl reload apache2
```

*Tip:* `a2ensite`/`a2dissite` and `a2enmod`/`a2dismod` only create or remove symlinks; they don't touch the actual config files. That makes toggling a site or module on and off trivially reversible.

## Testing Configuration Before Reloading

Both servers can validate a configuration without applying it, which catches a typo before it takes down the site at reload time.

```bash
# Nginx
sudo nginx -t

# Apache
sudo apachectl configtest
```

*Tip:* Make this a reflex, not an occasional habit: `config test, then reload`, every single time, with no exceptions for "it's just a small change."

## Getting a Free Certificate with Certbot

Certbot, from the EFF, automates the entire ACME protocol conversation with Let's Encrypt: proving you control the domain and installing the resulting certificate.

```bash
# Install Certbot
sudo apt install certbot    # Debian/Ubuntu
sudo dnf install certbot    # RHEL/CentOS/Fedora

# Get a certificate without touching your web server config (webroot method)
sudo certbot certonly --webroot -w /var/www/example.com -d example.com -d www.example.com
```

*Tip:* `certonly` just obtains the certificate files; it's the right choice when you want full control over how your server config references them, rather than letting Certbot edit the config itself.

## Certbot's Nginx and Apache Plugins

For the common case, Certbot can also edit your web server's configuration directly, adding the certificate paths and setting up the HTTPS server block for you.

```bash
# Nginx plugin: obtains the cert and edits the config automatically
sudo apt install python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com

# Apache plugin: same idea
sudo apt install python3-certbot-apache
sudo certbot --apache -d example.com -d www.example.com
```

*Tip:* The plugin flow is genuinely "answer two prompts and you have HTTPS," but it does modify your config files. Keep them in version control so you can see exactly what Certbot changed.

## Verifying Auto-Renewal

Let's Encrypt certificates are valid for 90 days by design, specifically to force renewal automation rather than let certificates quietly expire a year later.

```bash
# Test the renewal process without actually renewing
sudo certbot renew --dry-run

# See what's scheduled to renew and when
sudo certbot certificates

# Check the systemd timer that normally handles this automatically
systemctl list-timers | grep certbot
```

*Tip:* Run `certbot renew --dry-run` right after initial setup, not just when something's already broken. It's the only way to know the renewal path actually works before you're relying on it at 90 days.

## Reverse Proxying to a Backend Application

Nginx and Apache both commonly sit in front of an application server, terminating TLS and forwarding plain HTTP to the app.

```nginx
# Nginx reverse proxy
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

```apache
# Apache reverse proxy
<VirtualHost *:80>
    ServerName example.com
    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/
</VirtualHost>
```

*Tip:* Always set `X-Real-IP` or `X-Forwarded-For` when proxying. Without it, every request in your backend's logs appears to come from `127.0.0.1`, which makes debugging abuse or traffic patterns far harder than it needs to be.

## Bonus: Redirecting HTTP to HTTPS and Enabling HSTS

Once a certificate is in place, plain HTTP should exist only to redirect to HTTPS, and HSTS tells browsers to skip HTTP entirely on future visits.

```nginx
# Nginx: redirect all HTTP to HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

# Add HSTS inside the HTTPS server block
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

*Tip:* Test HSTS carefully before setting a long `max-age`. Browsers cache the HSTS policy, so if you later need to serve plain HTTP again for any reason, visitors who already received the header will refuse to downgrade until it expires.
