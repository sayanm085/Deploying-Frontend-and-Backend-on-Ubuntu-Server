# Step-by-Step Guide: Deploying Frontend and Backend on Ubuntu Server

## 1. Set Up Your Ubuntu Server

### Update and Secure the Server

Before installing any software, update the package list and upgrade installed packages to ensure your system is up to date and secure:

```sh
sudo apt update && sudo apt upgrade -y
```

Install essential tools that will be required for deployment:

```sh
sudo apt install nginx curl git -y
```

- **nginx**: A web server and reverse proxy to handle HTTP requests.
- **curl**: A command-line tool for transferring data.
- **git**: A version control system for managing your project repositories.

### Install Node.js and PM2 (for backend)

Install Node.js, which is required to run your backend applications, and PM2, which is a process manager for keeping them running smoothly:

```sh
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
```

- **Node.js**: Runtime environment for executing JavaScript on the server.
- **PM2**: Keeps backend applications running even after system restarts.

---

## 2. Deploy the Backend (`api.shotlin.com`)

### Clone Your Backend Repository

Navigate to the web directory and clone your backend repository from GitHub:

```sh
cd /var/www/
git clone https://github.com/your-repo/backend.git
cd backend
npm install
```

- Clones the backend repository into `/var/www/`.
- Installs project dependencies using npm.

### Start the Backend with PM2

Compile and start the backend application:

```sh
npm run build   # If using TypeScript
npm start       # Or use `node index.js`
pm2 start server.js --name backend
pm2 save
```

- Runs the backend application in the background.
- Saves the PM2 process so it restarts automatically after a reboot.

### Configure Nginx as a Reverse Proxy for Backend

To allow external access to your backend, configure Nginx to forward requests:

```sh
sudo nano /etc/nginx/sites-available/api.shotlin.com
```

Add this configuration:

```nginx
server {
  listen 80;
  server_name api.shotlin.com;

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

Enable the configuration and restart Nginx:

```sh
sudo ln -s /etc/nginx/sites-available/api.shotlin.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 3. Deploy the Frontend (`shotlin.com`)

### Build and Serve Static Files

Navigate to the web directory and clone your frontend repository:

```sh
cd /var/www/
git clone https://github.com/your-repo/frontend.git
cd frontend
npm install
npm run build
```

Move the generated build files to the web root directory:

```sh
sudo mv dist/* /var/www/html/
```

### Configure Nginx for Frontend

Create an Nginx configuration file for the frontend:

```sh
sudo nano /etc/nginx/sites-available/shotlin.com
```

Add this configuration:

```nginx
server {
  listen 80;
  server_name shotlin.com;

  root /var/www/html;
  index index.html;

  location / {
    try_files $uri /index.html;
  }
}
```

Enable the configuration and restart Nginx:

```sh
sudo ln -s /etc/nginx/sites-available/shotlin.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 4. Set Up Cloudflare DNS & Security

- Add **A Records** in Cloudflare:
  - `shotlin.com` → Your Server IP
  - `api.shotlin.com` → Your Server IP
- Enable **SSL/TLS Full Mode** for encrypted connections.
- Use **Cloudflare Firewall Rules** to enhance security.

---

## 5. Set Up CI/CD for Auto Deployment

### Generate SSH Key and Add to GitHub/GitLab

Create an SSH key to enable secure deployments from GitHub:

```sh
ssh-keygen -t rsa -b 4096
cat ~/.ssh/id_rsa.pub
```

- Copy and add the SSH key to **GitHub/GitLab Deploy Keys**.

### Create GitHub Actions Workflow (`.github/workflows/deploy.yml`)

Define a GitHub Actions workflow to automate deployments:

```yaml
name: Deploy to Ubuntu Server

on:
  push:
  branches:
    - main

jobs:
  deploy:
  runs-on: ubuntu-latest

  steps:
    - name: SSH into server and deploy
    uses: appleboy/ssh-action@master
    with:
      host: ${{ secrets.SERVER_IP }}
      username: ubuntu
      key: ${{ secrets.SSH_PRIVATE_KEY }}
      script: |
      cd /var/www/backend && git pull && npm install && pm2 restart backend
      cd /var/www/frontend && git pull && npm install && npm run build && sudo mv dist/* /var/www/html/
```

- Automates deployment when code is pushed to the repository.
- Uses **GitHub Secrets** for secure authentication with the server.

---

## 6. Optimize and Secure the Server

### Enable UFW Firewall

Allow only necessary connections:

```sh
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

- Restricts access to only essential services.

### Install Fail2Ban (Prevents Brute Force Attacks)

```sh
sudo apt install fail2ban -y
```

- Protects against repeated login attempts.

### Optimize Nginx

Improve performance by adjusting Nginx settings:

```sh
sudo nano /etc/nginx/nginx.conf
```

Modify:

```nginx
worker_processes auto;
client_max_body_size 50M;
gzip on;
gzip_types text/plain text/css application/json application/javascript;
```

- **`worker_processes auto;`**: Adjusts workers dynamically based on CPU cores.
- **`gzip on;`**: Enables compression to reduce file sizes and speed up loading.

Restart Nginx:

```sh
sudo systemctl restart nginx
```

---

## ✅ Final Thoughts

Now, your frontend and backend are **live**, with **CI/CD, security, and performance optimizations**.

### 🚀 Next Steps:

- Add **HTTPS (SSL Certificates)** for secure connections.
- Set up **Monitoring Tools** to track server health.
- Automate **Database Backups** to prevent data loss.

Let me know if you need improvements! 🚀
