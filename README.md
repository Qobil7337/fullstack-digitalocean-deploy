
# Project stack: Angular (frontend), NestJS (backend), MySQL (DigitalOcean managed), Nginx, Adminer, domain, SSL

## First create two droplets and one managed database

## Then connect godaddy domain to digital ocean

### Step 1: Go to networking tab in digital ocean
- Click "Create" under Domains.
- Enter your domain name (e.g., yourdomain.com) and click "Add Domain".
- Copy the nameservers from DigitalOcean
- Once added, you’ll see something like:
  ```
  ns1.digitalocean.com  
  ns2.digitalocean.com  
  ns3.digitalocean.com  
  ```
- Go to GoDaddy DNS settings
- Go to “My Products” → click on your domain.
- Click “Manage DNS”.
- Scroll down to “Nameservers”.
- Click Change, then Enter my own nameservers.
- Paste the 3 DigitalOcean nameservers.
- Save and wait up to 30 minutes (DNS propagation time).

---

### part 2: SET UP SUBDOMAINS IN DIGITALOCEAN DNS
In the domain's DNS section in DigitalOcean:  
Create two A records:
- Type: A
- Name: api       →  your droplet IP
- Name: adminer   →  your droplet IP

You can now use:
- `api.yourdomain.com` → NestJS API
- `adminer.yourdomain.com` → Adminer

---

## Step 1: Initial Setup on Droplet

```bash
apt update && apt upgrade -y
apt install curl wget git ufw unzip -y
```

---

## Step 2: Install Node.js, PM2, Nginx

Install Node.js:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
```

Install PM2 (to run NestJS in background):

```bash
npm install -g pm2
```

Install Nginx:

```bash
apt install nginx -y
```

Enable Nginx:

```bash
systemctl enable nginx
systemctl start nginx
```

---

## Step 3: Set Up UFW Firewall

```bash
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw enable
```

---

## Step 4: Clone & Start NestJS App

```bash
cd /var/www
git clone https://github.com/yourusername/your-nestjs-repo.git api
cd api
npm install
```

Configure `.env` and other environment-specific files.  
Build the api:

```bash
npm run build
```

Start the app with PM2:

```bash
pm2 start dist/main.js --name nest-api
pm2 save
pm2 startup
```

Make sure your app listens on `localhost:3000` or change if needed.

---

## Step 5: Install Adminer

```bash
mkdir -p /var/www/adminer
cd /var/www/adminer
wget https://github.com/vrana/adminer/releases/download/v4.8.1/adminer-4.8.1.php -O index.php
```

---

## Step 6: Configure Nginx Server Blocks

### 6.1 Create config for api.saidoon.com

```bash
nano /etc/nginx/sites-available/api.saidoon.com
```

Paste:

```nginx
server {
    listen 80;
    server_name api.saidoon.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### 6.2 Create config for adminer.saidoon.com

```bash
nano /etc/nginx/sites-available/adminer.saidoon.com
```

Paste:

```nginx
server {
    listen 80;
    server_name adminer.saidoon.com;

    root /var/www/adminer;
    index index.php;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;  # Adjust PHP version if needed
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

## Step 7: Enable Sites

```bash
ln -s /etc/nginx/sites-available/api.saidoon.com /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/adminer.saidoon.com /etc/nginx/sites-enabled/
```

Test Nginx config:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

If you're using Nginx as your main web server (which is common), you do not need Apache. To disable Apache:

```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
```

---

## Step 8: Install PHP for Adminer

```bash
apt install php php-fpm -y
```

Install the MySQL extension for PHP 8.3:

```bash
sudo apt install php8.3-mysql
```

Make sure PHP-FPM is running:

```bash
sudo systemctl enable php8.3-fpm
sudo systemctl start php8.3-fpm
```

And enable it to run on boot:

```bash
sudo systemctl enable php8.3-fpm
systemctl status php8.3-fpm
```

---

## Step 9: Add SSL with Let's Encrypt

Install Certbot:

```bash
apt install certbot python3-certbot-nginx -y
```

Run for both domains:

```bash
certbot --nginx -d api.saidoon.com -d adminer.saidoon.com
```

Auto-renew:

```bash
echo "0 0 * * * /usr/bin/certbot renew --quiet" >> /etc/crontab
```
