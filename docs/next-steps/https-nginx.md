# HTTPS Setup with Nginx

This guide shows you how to set up a secure HTTPS connection for HCPI using Nginx as a reverse proxy with SSL/TLS certificates.

!!! info "For Production Only"
    This guide is intended for production Linux servers. Development environments typically don't require HTTPS setup.

## Prerequisites

- HCPI installed and running on a Linux server
- A domain name pointing to your server's IP address
- Root or sudo access to the server
- Ports 80 and 443 open in your firewall

## Step 1: Install Nginx

Install Nginx on your Ubuntu server:

```bash
sudo apt update
sudo apt install -y nginx
```

Verify Nginx is running:

```bash
sudo systemctl status nginx
```

## Step 2: Install Certbot for SSL Certificates

Install Certbot to obtain free SSL certificates from Let's Encrypt:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

## Step 3: Configure Nginx for HCPI

Create a new Nginx configuration file for HCPI:

```bash
sudo nano /etc/nginx/sites-available/hcpi
```

Add the following configuration (replace `your-domain.com` with your actual domain):

```nginx
# HTTP server - redirects to HTTPS
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;

    # Redirect all HTTP traffic to HTTPS
    return 301 https://$server_name$request_uri;
}

# HTTPS server
server {
    listen 443 ssl http2;
    server_name your-domain.com www.your-domain.com;

    # SSL certificate paths (will be configured by Certbot)
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    # Logging
    access_log /var/log/nginx/hcpi_access.log;
    error_log /var/log/nginx/hcpi_error.log;

    # Proxy settings
    location / {
        proxy_pass http://127.0.0.1:9201;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Timeouts
        proxy_connect_timeout 600;
        proxy_send_timeout 600;
        proxy_read_timeout 600;
        send_timeout 600;
    }

    # Increase client max body size for file uploads
    client_max_body_size 100M;
}
```

!!! warning "Update Domain Name"
    Replace `your-domain.com` with your actual domain name throughout the configuration file.

## Step 4: Enable the Configuration

Create a symbolic link to enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/hcpi /etc/nginx/sites-enabled/
```

Remove the default Nginx site if it exists:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Test the Nginx configuration:

```bash
sudo nginx -t
```

## Step 5: Obtain SSL Certificate

Use Certbot to obtain and install an SSL certificate:

```bash
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

Follow the prompts:
- Enter your email address
- Agree to the terms of service
- Choose whether to redirect HTTP to HTTPS (recommended: Yes)

Certbot will automatically:
- Obtain the SSL certificate
- Configure Nginx to use it
- Set up automatic renewal

## Step 6: Configure Firewall

Allow HTTP and HTTPS traffic through the firewall:

```bash
sudo ufw allow 'Nginx Full'
sudo ufw delete allow 9201
```

This allows ports 80 and 443, and removes direct access to port 9201.

## Step 7: Restart Services

Restart Nginx to apply changes:

```bash
sudo systemctl restart nginx
```

Ensure HCPI is running:

```bash
sudo systemctl status hcpi
```

## Step 8: Test Your Setup

Visit your domain in a web browser:

```
https://your-domain.com
```

You should see:
- A secure padlock icon in the browser
- HCPI login page
- Valid SSL certificate

## Step 9: Configure Auto-Renewal

Certbot automatically sets up certificate renewal. Test the renewal process:

```bash
sudo certbot renew --dry-run
```

If successful, your certificates will automatically renew before expiration.

## Update HCPI Configuration (Optional)

You can configure HCPI to only listen on localhost since Nginx will handle external connections:

Edit `/opt/hcpi/conf/hcpi.conf`:

```ini
[options]
# ... other settings ...
http_interface = 127.0.0.1
http_port = 9201
```

Restart HCPI:

```bash
sudo systemctl restart hcpi
```

## Troubleshooting

### Nginx Won't Start

Check the error log:
```bash
sudo tail -f /var/log/nginx/error.log
```

Verify configuration syntax:
```bash
sudo nginx -t
```

### Certificate Issues

Check Certbot logs:
```bash
sudo tail -f /var/log/letsencrypt/letsencrypt.log
```

Verify domain DNS is pointing to your server:
```bash
nslookup your-domain.com
```

### 502 Bad Gateway Error

Ensure HCPI is running:
```bash
sudo systemctl status hcpi
```

Check HCPI logs:
```bash
tail -f /opt/hcpi/log/hcpi.log
```

### Connection Timeout

Check firewall rules:
```bash
sudo ufw status
```

Verify Nginx is listening on ports 80 and 443:
```bash
sudo netstat -tlnp | grep nginx
```

## Performance Optimization

### Enable Gzip Compression

Add to your Nginx configuration inside the `server` block:

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
gzip_min_length 1000;
```

### Enable Caching for Static Files

Add inside the `server` block:

```nginx
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
    proxy_pass http://127.0.0.1:9201;
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

### Rate Limiting

Protect against abuse by adding rate limiting:

```nginx
# Add this outside the server block
limit_req_zone $binary_remote_addr zone=hcpi:10m rate=10r/s;

# Add this inside the server block
location / {
    limit_req zone=hcpi burst=20 nodelay;
    # ... rest of proxy configuration ...
}
```

## Security Best Practices

1. **Keep Software Updated**: Regularly update Nginx, Certbot, and the OS
2. **Monitor Logs**: Review Nginx and HCPI logs regularly
3. **Use Strong SSL Configuration**: Keep SSL protocols and ciphers up to date
4. **Implement Fail2ban**: Protect against brute force attacks
5. **Regular Backups**: Back up your Nginx configuration and SSL certificates

## Next Steps

- Set up regular database backups
- Configure monitoring and alerting
- Implement log rotation for Nginx and HCPI logs
- Consider setting up a CDN for static assets
- Review and harden server security settings
