
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

## 📦 Infrastructure Setup For Api and Adminer

### 1. Create Resources on DigitalOcean
- 2 Droplets (1 for API/Adminer, 1 for Frontend)
- 1 Managed MySQL Database

### 2. Connect GoDaddy Domain to DigitalOcean
1. In DigitalOcean → Networking → Domains → Add your domain (e.g., `yourdomain.com`)
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

### 📄 Set up `.env` file

Create a `.env` file in the root of your project:

```bash
nano .env
```

Paste the following content:

```env
# Server
PORT=3000

# Database
DB_HOST=localhost
DB_PORT=3306
DB_USERNAME=root
DB_PASSWORD=your_password
DB_NAME=your_database

# JWT
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRES_IN=3600s

# Other
NODE_ENV=production
API_PREFIX=api
```

Save and exit (`CTRL + O`, `ENTER`, then `CTRL + X`).

---

### 🛠️ Build and run the app

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

### Create config for api.yourdomain.com

```bash
nano /etc/nginx/sites-available/api.yourdomain.com
```

Paste:

```nginx
server {
    listen 80;
    server_name api.yourdomain.com;

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

### `adminer.yourdomain.com`

```bash
nano /etc/nginx/sites-available/adminer.yourdomain.com
```

Paste:

```nginx
server {
    listen 80;
    server_name adminer.yourdomain.com;

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
ln -s /etc/nginx/sites-available/api.yourdomain.com /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/adminer.yourdomain.com /etc/nginx/sites-enabled/
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
certbot --nginx -d api.yourdomain.com -d adminer.yourdomain.com
```

Auto-renewal:

```bash
echo "0 0 * * * /usr/bin/certbot renew --quiet" >> /etc/crontab
```

---

✅ Done! Your NestJS backend and Adminer should be securely available at:
- `https://api.yourdomain.com`
- `https://adminer.yourdomain.com`

---



## 📦 Infrastructure Setup For App

## 🌐 DNS Configuration for Angular Frontend Deployment

To serve your Angular frontend using a custom domain like `https://yourdomain.com`, follow these steps to set up DNS records on DigitalOcean.

---

## 🛠 Prerequisites

- Your domain is already **pointed to DigitalOcean** via nameservers from GoDaddy or another registrar.
- You have a **frontend droplet** ready and know its public IP.

---

## 📌 Steps to Add DNS Records

1. **Go to DigitalOcean Control Panel**
  - Navigate to `Networking` → `Domains`
  - Select your domain: `yourdomain.com`

2. **Add A Records**

| Type | Hostname | Value (Droplet IP) | Description                         |
|------|----------|---------------------|-------------------------------------|
| A    | `@`      | `123.123.123.123`   | Root domain → your frontend droplet |
| A    | `www`    | `123.123.123.123`   | www subdomain → same droplet        |

> Replace `123.123.123.123` with your actual droplet IP address.

---

## ✅ Verify DNS

You can check if DNS propagation is complete:

```bash
ping yourdomain.com
ping www.yourdomain.com
```

Or use an online DNS checker like [https://dnschecker.org](https://dnschecker.org)

Once your domain resolves to the correct IP, you're ready to move on.


### 1. Initial Setup
```bash
apt update && apt upgrade -y
apt install curl wget git ufw unzip -y
```

### 2. Install Node.js & Angular CLI
```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
npm install -g @angular/cli
```

### 3. Install Nginx
```bash
apt install nginx -y
systemctl enable nginx
systemctl start nginx
```

### 4. Configure UFW Firewall
```bash
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw enable
```

### 5. Clone Your Angular Project
```bash
cd /var/www
git clone https://github.com/yourusername/your-angular-repo.git angular-app
cd angular-app
npm install
```

### 6. Build Angular App
```bash
ng build
```

### 7. Configure Nginx to Serve Angular
Create config file:
```bash
nano /etc/nginx/sites-available/yourdomain.com
```

Paste:
```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    root /var/www/angular-app/dist/your-app-name;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

Enable config and reload:
```bash
ln -s /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

## 🔒 Add SSL with Let's Encrypt

Install Certbot:

```bash
apt install certbot python3-certbot-nginx -y
```

Get certificates:

```bash
certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Auto-renewal:

```bash
echo "0 0 * * * /usr/bin/certbot renew --quiet" >> /etc/crontab
```

✅ Done!
Your Angular app is now live and secure at:

🌍 https://yourdomain.com



CI CD...
