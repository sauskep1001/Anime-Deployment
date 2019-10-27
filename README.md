# Anime Project Server Config


## About

This tutorial is a walkthrough of how we can deploy our Flask Application created in the past on a Linux server and configure it for security and create users which are able to ssh to the server.

I've used Ubuntu 16.04 x64 on a DigitalOcean Droplet for thie tutorial.

### Technical Information About the Project

- **Server IP Address:** 167.71.147.83
- **SSH server access port:** 2200
- **SSH login username:** `grader`
- **Application URL:** http://167.71.147.83.xip.io/

## Table of Contents

<!-- TOC -->

- [Anime Project Server Config](#anime-project-server-config)
    - [About](#about)
        - [Technical Information About the Project](#technical-information-about-the-project)
    - [Table of Contents](#table-of-contents)
    - [Steps to Set up the Server](#steps-to-set-up-the-server)
        - [1. Creating the RSA Key Pair](#1-creating-the-rsa-key-pair)
        - [2. Setting Up a DigitalOcean Droplet](#2-setting-up-a-digitalocean-droplet)
        - [3. Logging In as `root` via SSH and Updating the System](#3-logging-in-as-root-via-ssh-and-updating-the-system)
            - [3.1. Logging in as `root` via SSH](#31-logging-in-as-root-via-ssh)
            - [3.2. Updating the System](#32-updating-the-system)
        - [4. Changing the SSH Port from 22 to 2200](#4-changing-the-ssh-port-from-22-to-2200)
        - [5. Configure Timezone to Use UTC](#5-configure-timezone-to-use-utc)
        - [6. Setting Up the Firewall](#6-setting-up-the-firewall)
        - [7. Creating the User `grader` and Adding it to the `sudo` Group](#7-creating-the-user-grader-and-adding-it-to-the-sudo-group)
            - [7.1. Creating the User `grader`](#71-creating-the-user-grader)
            - [7.2. Adding `grader` to the Group `sudo`](#72-adding-grader-to-the-group-sudo)
        - [8. Adding SSH Access to the user `grader`](#8-adding-ssh-access-to-the-user-grader)
        - [9. Disabling Root Login](#9-disabling-root-login)
        - [10. Installing Apache Web Server](#10-installing-apache-web-server)
        - [11. Installing `pip`](#11-installing-pip)
        - [12. Installing and Configuring Git](#12-installing-and-configuring-git)
            - [12.1. Installing Git](#121-installing-git)
            - [12.2. Configuring Git](#122-configuring-git)
        - [13. Installing and Configuring PostgreSQL](#13-installing-and-configuring-postgresql)
            - [13.1. Installing PostgreSQL](#131-installing-postgresql)
            - [13.2. Configuring PostgreSQL](#132-configuring-postgresql)
        - [14. Setting Up Apache to Run the Flask Application](#14-setting-up-apache-to-run-the-flask-application)
            - [14.1. Installing `mod_wsgi`](#141-installing-mod_wsgi)
            - [14.2. Cloning the Item Catalog Flask application](#142-cloning-the-item-catalog-flask-application)
            - [14.3. Setting Up Virtual Hosts](#143-setting-up-the-virtualhost-configuration)
    - [Debugging](#debugging)
    - [References](#references)


## Steps to Set up the Server

### 1. Creating the RSA Key Pair

Generate a private and public key pair on your local machine and then add this public key while created the Droplet, this key will allow use to ssh to our server.

To generate a key pair, run the following command:

   ```console
   $ ssh-keygen
   ```

For this I've created the key with the name `catalog_root`.

You now have a public and private key that you can use to authenticate. The public key is called `catalog_root.pub` and the corresponding private key is called `catalog_root`. The key pair is stored inside the `~/.ssh/` directory.

### 2. Setting Up a DigitalOcean Droplet

1. Log in or create an account on [DigtalOcean](https://cloud.digitalocean.com/login).

2. Go to the Dashboard, and click **Create Droplet**.

3. Choose **Ubuntu 16.04 x64** image from the list of given images.

4. Choose a preferred size. In this project, I have chosen the **1GB/1 vCPU/25GB** configuration.

5. In the section **Add Your SSH Keys**, paste the content of your public key, `catalog_root.pub`:


   This step will automatically create the file `~/.ssh/authorized_keys` with appropriate permissions and add your public key to it. It would also add the following rule in the `/etc/ssh/sshd_config` file automatically:

   ```
   PasswordAuthentication no
   ```

   This rule essentially disables password authentication on the `root` user, and rather enforces SSH logins only. For me it was already disabled but it's good to cross check it once.

 6. Click **Create** to create the droplet. I have the hostname as `ubuntu-anime`. This will take some time to complete. After the droplet has been created successfully, a public IP address will be assigned. In this project, the public IPv4 address that I have been assigned is `167.71.147.83`.

### 3. Logging In as `root` via SSH and Updating the System

#### 3.1. Logging in as `root` via SSH

As the droplet has been successfully created, you can now log into the server as `root` user by running the following command in your host machine:

```
  $ ssh -i catalog_root root@167.71.147.83
```

Cross check that you're providing the correct path to the private key as this is crucial for ssh-ing to the server.


#### 3.2. Updating the System

Run the following command to update the packages:

```
 # apt update && apt upgrade
```

This will update all the packages. If the available update is a kernel update, you might need to reboot the server by running the following command:

```
# reboot
```

### 4. Changing the SSH Port from 22 to 2200

1. Open the `/etc/ssh/sshd_config` file with `nano` or any other text editor of your choice:

   ```
   # nano /etc/ssh/sshd_config
   ```

2. Find the line `#Port 22` (would be located around line 13) and change it to `Port 2200`, and save the file.

3. Restart the SSH server to reflect those changes:
   ```
   # service ssh restart
   ```

4. To confirm whether the changes have come into effect or not, run:
   ```
   # exit
   ```

   This will take you back to your host machine. After you are back to your local machine, run:

   ```
   $ ssh -p catalog_root root@167.71.147.83 -p 2200
   ```
   
   You should now be able to log in to the server as `root` on port 2200. The `-p` option explicitly tells at what port the SSH server operates on. It now no more operates on port number 22. 

### 5. Configure Timezone to Use UTC

To configure the timezone to use UTC, run the following command:

```
# sudo dpkg-reconfigure tzdata
```

It then shows you a list. Choose ``None of the Above`` and press enter. In the next step, choose ``UTC`` and press enter.

You should now see an output like this:

```
Current default time zone: 'Etc/UTC'
Local time is now:      Thu May 24 11:04:59 UTC 2018.
Universal Time is now:  Thu May 24 11:04:59 UTC 2018.
```

### 6. Setting Up the Firewall

Now we would configure the firewall to allow only incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123):

```
$ ufw allow 2200/tcp
$ ufw allow 80/tcp
$ ufw allow 123/udp
```

To enable the above firewall rules, run:

```
$ ufw enable
```

To confirm whether the above rules have been successfully applied or not, run:

```
$ ufw status
```

You should see something like this:

```
Status: active

To                         Action      From
--                         ------      ----
2200/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
123/udp                    ALLOW       Anywhere
2200/tcp (v6)              ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
123/udp (v6)               ALLOW       Anywhere (v6)
```

### 7. Creating the User `grader` and Adding it to the `sudo` Group

#### 7.1. Creating the User `grader`

While being logged into the virtual server, run the following command and proceed:

```
  # adduser grader
```

The output would look like this:

```
  Adding user `grader' ...
  Adding new group `grader' (1000) ...
  Adding new user `grader' (1000) with group `grader' ...
  Creating home directory `/home/grader' ...
  Copying files from `/etc/skel' ...
  Enter new UNIX password:
  Retype new UNIX password:
  passwd: password updated successfully
  Changing the user information for grader
  Enter the new value, or press ENTER for the default
	  Full Name []: Grader
	  Room Number []:
	  Work Phone []:
	  Home Phone []:
	  Other []:
  Is the information correct? [Y/n]
```

**Note**: Above, the UNIX password I have entered for the user `grader` is, `password`. 

#### 7.2. Adding `grader` to the Group `sudo`

Run the following command to add the user `grader` to the `sudo` group to grant it administrative access:

```
  # usermod -aG sudo grader
```

### 8. Adding SSH Access to the user `grader`

To allow SSH access to the user `grader`, first log into the account of the user `grader` from your server:

```
# su - grader
```

You should see a prompt like this:

```console
grader@ubuntuanime:~$
```

Now enter the following commands to allow SSH access to the user `grader`:

```
$ mkdir .ssh
$ chmod 700 .ssh
$ cd .ssh/
$ touch authorized_keys
$ chmod 644 authorized_keys
```

After you have run all the above commands, go back to your local machine and copy the content of the public key file `~/.ssh/catalog_root.pub`. Paste the public key to the server's `authorized_keys` file using `nano` or any other text editor, and save.

After that, run `exit`. You would now be back to your local machine. To confirm that it worked, run the following command in your local machine:


### 9. Disabling Root Login

1. Run the following command on your local machine to log in as `root` in the server:
   ```
   $ ssh -i catalog_root root@167.71.147.83 -p 2200
   ```

2. After you are logged in, open the file `/etc/ssh/sshd_config` with `nano`:
   ```
   # nano /etc/ssh/sshd_config
   ```

3. Find the line `PermitRootLogin yes` and change it to `PermitRootLogin no`.

4. Restart the SSH server:
   ```
   # service ssh restart
   ```

5. Terminate the connection:
   ```
   # exit
   ```


### 10. Installing Apache Web Server

To install the Apache Web Server, run the following command after logging in as the `grader` user via SSH:

```
$ sudo apt update
$ sudo apt install apache2
```

To confirm whether it successfully installed or not, enter the URL `http://167.71.147.83` in your Web browser:

If the installation has succeeded, you should see a default html page served by apache:


### 11. Installing `pip3`

We need pip3 to install project dependencies


If you are using Python 3, to install it, run:

```
$ sudo apt install python3-pip
```

### 12. Installing Git

#### 12.1. Installing Git

We need git to clone our repository from GitHub so we can get our project code into the server.

```
$ sudo add-apt-repository ppa:git-core/ppa
$ sudo apt update
$ sudo apt install git
```


### 13. Installing PostgreSQL

1. Install PostgreSQL:

   ```
   $ sudo apt install postgresql
   ```

#### 13.2. Configuring PostgreSQL

1. Log in as the user `postgres` that was automatically created during the installation of PostgreSQL Server:

   ```
   $ sudo -u postgres psql
   ```

2. Open the `psql` shell:

   ```
   $ psql
   ```

3. This will open the `psql` shell. Now type the following commands one-by-one:

   ```sql
   postgres=# create user catalog with password 'password';
   postgres=# create database catalog with owner catalog;
   ```

   Then exit from the terminal by running `\q` followed by `exit`.

    To disable remote connections, make sure you don't have any other IPs besides 127.0.0.1 in the following file.

    > sudo nano /etc/postgresql/VERSION/main/pg_hba.conf

4. We need to change our code to use PostgreSQL instead of sqlite, this is really simple, just replace the connection URI in `main.py`,`'db_setup.py` and `seed_db.py` with:

    > postgresql://catalog:password@localhost/catalog

### 14. Setting Up Apache to Run the Flask Application

#### 14.1. Installing `mod_wsgi`

The module `mod_wsgi` will allow your Python applications to run from Apache server. 


If you are running Python 3, run this:
   
```
$ sudo apt install libapache2-mod-wsgi-py3
```

This would also enable `wsgi`. So, you don't have to enable it manually.

After the installation has succeeded, restart the Apache server:

```
$ sudo service apache2 restart
```

#### 14.2. Cloning the Item Catalog Flask application

1. Change the current working directory to `/var/www/`:

   ```
   $ cd /var/www/
   ```

2. Create a directory called `FlaskApp` and change the working directory to it:

   ```
   $ sudo mkdir FlaskApp
   $ cd FlaskApp/
   ```

3. Clone your GitHub repository of your _Item Catalog application project_ (Flask project) as the directory `FlaskApp`. For example:

   ```
   $ sudo git clone https://github.com/sauskep1001/FSND---Anime-App FlaskApp
   ```

4. Change the current working directory to the newly created directory:

   ```
   $ cd FlaskApp/
   ```

5. Install required packages:
   
   ```
   $ sudo pip3 install -r requirements.txt
   ```

6. Initialize our DB:
   
   ```
   $ sudo python3 db_setup.py
   ```

7. Seed our DB with some initial data:
   
   ```
   $ sudo python3 seed_db.py
   ```


#### 14.3. Setting Up the VirtualHost Configuration

1. Run the following command in terminal to set up a file called `FlaskApp.conf` to configure the virtual hosts:

   ```
   $ sudo nano /etc/apache2/sites-available/FlaskApp.conf
   ```

2. Add the following lines to it:

   ```

   <VirtualHost *:80>
      ServerName 167.71.147.83
      ServerAlias 167.71.147.83.xip.io
      ServerAdmin sauskep1001@gmail.com
      WSGIScriptAlias / /var/www/FlaskApp/FlaskApp/flaskapp.wsgi
      <Directory /var/www/FlaskApp/FlaskApp/>
          Require all granted
      </Directory>
      Alias /static /var/www/FlaskApp/FlaskApp/static
      <Directory /var/www/FlaskApp/FlaskApp/static/>
          Require all granted
      </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
   </VirtualHost>

   ```
   
   **Note:** Kindly change the IP address to your own server's public IP and the email to yours. 

3. Enable the virtual host:

   ```
   $ sudo a2ensite FlaskApp
   ```

4. Restart Apache server:

   ```
   $ sudo service apache2 restart
   ```

5. Creating the .wsgi File

   Apache uses the `.wsgi` file to serve the Flask app. Move to the `/var/www/FlaskApp/` directory and create a file named `flaskapp.wsgi` with following commands:

   ```
   $ cd /var/www/FlaskApp/FlaskApp/
   $ sudo nano flaskapp.wsgi
   ```

   Add the following lines to the `flaskapp.wsgi` file:

   ```python
   import sys
   import logging
   logging.basicConfig(stream=sys.stderr)
   sys.path.insert(0, "/var/www/FlaskApp/FlaskApp/")

   from main import app as application
   application.secret_key='some secret key'
   ```
   
   In the above code, replace `main` with the name of the main module.
   
6. Restart Apache server:

   ```
   $ sudo service apache2 restart
   ```
   
   Now you should be able to run the application at <http://167.71.147.83.xip.io/>.
   
   **Note**: You might still see the default Apache page despite setting everything up correctly. To resolve it and see your Flask app running, run the following commands in order:
   
   ```
   $ sudo a2dissite 000-default.conf
   $ sudo service apache2 restart
   ```

    To enable social auth to work, you'd need to create a `client_secrets.json` at the project root, instructions are provided in the cloned repository README.

## Debugging

If you are getting an _Internal Server Error_ or any other error(s), make sure to check out Apache's error log for debugging:

```
$ sudo cat /var/log/apache2/error.log
```

## References

[1] https://stackoverflow.com/questions/12728004/error-no-module-named-psycopg2-extensions

[1] https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04
