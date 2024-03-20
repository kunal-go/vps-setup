# VPS Setup for Ubuntu Server

### 1. Installing Node.JS and clone the project source code

Install Node.JS and NPM
```
curl -sL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs
```

Verify Node.JS installation
```
node --version
```

Install PM2 package globally
```
npm install -g pm2
```

Clone project from git
```
git clone <git_repo_url>
```

### 2. Setup PostgreSQL database

Install PostgreSQL service
```
sudo apt install postgresql
```

At the end of the installation process, OS starts postgresql service.<br />
We can check with the systemd init system to make sure the service is running:
```
systemctl status postgresql
```

Now that we can connect to our PostgreSQL server, the next step is to set a password for the postgres user.<br />
Run the following command at a terminal prompt to connect to the default PostgreSQL template database:
```
sudo -u postgres psql template1
```

Once you connect to the PostgreSQL server, you will be at an SQL prompt. <br />
You can run the following SQL command at the psql prompt to configure the password for the user postgres:
```
ALTER USER postgres with encrypted password 'your_password';
```

### 3. Build and serve the service

Install project dependencies
```
npm install
```

Create server.config.js at the root of the project folder.
```
module.exports = {
  apps: [
    {
      name: 'My Awesome App',
      script: './dist/main.js',
      instances: 0,
      exec_mode: 'cluster',
      watch: false
    }
  ]
};
```

Set all the environment variables in .env file.
Build the project before starting service (for Nest.JS)
```
npm run build
npm run migration:run
```

Start the service with PM2
```
pm2 start server.config.js
```

Save PM2 process for when a machine wasrestart, PM2 can running the same configuration
```
pm2 save
pm2 startup ubuntu
```


### 4. Setup HTTP server and reverse proxy

Install Nginx
```
sudo apt update
sudo apt install nginx
```

Update firewall to open port 80 (http) and 443 (https)
```
sudo ufw allow 'OpenSSH'
sudo ufw allow 'Nginx HTTP'
sudo ufw allow 'Nginx HTTPS'
```

Apply firewall changes and enable it
```
sudo ufw enable
```

Check the firewall status
```
sudo ufw status
```

At the end of the installation process, Ubuntu 20.04 starts Nginx. The web server should already be up and running.<br />
We can check with the systemd init system to make sure the service is running by typing:
```
systemctl status nginx
```

When you have your server’s IP address, enter it into your browser’s address bar:
```
http://<your_server_ip>
```
You should see the default Nginx landing page <br />


Open server's default config file
```
nano /etc/nginx/sites-available/default
```

Add the following to the location part of the server block
```
server {
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://localhost:8001; #whatever port your app runs on
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
```

Check and validate Nginx config syntax
```
sudo nginx -t
```

Restart NGINX
```
sudo nginx -s reload
```


### 5. Add SSL with LetsEncrypt

Install certbot
```
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Only valid for 90 days, test the renewal process with
```
certbot renew --dry-run
```
