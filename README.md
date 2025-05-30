
# 🌐 Full Stack Deployment Guide: Angular + NestJS + MySQL (DigitalOcean)

## ✅ Stack
- Frontend: Angular
- Backend: NestJS
- Database: DigitalOcean Managed MySQL
- Web Server: Nginx
- Domain: GoDaddy (pointed to DigitalOcean)
- Admin Panel: Adminer (via subdomain)
- SSL: Let's Encrypt (Certbot)
- PM2: Node app process manager

---

## 📦 Infrastructure Setup

### 1. Create Resources on DigitalOcean
- 2 Droplets (1 for API/Adminer, 1 for Frontend)
- 1 Managed MySQL Database

### 2. Connect GoDaddy Domain to DigitalOcean
1. In DigitalOcean → Networking → Domains → Add your domain (e.g., `saidoon.com`)
2. Copy nameservers:
    ```
    ns1.digitalocean.com
    ns2.digitalocean.com
    ns3.digitalocean.com
    ```
3. In GoDaddy → My Products → DNS Settings → Change Nameservers → Enter custom → Paste DO nameservers
4. Wait ~30 mins for DNS propagation

### 3. Create Subdomain A Records in DigitalOcean
- Type: A
- Name: `api` → points to droplet IP
- Name: `adminer` → points to droplet IP

---

## 🚀 Droplet Setup

### 1. Initial Setup
```bash
apt update && apt upgrade -y
apt install curl wget git ufw unzip -y
```

### 2. Install Node.js, PM2, Nginx
```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
npm install -g pm2
apt install nginx -y
systemctl enable nginx
systemctl start nginx
```

### 3. Configure UFW Firewall
```bash
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw enable
```

---

## ⚙️ Deploy NestJS API

```bash
cd /var/www
git clone https://github.com/yourusername/your-nestjs-repo.git api
cd api
npm install
```

- Set up `.env` file
- Build and run:

```bash
npm run build
pm2 start dist/main.js --name nest-api
pm2 save
pm2 startup
```

Make sure your app listens on `localhost:3000`.

---

## 🛠️ Setup Adminer

```bash
mkdir -p /var/www/adminer
cd /var/www/adminer
wget https://github.com/vrana/adminer/releases/download/v4.8.1/adminer-4.8.1.php -O index.php
```

### Install PHP & Extensions

```bash
apt install php php-fpm -y
apt install php8.3-mysql
systemctl enable php8.3-fpm
systemctl start php8.3-fpm
```
After php installation you can see that apache is also installed and serving your droplet, you can check it using 
```bash
apache2 -v
```

# Remove Apache and prevent it from reinstalling
```bash
sudo systemctl stop apache2
sudo systemctl disable apache2
sudo apt remove apache2*
sudo apt purge apache2*
sudo apt autoremove
```

---

## 🌐 Configure Nginx

### `api.saidoon.com`

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

### `adminer.saidoon.com`

```nginx
server {
    listen 80;
    server_name adminer.saidoon.com;

    root /var/www/adminer;
    index index.php;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable both sites:

```bash
ln -s /etc/nginx/sites-available/api.saidoon.com /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/adminer.saidoon.com /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

---

## 🔒 Add SSL with Let's Encrypt

Install Certbot:

```bash
apt install certbot python3-certbot-nginx -y
```

Get certificates:

```bash
certbot --nginx -d api.saidoon.com -d adminer.saidoon.com
```

Auto-renewal:

```bash
echo "0 0 * * * /usr/bin/certbot renew --quiet" >> /etc/crontab
```

---

✅ Done! Your NestJS backend and Adminer should be securely available at:
- `https://api.saidoon.com`
- `https://adminer.saidoon.com`
