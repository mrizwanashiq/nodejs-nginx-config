# Setting Up Nginx and Auto-Renewing SSL for NestJS on Amazon Linux EC2

This guide details how to configure Nginx as a reverse proxy for a NestJS application on an Amazon Linux 2 EC2 instance, secure it with a free Let’s Encrypt SSL certificate, and enable automatic certificate renewal. The setup is tailored for the domain `examle.com`, with the NestJS app running on port 3000.

## Prerequisites

- **EC2 Instance**: Amazon Linux 2, with SSH access and `sudo` privileges.
- **Domain**: `examle.com` with an A record pointing to the EC2 instance’s public IP.
- **NestJS App**: Running on port 3000 (adjust if different).
- **Security Group**: Allows inbound traffic on ports 80 (HTTP) and 443 (HTTPS).
- **DNS**: Verify resolution with:
  ```bash
  dig examle.com
  ```

## Step-by-Step Instructions

### 1. Update System Packages
Ensure the system is up to date to avoid compatibility issues:
```bash
sudo yum update -y
```

### 2. Install Nginx
Install Nginx to serve as a reverse proxy for the NestJS app:
```bash
sudo yum install -y nginx
```

Verify installation:
```bash
nginx -v
```

### 3. Resolve Port Conflicts
Check if port 80 is in use:
```bash
sudo netstat -tuln | grep ':80' || sudo ss -tuln | grep ':80'
```

If a process (e.g., Apache `httpd`) is using port 80, stop and disable it:
```bash
sudo systemctl stop httpd
sudo systemctl disable httpd
```

Alternatively, terminate the conflicting process:
```bash
sudo lsof -i :80
sudo kill -9 <PID>  # Replace <PID> with the process ID
```

Verify port 80 is free (no output):
```bash
sudo ss -tuln | grep ':80'
```

### 4. Start and Enable Nginx
Start Nginx and ensure it runs on boot:
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

Verify Nginx is running:
```bash
sudo systemctl status nginx
```
Confirm `Active: active (running)` in the output.

### 5. Configure Nginx for NestJS
Create an optimized Nginx configuration for the NestJS app:
```bash
sudo mkdir -p /etc/nginx/conf.d
cat | sudo tee /etc/nginx/conf.d/nestjs.conf <<EOF
# Upstream configuration for load balancing and connection pooling
upstream nestjs_app {
    server localhost:3000;
    keepalive 32; # Maintain persistent connections to reduce overhead
}

server {
    listen 80;
    server_name examle.com;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Gzip compression for better performance
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 256;

    location / {
        proxy_pass http://nestjs_app; # Use upstream for better performance
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering on; # Enable buffering for stable responses
        proxy_buffer_size 4k;
        proxy_buffers 4 32k;
    }

    # Error handling
    error_page 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
EOF
```

Test the configuration:
```bash
sudo nginx -t
```
Expected output:
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Apply the configuration:
```bash
sudo systemctl reload nginx
```

### 6. Install Certbot
Install Certbot for Let’s Encrypt SSL certificates using `amazon-linux-extras` on Amazon Linux 2:
```bash
sudo amazon-linux-extras install epel -y
sudo yum install -y certbot python2-certbot-nginx
```

If `amazon-linux-extras` fails or Certbot is unavailable, use `pip` as a fallback:
```bash
sudo yum install -y python2-pip
sudo pip install certbot certbot-nginx
```

Verify installation:
```bash
certbot --version
```

Check Certbot binary path:
```bash
which certbot
```
Note the path (e.g., `/usr/bin/certbot` or `/usr/local/bin/certbot`) for Step 8.

### 7. Obtain SSL Certificate
Use Certbot to issue an SSL certificate and configure Nginx for HTTPS:
```bash
sudo certbot --nginx -d examle.com --non-interactive --agree-tos --email mobeenikhtiar21@gmail.com --redirect
```

If the `certbot-nginx` plugin fails, use standalone mode:
```bash
sudo systemctl stop nginx
sudo certbot certonly --standalone -d examle.com --non-interactive --agree-tos --email mobeenikhtiar21@gmail.com
sudo systemctl start nginx
```

Manually update `/etc/nginx/conf.d/nestjs.conf` for SSL:
```bash
cat | sudo tee /etc/nginx/conf.d/nestjs.conf <<EOF
upstream nestjs_app {
    server localhost:3000;
    keepalive 32;
}

server {
    listen 443 ssl;
    server_name examle.com;

    ssl_certificate /etc/letsencrypt/live/examle.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/examle.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 256;

    location / {
        proxy_pass http://nestjs_app;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 4 32k;
    }

    error_page 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}

server {
    listen 80;
    server_name examle.com;
    return 301 https://\$host\$request_uri;
}
EOF
```

Test and reload Nginx:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

Certificates are stored in `/etc/letsencrypt/live/examle.com/`.

### 8. Configure Auto-Renewal
Install `cronie` for `crontab` support (not installed by default on Amazon Linux 2):
```bash
sudo yum install -y cronie
```

