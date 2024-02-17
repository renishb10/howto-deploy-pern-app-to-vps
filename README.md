# How to Deploy a PERN app to VPS (Ubuntu 22.x)

## Step 1: (Setting up the Server)

1. SSH or Putty login to the server. By default `root` user is available.
   ```
   ssh root@<ip-address>
   ```
   ![Putty Login (Windows)](/assets/img/putty.jpg?raw=true 'Putty login')
1. Create new user & give enough permissions
   ```
   adduser <username>
   ```

## Step N: (Configuring Github Actions)

1. **Create a Runner.**

   Go to the repository -> settings -> Actions -> Runners. Then run the script on the server, via a **Non-Root** user.

   You can find the below script on Runner's instruction itself.

   Download

   ```
   # Create a folder
   $ mkdir actions-runner && cd actions-runner# Download the latest runner package

   $ curl -o actions-runner-linux-x64-2.313.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.313.0/actions-runner-linux-x64-2.313.0.tar.gz# Optional: Validate the hash

   $ echo "sdfd  actions-runner-linux-x64-2.313.0.tar.gz" | shasum -a 256 -c# Extract the installer

   $ tar xzf ./actions-runner-linux-x64-2.313.0.tar.gz
   ```

   Configure

   ```
   # Create the runner and start the configuration experience
   $ ./config.sh --url https://github.com/renishb10/yourrepo-api --token <something># Last step, run it!

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
