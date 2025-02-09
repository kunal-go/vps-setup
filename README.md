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

Install PM2 & Yarn packages globally
```
npm install -g pm2
npm install -g yarn
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

Install Unzip
```
sudo apt-get install zip unzip
```

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

Create deployment space for uploading files
```
cd /home/sftp_user/
mkdir deployments
cd deployments
mkdir your_domain
```


### 6. Create SSH Private Key for Github Actions Secret

Generate SSH Keys
```
ssh-keygen -t rsa -b 4096 -m PEM -C "github-actions-node1"
```

Copy Public Key to authorized key
```
cat id_rsa.pub

# Copy the output to
nano ~/.ssh/authorized_keys
```

Copy Private Key
```
# Copy the output
cat ~/.ssh/id_rsa
```

### 7. Github Action Script
```
    
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      deployments: write
    name: Deploy Service to VPS
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Build
        run: yarn install && yarn build

      - name: Create .env file
        env:
          ENV_1: ${{ secrets.ENV_1 }}
          ENV_2: ${{ secrets.ENV_2 }}
        run: |
          touch .env
          echo "NODE_ENV=production" >> .env
          echo "PORT=<YOUR_PORT>" >> .env
          echo "ENV_1=$ENV_1" >> .env
          echo "ENV_2=$ENV_2" >> .env
          echo "TZ=Asia/Kolkata" >> .env
          cat .env

      - name: Prepare zip file
        run: |
          cd server
          zip -r new_deployment.zip . -x ".git/*" ".github/*" "node_modules/*" "src/*"
          mkdir deployments
          mv new_deployment.zip deployments/new_deployment.zip

      - name: Deploy to VPS
        uses: wlixcc/SFTP-Deploy-Action@v1.2.4
        env:
          FTP_USERNAME: ${{ secrets.FTP_USERNAME }}
        with:
          username: ${{ secrets.FTP_USERNAME }}
          server: ${{ secrets.FTP_HOST }}
          port: ${{ secrets.FTP_PORT }}
          local_path: ./deployments/*
          remote_path: deployments/your_domain
          sftp_only: true
          password: ${{ secrets.FTP_PASSWORD }}

      - name: Restart the server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: 22
          script: |
            cd /home/sftp_user/deployments/your_domain
            unzip new_deployment.zip -d new_deployment
            rm new_deployment.zip
            cd ../..
            rsync -av deployments/your_domain/new_deployment/* your_domain/
            rm -rf deployments/your_domain/new_deployment
            cd your_domain
            pm2 stop your_domain
            yarn install --production --immutable --immutable-cache --check-cache
            yarn migration:run
            pm2 start pm2.config.js
            pm2 save
            pm2 startup ubuntu

```
