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

1. Firewall Setup - Install UFW (Uncomplicated Firewall)
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
     sudo -u postgres psql

     # create DB
     CREATE DATABASE <EXAMPLE_DB>;

     # create user
     CREATE USER <EXAMPLE_USER> WITH ENCRYPTED PASSWORD <YOUR PASSWORD>;


     # connect to database "<EXAMPLE_DB>" as user "postgres".
     \c <EXAMPLE_DB> postgres

     # Grant privileges
     GRANT ALL PRIVILEGES ON DATABASE <EXAMPLE_DB> TO <EXAMPLE_USER>;
     GRANT ALL ON SCHEMA public TO <EXAMPLE_USER>;
     GRANT USAGE ON SCHEMA public TO <EXAMPLE_USER>;
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

## Step 3: (Configure DNS - Domain)

1. Go to your domain provider website > DNS settings
1. Create a new **A Record** type with following details

   - `Host`: **@** and **www** for direct domain or go for subdomain eg: **api**
   - `Value/Data`: Your server's **IP** address
   - `TTL`: Default or `600 seconds` or `30 mins`

     ![DNS A Record (Windows)](/assets/img/dns_config.png?raw=true 'DNS A Record')

#### Install SSL Certificate

Install Let's Encrypt SSL (Free one), using `CertBot` script. Run the below commands for Ubuntu 20.x server. Visit https://certbot.eff.org/ for more details.

```
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```

Check your domain, it should auto redirect to HTTPS

## Step 4: (Configuring Github Actions)

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

   - Sample Workflow:

     ```
         name: Deploy Fiveto9Jobs API

         on:
             push:
                 branches: ['main']
             pull_request:
                 branches: ['main']

         jobs:
             build:
                 runs-on: self-hosted

                 steps:
                     - uses: actions/checkout@v3

                     - name: Use Node.js '20'
                     uses: actions/setup-node@v3
                     with:
                         node-version: '20'
                         cache: 'npm'

                     - name: Clean Previous Installations
                     run: |
                         rm -rf node_modules
                         rm -rf .prisma

                     - name: Install Dependencies
                     run: npm ci

                     - name: Install pm2
                     run: npm install -g pm2

                     - name: Set Environment Variables
                     run: |
                         touch .env
                         echo "${{secrets.GITHUB_SECRETS}}" > .env

                     - name: Run Prisma Migrations
                     run: npx prisma migrate deploy

                     - name: Stop pm2 Process (if running)
                     run: |
                     pm2 stop <PROCESS NAME> || true

                     - name: Start pm2 Process
                     run: |
                         pm2 start ecosystem.config.js --env production
                         pm2 save

                     - name: Restart Nginx
                     run: sudo service nginx restart
     ```

   - Sample `ecosystem.config.js`

     ```
     module.exports = {
         apps: [
             {
                 name: <PROCESS NAME>,
                 script: 'npm',
                 args: 'start',
                 cwd: '/home/user/apps/actions-runner/_work/project/project',
                 env: {
                     NODE_ENV: 'production',
                 },
             },
         ],
     };

     ```

## Step 5: Monitoring: (Install Prometheus and Grafana)

Let us install Prometheus and Grafana for monitoring our VPS server

#### Install Prometheus