Verify:
```bash
crontab -v
```

Start and enable `crond`:
```bash
sudo systemctl start crond
sudo systemctl enable crond
```

Set up the cron job (use the Certbot path from `which certbot`, e.g., `/usr/bin/certbot` or `/usr/local/bin/certbot`):
```bash
(crontab -l 2>/dev/null; echo "0 */12 * * * /usr/bin/certbot renew --quiet --post-hook 'systemctl reload nginx'") | crontab -
```

Verify:
```bash
crontab -l
```
Expected output:
```
0 */12 * * * /usr/bin/certbot renew --quiet --post-hook 'systemctl reload nginx'
```

Test renewal:
```bash
sudo /usr/bin/certbot renew --dry-run
```

**Alternative: Systemd Timer** (more reliable):
```bash
cat | sudo tee /etc/systemd/system/certbot-renew.service <<EOF
[Unit]
Description=Certbot Renewal Service

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot renew --quiet --post-hook "systemctl reload nginx"
EOF

cat | sudo tee /etc/systemd/system/certbot-renew.timer <<EOF
[Unit]
Description=Run Certbot Renewal Twice Daily

[Timer]
OnCalendar=*-*-* 00,12:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF
```

Enable and start:
```bash
sudo systemctl enable certbot-renew.timer
sudo systemctl start certbot-renew.timer
```

Verify:
```bash
systemctl list-timers --all
```

### 9. Configure Firewall and Security Group
If `firewalld` is active:
```bash
sudo yum install -y firewalld
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

In the AWS EC2 console, ensure the security group allows:
- HTTP: TCP, port 80, source 0.0.0.0/0
- HTTPS: TCP, port 443, source 0.0.0.0/0

### 10. Verify Setup
- **Test HTTPS**: Open `https://examle.com` in a browser. Confirm a secure connection (green padlock) and NestJS app functionality.
- **Check Certificate**: Verify it’s issued by Let’s Encrypt in browser certificate details.
- **Check Logs**:
  ```bash
  sudo tail -n 20 /var/log/nginx/error.log
  sudo tail -n 20 /var/log/nginx/access.log
  sudo tail -n 20 /var/log/letsencrypt/letsencrypt.log
  ```

## Troubleshooting

- **Nginx Fails to Start**:
  ```bash
  sudo systemctl status nginx
  sudo tail -n 20 /var/log/nginx/error.log
  sudo journalctl -xeu nginx.service_sensor | tail -n 20
  ```
  Fix syntax errors in `/etc/nginx/conf.d/nestjs.conf` if reported by `sudo nginx -t`.

- **Port Conflicts**:
  ```bash
  sudo lsof -i :80
  sudo lsof -i :443
  ```
  Terminate or reconfigure conflicting services.

- **SELinux Issues**:
  Temporarily set permissive mode:
  ```bash
  sudo setenforce 0
  ```
  For production:
  ```bash
  sudo yum install -y policycoreutils-python
  sudo semanage fcontext -a -t cert_t "/etc/letsencrypt(/.*)?"
  sudo restorecon -R /etc/letsencrypt
  sudo semanage port -a -t http_port_t -p tcp 80
  sudo semanage port -a -t http_port_t -p tcp 443
  sudo restorecon -R /etc/nginx
  sudo setenforce 1
  ```

- **DNS Issues**:
  ```bash
  dig examle.com
  ```

- **NestJS Not Reachable**:
  ```bash
  curl http://localhost:3000
  ```
  Adjust `proxy_pass` in `/etc/nginx/conf.d/nestjs.conf` if the port differs.

## Notes
- **NestJS Port**: If your app uses a port other than 3000, update `proxy_pass http://nestjs_app` in `/etc/nginx/conf.d/nestjs.conf`.
- **Certificate Location**: Certificates are in `/etc/letsencrypt/live/examle.com/`.
- **Alternative SSL in NestJS**: For direct SSL termination (not recommended), update `main.ts`:
  ```typescript
  import { NestFactory } from '@nestjs/core';
  import { AppModule } from './app.module';
  import * as fs from 'fs';

  async function bootstrap() {
    const httpsOptions = {
      key: fs.readFileSync('/etc/letsencrypt/live/examle.com/privkey.pem'),
      cert: fs.readFileSync('/etc/letsencrypt/live/examle.com/fullchain.pem'),
    };
    const app = await NestFactory.create(AppModule, { httpsOptions });
    await app.listen(443);
  }
  bootstrap();
  ```
  Nginx is preferred for SSL termination to optimize performance.

## References
- [Certbot Instructions for Amazon Linux](https://certbot.eff.org/instructions?ws=nginx&os=centosrhel)
- [Nginx Reverse Proxy for Node.js](https://www.nginx.com/resources/wiki/start/topics/examples/full/)
- [Let’s Encrypt Documentation](https://letsencrypt.org/docs/)

This setup ensures `https://examle.com` is secure with auto-renewing SSL certificates, replacing manual renewals with a robust, automated solution.
