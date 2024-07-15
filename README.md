# How to Deploy a PERN app to VPS (Ubuntu 22.x)

## Step 1: (Setting up the Server)

1. SSH or Putty login to the server. By default `root` user is available.

   ```
   ssh root@<ip-address>
   ```

   ![Putty Login (Windows)](/assets/img/putty.jpg?raw=true 'Putty login')

1. Update the System
   ```
   apt update && apt upgrade -y
   ```
1. Create new user & give enough permissions
   ```
   adduser <username>
   usermod -aG sudo <username>
   ```
1. Secure SSH Access
   - Edit the SSH configuration file:
     ```sh
     nano /etc/ssh/sshd_config
     ```
   - Disable root login and change the SSH port:
     ```sh
     PermitRootLogin no
     Port 2222
     ```
   - Restart SSH service:
     ```sh
     systemctl restart ssh
     ```
   - Setup SSH keys for the new user (on your **Local machine**):
     ```sh
     ssh-keygen -t rsa
     ssh-copy-id -p 2222 <username>@<your_server_ip>
     ```
     or manually copy the ssh public key on to the server's `~/.ssh/authorized_keys`
1. Firewall Setup
   Install UFW (Uncomplicated Firewall)
   ```sh
   apt install ufw
   ufw allow 2222/tcp
   ufw allow http
   ufw allow https
   ufw enable
   ```
   After allowing `2222/tcp` and enabling `ufw`, the default ssh port `22` is disabled implicitly. If there is an issue connecting to port `2222/tcp` try
   ```
   sudo ufw reload
   sudo systemctl restart ssh
   ```
   If nothing works out, try rebooting the VPS from provider's dashboard.

## Step 2: (Install & Setup Essential Softwares)

1. **Install Git**

   Be as new user, because when you create runner we use non-root user and where the runner needs access to node & pm2 build the app.

   ```sh
   apt install git
   ```

1. **Setup Postgres**

   ```sh
   apt install postgresql postgresql-contrib
   ```

   - secure Postgresql:

   ```sh
   sudo -u postgres psql
   \password postgres
   \q
   ```

   - Create a new database and user:

   ```sh
   sudo -u postgres createuser newuser
   sudo -u postgres createdb newdb
   sudo -u postgres psql
   ALTER USER newuser WITH PASSWORD 'password';
   GRANT ALL PRIVILEGES ON DATABASE newdb TO newuser;
   \q
   ```

1. **Install Node.js**

   ```sh
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
   nvm install 20
   ```

1. **Install PM2**

   ```sh
   npm i pm2 -g
   ```

1. **Setup Nginx Reverse Proxy**

   ```sh
   apt install nginx
   ```

   For multiple site configuration, let us leave the `/etc/nginx/sites-available/default` server block as it is and let us create new server block files for each websites/apps we planned to host in this machine.

   - Create a new server block for your app. Name it as per your domain name.

     ```sh
     vi /etc/nginx/sites-available/api.example.com
     ```

   - Add the following content

     ```nginx
     server {
             listen 80;
             server_name api.example.com;

             location / {
                 proxy_pass http://localhost:3001; # Replace with your Express API port
                 proxy_http_version 1.1;
                 proxy_set_header Upgrade $http_upgrade;
                 proxy_set_header Connection 'upgrade';
                 proxy_set_header Host $host;
                 proxy_cache_bypass $http_upgrade;
             }
         }
     ```

   - Create symbolic links to enable the configuration
     ```sh
     sudo ln -s /etc/nginx/sites-available/api.example.com /etc/nginx/sites-enabled/
     ```
   - Test the configuration
     ```sh
     sudo nginx -t
     ```
   - Restart Nginx
     ```sh
     sudo systemctl restart nginx
     ```

## Step N: (Configure DNS - Domain)

1. Go to your domain provider website > DNS settings
1. Create a new **A Record** type with following details

   - `Host`: **@** and **www** for direct domain or go for subdomain eg: **api**
   - `Value/Data`: Your server's **IP** address
   - `TTL`: Default or `600 seconds` or `30 mins`

     ![DNS A Record (Windows)](/assets/img/dns_config.png?raw=true 'DNS A Record')

## Step N: (Configuring Github Actions)

1. **Create a Runner.**

   Go to the repository -> settings -> Actions -> Runners. Then run the script on the server, via a **Non-Root** user.

   You can find the below script on Runner's instruction itself.

   Download

   ```
    # Create a folder
    $ mkdir actions-runner && cd actions-runner

    # Download the latest runner package
    $ curl -o actions-runner-linux-x64-2.317.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.317.0/actions-runner-linux-x64-2.317.0.tar.gz

    # Optional: Validate the hash
    $ echo "9e883d210df8c6028aff475475a457d380353f9d01877d51cc01a17b2a91161d  actions-runner-linux-x64-2.317.0.tar.gz" | shasum -a 256 -c

    # Extract the installer
    $ tar xzf ./actions-runner-linux-x64-2.317.0.tar.gz
   ```

   Configure

   ```
    # Create the runner and start the configuration experience
    $ ./config.sh --url https://github.com/renishb10/fiveto9jobs-api --token <Get the Token from Github Settings/Runners>

    # Last step, run it!
    $ ./run.sh
   ```

   Using your self-hosted runner

   ```
   # Use this YAML in your workflow file for each job
   runs-on: self-hosted
   ```

   To run the `runner` as a service

   ```
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```

1. **Create Github Workflow**

   Go to Actions -> Continous Integration -> Node.js (template) -> Configure

   Sample Action:

   ```
       # This workflow will do a clean installation of node dependencies, cache/restore them, build the source code and run tests across different versions of node
       # For more information see: https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-nodejs

       name: Deploy Fiveto9Jobs API

       on:
       push:
           branches: [ "main" ]
       pull_request:
           branches: [ "main" ]

       jobs:
       build:

           runs-on: self-hosted

           steps:
           - uses: actions/checkout@v3
           - name: Use Node.js ${{ matrix.node-version }}
               uses: actions/setup-node@v3
               with:
               node-version: ${{ matrix.node-version }}
               cache: "npm"

           - run: npm ci
           - run: |
               touch .env
               echo "${{secrets.ENV_SECRET_FILE}}" > .env
           - run: |
               pm2 stop <process-name>
               pm2 start <process-name>
               pm2 save
               sudo service nginx restart
   ```

## Step N: (Setting up Postgresql)

```
CREATE DATABASE EXAMPLE_DB;
CREATE USER EXAMPLE_USER WITH ENCRYPTED PASSWORD 'Sup3rS3cret';
GRANT ALL PRIVILEGES ON DATABASE EXAMPLE_DB TO EXAMPLE_USER;
\c EXAMPLE_DB postgres
# You are now connected to database "EXAMPLE_DB" as user "postgres".
GRANT ALL ON SCHEMA public TO EXAMPLE_USER;
GRANT USAGE ON SCHEMA public TO EXAMPLE_USER;
```

## Step N: (Install SSL Certificate)

Install Let's Encrypt SSL (Free one), using `CertBot` script. Run the below commands for Ubuntu 20.x server. Visit https://certbot.eff.org/ for more details.

```
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```

Check your domain, it should auto redirect to HTTPS