- Go to [Prometheus website](https://prometheus.io/download/) and get the latest prometheus release download link.

  ```sh
  # download and unzip
  wget <download link>
  tar xvfz prometheus-*.tar.gz

  # move it to /etc/prometheus
  sudo mv prometheus-x.x.x.linux-amd64 /etc/prometheus

  # delete the download if you want
  rm prometheus-x.x.x.linux-amd64.tar.gz
  ```

- Create a Prometheus service

  ```sh
  vi /etc/systemd/system/prometheus.service
  ```

  And paste the below code

  ```
  [Unit]
  Description=Prometheus
  Wants=network-online.target
  After=network-online.target

  [Service]
  ExecStart=/etc/prometheus/prometheus --config.file=/etc/prometheus/prometheus.yml
  Restart=always

  [Install]
  WantedBy=multi-user.target
  ```

  Start and Enable the Prometheus Service

  ```sh
  sudo systemctl daemon-reload
  sudo systemctl start prometheus
  sudo systemctl enable prometheus

  sudo systemctl status prometheus
  ```

- Adjust firewall and visit Prometheus CPanel

  ```sh
  sudo ufw allow 9090/tcp
  sudo ufw status
  ```

  then visit -> **ServerIP:9090** you should see

  ![Prometheus CPanel](/assets/img/prometheus_cpanel.png?raw=true 'Prometheus CPanel')

#### Setup Node Exporter

- Go to [Prometheus website](https://prometheus.io/download/) and but this time get the latest Node Exporter download link.

  ```sh
  # download and unzip
  wget <download node_exporter link>
  tar xvfz node_exporter*.tar.gz

  # move it to /etc/node_exporter
  sudo mv node_exporter-x.x.x.linux-amd64 /etc/node_exporter

  # delete the download if you want
  rm node_exporter-x.x.x.linux-amd64.tar.gz
  ```

- Create a Node Exporter service

  ```sh
  vi /etc/systemd/system/node_exporter.service
  ```

  And paste the below code

  ```
  [Unit]
  Description=Node Exporter
  Wants=network-online.target
  After=network-online.target

  [Service]
  ExecStart=/etc/node_exporter/node_exporter
  Restart=always

  [Install]
  WantedBy=multi-user.target
  ```

  Start and Enable the Node Exporter Service

  ```sh
  sudo systemctl daemon-reload
  sudo systemctl start node_exporter
  sudo systemctl enable node_exporter

  sudo systemctl status node_exporter
  ```

- Configure Prometheus

  ```sh
  sudo vi /etc/prometheus/prometheus.yml
  ```

  Replace your `prometheus.yml` with below code

  ```yml
  global:
  scrape_interval: 15s

  scrape_configs:
    - job_name: node
      static_configs:
        - targets: ['<Your Server IP>:9100']
  ```

  Restart Prometheus service

  ```
  sudo systemctl restart prometheus
  sudo systemctl status prometheus
  ```

- Check Prometheus CPanel for a new Node Exporter **Target**

  Go to **Server IP:9090** > Status > Targets

  ![Prometheus Targets](/assets/img/prometheus_targets.png?raw=true 'Prometheus Targets')

  We should see new node exporter target metric like above

#### Grafana Setup

- Install Grafana

  Go to [Grafana Website](https://grafana.com/grafana/download) and find the latest version and installation steps

  ```sh
  sudo apt-get install -y adduser libfontconfig1 musl

  wget https://dl.grafana.com/enterprise/release/grafana-enterprise_x.x.x_amd64.deb

  sudo dpkg -i grafana-enterprise_x.x.x_amd64.deb
  ```

  ```sh
  # remove downloaded file
  rm grafana-enterprise_x.x.x_amd64.deb
  ```

- Start and Enable Grafana Server service

  ```sh
  sudo systemctl daemon-reload
  sudo systemctl start grafana-server
  sudo systemctl enable grafana-server

  sudo systemctl status grafana-server
  ```

- Adjust firewall

  ```sh
  sudo ufw allow 3000/tcp
  sudo ufw status
  ```

- Grafana Dashboard Setup

  Go to **ServerIP:3000** to see the Grafana Dashboard.

  - Login with user **admin** and password **admin** for the first time.
  - It prompts for new password, please go for it.

  You should see

  ![Grafana Welcome Page](assets/img/grafana_launch.png 'Prometheus Targets')

- Add Prometheus as Data source

  - Go to Grafana dashboard and click on **DATA SOURCES** then click on **Prometheus**

  You should see

  ![Grafana Data Source](assets/img/grafana_datasource.png 'Grafana Data Source')

  Add your **ServerIP:9090** in the Connection > Prometheus Server URL input as like above.

- Import Grafana Node Exporter Dashboard

  - Go to [Grafana Labs Dashboard](https://grafana.com/grafana/dashboards/) and find the Node Exporter Full (Prometheus) dashboard and copy its ID. (**1860**)
  - Now go to Local Grafana Dashboard **ServerIP:9090** and go to the Dashboard section, click on **New** and click on **Import**, then paste the ID (1860), we just copied and create dashboard.

    ![Grafana Import Dashboard](assets/img/grafana_import_dashboard.png 'Grafana Import Dashboard')

- Here we go! Node Exporter Dashboard is ready!

  ![Grafana Node Exporter Dashboard](assets/img/grafana_node_dashboard.png 'Grafana Node Exporter Dashboard')
