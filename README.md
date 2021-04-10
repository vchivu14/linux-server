# Linux Server Configuratio

## Project Overview:

In this final project we will learn how to set up Linux distribution on a virtual machine and install and configure a web server to host our application.

## Project Description

This application we're building will allow authenticated users to post a CAUSE they believe needs attention from the public authorities and then let users come and respond with possible SOLUTIONS. Users will be the only ones in charge of their content with the possibility to update and delete their posts. The website is open for reading to anyone but you must login to post a cause or respond to one.
In this project we will use:

- The Linux distribution [Ubuntu](https://www.ubuntu.com/download/server) 16.04.
- The virtual private server [Amazon Lighsail](https://lightsail.aws.amazon.com/).
- The web application that can be found here: [Threads project](https://github.com/vchivu14/threads).
- [Vagrant](https://www.vagrantup.com/), [VirtualBox](https://www.virtualbox.org/) and [Git](https://git-scm.com/).
- My local machine that is using Microsoft Windows 10.

## Setting up the project

### Step 1: Start a new Ubuntu Linux server instance on Amazon Lightsail

- Login into [Amazon Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home/resources) using an Amazon Web Services account.
- Look for the `Create instance` button.
- Choose the `Linux/Unix` platform, `OS Only` and `Ubuntu 16.04 LTS`.
- Choose a instance plan the cheapest will work which you can unsubscribe after you're done with the project.
- Write a name for your instance.
- Click `Create`.
- Wait for the instance to start up.

**Reference**
Udacity, [Web Application Servers](https://classroom.udacity.com/nanodegrees/nd004/parts/b2de4bd4-ef07-45b1-9f49-0e51e8f1336e/modules/56cf3482-b006-455c-8acd-26b37b6458d2/lessons/4340119836/concepts/48089485480923).

### Step 2: SSH into the server

- From the `Account` menu on Amazon Lightsail, click on `SSH keys` tab and download the Default Private Key.
- Move the private key file into the folder `~/.ssh` and rename it as you wish, I renamed it `grader`.
- In your terminal, type these commands: 
* `chmod 700 ~/.ssh/grader`
* `chmod 644 /home/grader/.ssh/authorized_keys`
- To connect to the instance from the terminal: `ssh -i ~/.ssh/(your key) -p 22 ubuntu@(your public IP address)`, in my case `ssh -i ~/.ssh/grader.pem ubuntu@34.215.80.168`.

### Step 3: Update and upgrade installed packages

```
sudo apt-get update
sudo apt-get upgrade
```

### Step 4: Install the dependencies

#### Install python and pip

```
sudo apt-get install python-minimal
sudo apt install python-pip
```

#### Setup virtualenv

```
pip install virtualenv
virtualenv venv
source venv/bin/activate
```

#### Install nginx

```
sudo apt-get install nginx
sudo ufw allow 'Nginx HTTP'
```

#### Nginx configuration

```
pip install gunicorn
```
- Navigate to `/home/ubuntu/` directory.
- And then `git clone https://github.com/vchivu14/threads`.
- Nginx + wsgi configuration:
* Modify `/etc/systemd/system/threads.service` in order to create a system service that runs the wsgi app with gunicorn, which also starts at boot time.
* [Unit]
`Description=Gunicorn instance to serve threads`
`After=network.target`
* [Service]
`User=grader`
`Group=www-data`
`WorkingDirectory=/home/ubuntu/threads`
`Environment="PATH=/home/ubuntu/threads/venv/bin"`
`ExecStart=/home/ubuntu/threads/venv/bin/gunicorn --workers 2 --bind unix:/home/ubuntu/threads/run/threads.sock -m 007 wsgi:app`
* [Install]
`WantedBy=multi-user.target`
* Enable and start service:
`sudo systemctl start threads`
`sudo systemctl enable threads`
* Create wsgi entry point
`from threadsPlainFinal import app`
* In nginx config file, proxy requests to the wsgi entry point server_name 34.215.80.168.xip.io, www.34.215.80.168.xip.io;
* location / {
    include proxy_params;
    proxy_pass http://unix:/home/ubuntu/threads/run/threads.sock;
}
* `gunicorn wsgi:app --bind=unix:/home/ubuntu/threads/run/threads.sock`

**Reference**
<ol>
    <li>how to setup Nginx</li>
        https://www.linode.com/docs/web-servers/nginx/how-to-configure-nginx/ 
    <li>set gunicorn and Nginx</li>
        https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-gunicorn-and-nginx-on-ubuntu-18-04
</ol>

### Step 4: Change the SSH port from 22 to 2200

- Use `sudo nano /etc/ssh/sshd_config` to change the port numbers from `22` to `2200`.
- Find the PermitRootLogin, PasswordAuthentication and ProhibitRootLogin, in the same file, and set them all to no.
- Save and exit using CTRL+X and confirm with Y.
- Restart SSH: `sudo service ssh restart`.

### Step 5: Update and configure scripts to automatically manage updates

- `sudo apt-get update`.
- `sudo apt-get upgrade`.
- `sudo apt-get install unattended-upgrades`
- `sudo dpkg-reconfigure --priority=low unattended-upgrades`

### Step 6: Configure the Uncomplicated Firewall (UFW)

- Configure the default firewall for Ubuntu to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
  ```
  sudo ufw status                  # The UFW should be inactive.
  sudo ufw default deny incoming   # Deny any incoming traffic.
  sudo ufw default allow outgoing  # Enable outgoing traffic.
  sudo ufw allow 2200/tcp          # Allow incoming tcp packets on port 2200.
  sudo ufw allow www               # Allow HTTP traffic in.
  sudo ufw allow 123/udp           # Allow incoming udp packets on port 123.
  sudo ufw deny 22                 # Deny tcp and udp packets on port 53.
  ```
- Turn UFW on: `sudo ufw enable`. The output should be like this:
  ```
  Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
  Firewall is active and enabled on system startup
  ```
- Check the status of UFW to list current roles: `sudo ufw status`. The output should be like this:

  ```
  Status: active

  To                         Action      From
  --                         ------      ----
  Nginx Full                 ALLOW       Anywhere
  2200/tcp                   ALLOW       Anywhere
  80/tcp                     ALLOW       Anywhere
  123/udp                    ALLOW       Anywhere
  22                         DENY        Anywhere
  Nginx Full (v6)            ALLOW       Anywhere (v6)
  2200/tcp (v6)              ALLOW       Anywhere (v6)
  80/tcp (v6)                ALLOW       Anywhere (v6)
  123/udp (v6)               ALLOW       Anywhere (v6)
  22 (v6)                    DENY        Anywhere (v6)
  ```

- Exit the SSH connection: `ctrl+D`.
- Click on the `Manage` in the Amazon Lightsail Instance, then the `Networking` tab, and then change the firewall configuration to match the internal firewall settings.
  HTTP TCP 80
  Custom UDP 123
  Custom TCP 2200
- SSH with:
  `ssh -i ~/.ssh/grader.pem -p 2200 ubuntu@34.215.80.168`

**Reference**
Official Ubuntu Documentation, [UFW - Uncomplicated Firewall](https://help.ubuntu.com/community/UFW).

### Step 7: Use `Fail2Ban` to ban attackers

`Fail2Ban` is an intrusion prevention software framework that protects computer servers from brute-force attacks.

- Install Fail2Ban: `sudo apt-get install fail2ban`.
- Install sendmail for email notice: `sudo apt-get install sendmail iptables-persistent`.
- Create a copy of a file: `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`.
- Change the settings in `/etc/fail2ban/jail.local` file:
  ```
  set bantime = 600
  destemail = useremail@domain
  action = %(action_mwl)s
  ```
- Restart the service: `sudo service fail2ban restart`.

### Step 8: Configure local timezone to UTC

- Change the timezone to UTC using following command:
  `sudo timedatectl set-timezone UTC`.

### Step 9: Create a new user named `grader`

- While logged in as `ubuntu`, add user: `sudo adduser grader`.
- Enter a password (twice) and fill out information for this new user.

### Step 10: Give `grader` the permission to sudo

- Edits the sudoers file: `sudo visudo`.
- Search for the line that looks like this:
  ```
  root    ALL=(ALL:ALL) ALL
  ```
- Below this line, add a new line to give sudo privileges to `grader` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```
- Save and exit using CTRL+X and confirm with Y.
- Verify that `grader` has sudo permissions. Run `su - grader`, enter the password,
  run `sudo -l` and enter the password again. The output should be like this:
  ```
  Matching Defaults entries for grader on ip-172-26-13-170.us-east-2.compute.internal:
      env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

  User grader may run the following commands on ip-172-26-13-170.us-east-2.compute.internal:
      (ALL : ALL) ALL
  ```

**Reference**
DigitalOcean, [How To Add and Delete Users on an Ubuntu 14.04 VPS](https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-an-ubuntu-14-04-vps)

### Step 11: SSH key pair for `grader`

- While logged in as `grader`
- Create a new directory called `~/.ssh` (`mkdir .ssh`)
- Copy the contents of the private key `grader` in `~/.ssh/authorized_keys`, create the folder if you don't have it.
- Give the permissions:
* `chown -R  grader.grader /home/grader/.ssh`
* `chmod 700 /home/grader/.ssh`
* `chmod 600 /home/grader/.ssh/authorized_keys`
- Restart SSH: `sudo service ssh restart`
- Connect again with `ssh -i ~/.ssh/grader.pem -p 2200 grader@34.215.80.168`

**Reference**
DigitalOcean, [How To Set Up SSH Keys](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2).

### Step 12: Authenticate login through Google

- Go to [Google Cloud Plateform](https://console.cloud.google.com/).
- Click `APIs & services` on left menu.
- Click `Credentials`.
- Create an OAuth Client ID (under the Credentials tab), and add http://34.215.80.168.xip.io as authorized JavaScript origins.
- Add as authorized redirect URI:
  http://34.215.80.168.xip.io/oauth2callback,
  http://34.215.80.168.xip.io/login and
  http://34.215.80.168.xip.io/gconnect
- Download the corresponding JSON file and replace it in the project `/home/ubuntu/threads/client_secret.json`
- Replace the client ID in the `templates/login.html` file.

### Step 13: start the web application

- Activate the environment with: `source /home/ubuntu/threads/venv/bin/activate`
- Start the app with: `python -i /home/ubuntu/threads/threadsPlainFinal.py`
- Access at http://34.215.80.168.xip.io/
