# WordPress Automated Deployment with GitHub Actions

This README file provides a detailed guide to set up and deploy a WordPress website using Nginx, LEMP stack, and GitHub Actions for Continuous Integration and Continuous Deployment (CI/CD). The deployment process ensures security, performance optimization, and automated workflows.

---

## Prerequisites

1. **Azure Virtual Machine (VM):** Provision an Ubuntu 22.04 VM on Azure with public IP access.
2. **GitHub Account:** Create a GitHub repository for version control and workflow automation.
3. **Domain Name:** Register a domain or use a free service like [No-IP](https://www.noip.com/) to generate a temporary hostname.
4. **Secrets Setup:** Store sensitive information like server IP, username, and password as GitHub Secrets.

---

## Deployment Overview

### 1. Server Provisioning

1. **Provision the VM:**

   - Log in to the Azure Portal.
   - Create a new Virtual Machine with the following configurations:
     - **OS:** Ubuntu 22.04 LTS
     - **Size:** Minimum 2 vCPUs, 4 GB RAM
     - **Disk:** 20 GB premium SSD
   - Enable SSH and HTTP/HTTPS ports in the Network Security Group (NSG).

2. **Update and Secure the Server:**

   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo ufw allow OpenSSH
   sudo ufw allow 'Nginx Full'
   sudo ufw enable
   ```

---

### 2. LEMP Stack Installation

1. **Install Nginx:**

   ```bash
   sudo apt install nginx -y
   sudo systemctl start nginx
   sudo systemctl enable nginx
   ```

2. **Install MySQL:**

   ```bash
   sudo apt install mysql-server -y
   sudo mysql_secure_installation
   ```

3. **Install PHP:**

   ```bash
   sudo apt install php-fpm php-mysql -y
   ```

4. **Verify Setup:**

   - Create a test PHP file:
     ```bash
     echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
     ```
   - Visit `http://<your_vm_ip>/info.php` to confirm PHP is working.

---

### 3. WordPress Installation

1. **Download and Configure WordPress:**

   ```bash
   wget https://wordpress.org/latest.tar.gz
   tar -xvf latest.tar.gz
   sudo mv wordpress /var/www/html/
   ```

2. **Set Up the Database:**

   ```bash
   sudo mysql -u root -p
   ```

   Run the following commands in the MySQL shell:

   ```sql
   CREATE DATABASE wordpress_db;
   CREATE USER 'wordpress_user'@'localhost' IDENTIFIED BY 'your_password';
   GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress_user'@'localhost';
   FLUSH PRIVILEGES;
   EXIT;
   ```

3. **Configure WordPress:**

   ```bash
   sudo cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php
   sudo nano /var/www/html/wordpress/wp-config.php
   ```

   Update the database credentials:

   ```php
   define( 'DB_NAME', 'wordpress_db' );
   define( 'DB_USER', 'wordpress_user' );
   define( 'DB_PASSWORD', 'your_password' );
   ```

4. **Set Permissions:**

   ```bash
   sudo chown -R www-data:www-data /var/www/html/wordpress
   sudo chmod -R 755 /var/www/html/wordpress
   ```

---

### 4. Securing the Site with SSL

1. **Install Certbot:**

   ```bash
   sudo apt install certbot python3-certbot-nginx -y
   ```

2. **Obtain SSL Certificate:**

   ```bash
   sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
   ```

3. **Verify Renewal:**

   ```bash
   sudo certbot renew --dry-run
   ```

---

### 5. Nginx Optimization

Edit the Nginx configuration file for WordPress:

```bash
sudo nano /etc/nginx/sites-available/wordpress
```

Add the following:

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    root /var/www/html/wordpress;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}
```

Enable the configuration and reload Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

### 6. GitHub Actions CI/CD Workflow

1. **Repository Setup:**

   - Create a new repository in GitHub.
   - Clone the repository locally:
     ```bash
     git clone <repository_url>
     ```
   - Add your WordPress files to the repository:
     ```bash
     git init
     git add .
     git commit -m "Initial commit"
     git branch -M main
     git remote add origin <repository_url>
     git push -u origin main
     ```

2. **GitHub Actions Workflow:**
   Create `.github/workflows/deploy.yml`:

   ```yaml
   name: Deploy WordPress

   on:
     push:
       branches:
         - main

   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
       - name: Checkout code
         uses: actions/checkout@v3

       - name: Install sshpass
         run: sudo apt-get install -y sshpass

       - name: Deploy to server
         env:
           SERVER_IP: ${{ secrets.SERVER_IP }}
           SERVER_USER: ${{ secrets.SERVER_USER }}
           SERVER_PASSWORD: ${{ secrets.SERVER_PASSWORD }}
         run: |
           sshpass -p $SERVER_PASSWORD ssh -o StrictHostKeyChecking=no $SERVER_USER@$SERVER_IP << 'EOF'
           cd /var/www/html
           git pull origin main
           sudo systemctl reload nginx
           EOF
   ```

3. **GitHub Secrets:**

   - Add the following secrets in your GitHub repository:
     - `SERVER_IP`: Public IP of your VM.
     - `SERVER_USER`: Username for VM access.
     - `SERVER_PASSWORD`: Password for the VM user.

---

### 7. Deployment Steps

1. Push code changes to the `main` branch:

   ```bash
   git add .
   git commit -m "Update WordPress"
   git push origin main
   ```

2. GitHub Actions will automatically deploy changes to the server.

---

### 8. Troubleshooting

1. **Check Nginx Logs:**

   ```bash
   sudo tail -f /var/log/nginx/error.log
   ```

2. **Verify GitHub Actions Logs:**

   - Go to the Actions tab in your GitHub repository.
   - Check logs for any deployment errors.

3. **Common Errors:**

   - Permission denied: Ensure correct file ownership and permissions.
   - Git pull error: Verify GitHub repository URL and access permissions.

---

### 9. Security Best Practices

- Use strong passwords for database and WordPress admin.
- Restrict access to SSH by IP address.
- Regularly update the server and installed packages.
- Enable automatic renewal for SSL certificates.

---

### 10. Additional Features

For bonus points, consider implementing:

- **Caching Plugins:** Install a WordPress caching plugin for improved performance.
- **Backup Strategy:** Set up automated backups for WordPress files and database.
- **Monitoring:** Use tools like Prometheus and Grafana for server and application monitoring.

---

This README serves as a comprehensive guide to set up and deploy a WordPress website using modern DevOps practices. If you encounter any

