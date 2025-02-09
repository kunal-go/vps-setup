# VPS Setup for Ubuntu Server

### 1. Installing Node.JS (v20) and PM2

Install Node.JS and NPM
```
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs
```

Verify Node.JS installation
```
node -v
```

Install PM2 package globally
```
npm install -g pm2
```

### 2. Setup Postgres and create database

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

Create database
```
CREATE DATABASE your_database_name;
```

Use exit command to go back to terminal
```
exit
```


### 3. Setup HTTP server and reverse proxy

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

Create reverse proxy config file for nginx
```
cd /etc/nginx/sites-available
touch <your_domain>
nano <your_domain>
```

Add the following to the location part of the server block
```
server {
    server_name <your_domain>;

    location / {
        proxy_pass http://localhost:<PORT>;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable this configuration file by creating a link from it to the sites-enabled directory that Nginx reads at startup
```
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
```

Check and validate Nginx config syntax
```
sudo nginx -t
```

Restart NGINX
```
sudo nginx -s reload
```


### 4. Add SSL with LetsEncrypt

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


### 5. Setup SFTP

Create SFTP group
```
sudo addgroup sftp
```

Create new sftp user
```
sudo useradd -m sftp_user -g sftp
```

Set the password for the newly created sftp user
```
sudo passwd sftp_user
```

Restrict Access to the User's Home Directory
```
sudo chmod 700 /home/sftp_user/
```

Login through the SFTP using command line
```
sftp sftp_user@127.0.0.1
```


### 5. Setup project directory

Create project directory in sftp space
```
cd /home/sftp_user/
mkdir your_domain
cd your_domain
touch pm2.config.js
nano pm2.config.js
```

Configure pm2.config.js that will be used by PM2
```
module.exports = {
  apps: [
    {
      name: 'Project Name',
      script: './dist/main.js',
      instances: 2,
      exec_mode: 'cluster',
      watch: false
    }
  ]
};
```


### 6. Github Action Script

Start the service with PM2
```
pm2 start pm2.config.js
```

Save PM2 process for when a machine was restart, PM2 can running the same configuration
```
pm2 save
pm2 startup ubuntu
```
