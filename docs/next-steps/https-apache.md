# HTTPS Setup with Apache

This guide shows you how to set up a secure HTTPS connection for HCPI using Apache as a reverse proxy with SSL/TLS certificates. This is the setup currently used by most existing HCPI deployments (including Uganda's).

!!! info "For Production Only"
    This guide is intended for production Linux servers. Development environments typically don't require HTTPS setup.

## Prerequisites

- HCPI installed and running on a Linux server
- A domain name pointing to your server's IP address
- Root or sudo access to the server
- Ports 80 and 443 (or your chosen HTTPS port) open in your firewall

## Step 1: Install Apache

Install Apache on your Ubuntu server:

```bash
sudo apt update
sudo apt install -y apache2
```

Verify Apache is running:

```bash
sudo systemctl status apache2
```

## Step 2: Enable Required Apache Modules

Apache needs several modules for SSL and reverse proxying:

```bash
sudo a2enmod ssl proxy proxy_http proxy_wstunnel rewrite headers
sudo systemctl restart apache2
```

## Step 3: Install Your SSL Certificate

You have two options depending on where your certificate comes from.

### Option A: Let's Encrypt (free, auto-renewing)

Install Certbot and request a certificate:

```bash
sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache -d your-domain.com -d www.your-domain.com
```

Follow the prompts:

- Enter your email address
- Agree to the terms of service
- Choose whether to redirect HTTP to HTTPS (recommended: Yes)

Certbot will place the certificate and key under `/etc/letsencrypt/live/your-domain.com/`.

### Option B: Commercial certificate (wildcard or purchased)

If your organization already has an SSL certificate — common for EAC statistics offices — you'll have these three files. Copy them into the standard locations:

```bash
sudo cp your-certificate.cer /etc/ssl/certs/certificate.cer
sudo cp your-ca-bundle.crt  /etc/ssl/certs/ca_bundle.crt
sudo cp your-private.key    /etc/ssl/private/wildcard.key
sudo chmod 600 /etc/ssl/private/wildcard.key
```

!!! tip "Where do these files come from?"
    Your certificate provider (e.g. DigiCert, Sectigo) sends them by email or through their control panel. If someone else in your organization manages SSL, ask them for the certificate file, the CA bundle, and the private key.

## Step 4: Configure the Apache Virtual Host

Create a new Apache config file for HCPI:

```bash
sudo nano /etc/apache2/sites-available/hcpi.conf
```

Paste the configuration below. Replace `your-domain.com` with your actual domain, and adjust the certificate paths if you used Let's Encrypt in Step 3.

```apache
# HTTP server - redirects to HTTPS
<VirtualHost *:80>
    ServerName your-domain.com
    ServerAlias www.your-domain.com

    RewriteEngine on
    RewriteCond %{SERVER_NAME} =your-domain.com [OR]
    RewriteCond %{SERVER_NAME} =www.your-domain.com
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>

# HTTPS server
<VirtualHost *:443>
    ServerName your-domain.com
    ServerAlias www.your-domain.com

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/certificate.cer
    SSLCertificateKeyFile /etc/ssl/private/wildcard.key
    SSLCertificateChainFile /etc/ssl/certs/ca_bundle.crt

    # Security headers
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"

    # Logging
    ErrorLog  /var/log/apache2/hcpi.error.log
    CustomLog /var/log/apache2/hcpi.access.log combined

    # Reverse proxy to HCPI
    ProxyPreserveHost On
    ProxyRequests Off
    ProxyPass        / http://localhost:9201/
    ProxyPassReverse / http://localhost:9201/

    RequestHeader set X-Forwarded-Proto "https"

    # WebSocket support (for long-polling / chatter notifications)
    RewriteEngine on
    RewriteCond %{HTTP:Upgrade}   =websocket [NC]
    RewriteRule /(.*)             ws://localhost:9201/$1 [P,L]
</VirtualHost>
```

!!! warning "If HCPI runs on a different port"
    Change `http://localhost:9201/` in both `ProxyPass` lines to match the `http_port` from your `hcpi.conf`.

!!! note "Running on a non-standard HTTPS port"
    Some existing deployments listen on a non-standard port (e.g. `*:9200`) instead of `*:443`. If that's what you need, change `<VirtualHost *:443>` to `<VirtualHost *:9200>` and make sure the port is declared in `/etc/apache2/ports.conf`:

    ```apache
    Listen 9200
    ```

## Step 5: Enable the Site

Disable the default site and enable HCPI:

```bash
sudo a2dissite 000-default.conf default-ssl.conf
sudo a2ensite hcpi.conf
```

Test the configuration for syntax errors:

```bash
sudo apache2ctl configtest
```

You should see `Syntax OK`.

## Step 6: Configure the Firewall

Allow HTTP and HTTPS traffic, and remove direct access to HCPI's port:

```bash
sudo ufw allow 'Apache Full'
sudo ufw delete allow 9201 2>/dev/null || true
```

## Step 7: Restart Apache

```bash
sudo systemctl restart apache2
```

Check it came up clean:

```bash
sudo systemctl status apache2
```

## Step 8: Make HCPI Proxy-Aware

Tell HCPI that it sits behind a reverse proxy so it generates correct HTTPS URLs. Edit `/opt/hcpi/conf/hcpi.conf`:

```ini
[options]
proxy_mode = True
http_interface = 127.0.0.1
http_port = 9201
```

Then restart HCPI:

```bash
sudo systemctl restart hcpi
```

## Step 9: Test Your Setup

Visit your domain in a web browser:

```
https://your-domain.com
```

You should see:

- A secure padlock icon
- The HCPI login page
- A valid SSL certificate

## Step 10: Certificate Renewal

### For Let's Encrypt

Certbot installs a renewal timer automatically. Verify it:

```bash
sudo certbot renew --dry-run
```

### For Commercial Certificates

These expire annually. Set a calendar reminder to replace the files in `/etc/ssl/certs/` and `/etc/ssl/private/` before the expiry date, then restart Apache:

```bash
sudo systemctl restart apache2
```

## Troubleshooting

### Apache Won't Start

Check the error log:

```bash
sudo tail -f /var/log/apache2/error.log
```

Re-run the syntax check:

```bash
sudo apache2ctl configtest
```

### 502 Bad Gateway

Means Apache can reach itself but not HCPI. Confirm HCPI is running and listening on the port named in `ProxyPass`:

```bash
sudo systemctl status hcpi
sudo ss -ltnp | grep 9201
```

### Browser Shows "Not Secure"

- Confirm you visited `https://`, not `http://`
- Verify the certificate paths in your config point to files that exist
- Check the certificate matches the domain you're visiting

### WebSocket / Chatter Not Updating

The `RewriteRule` block in Step 4 handles this. If you omitted it, long-polling features (notifications, chat) will fail. Re-add the block and restart Apache.

## Next Steps

- Set up regular database backups
- Configure monitoring and alerting
- Review the [Odoo 18 documentation](https://www.odoo.com/documentation/18.0/) for advanced configuration
