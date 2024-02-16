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
