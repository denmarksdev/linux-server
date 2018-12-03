# Linux Server Configuration
How to configures a Linux server to run web applications in a secure environment.

This configuration uses the [Catalog Item project](https://github.com/denmarksdev/linux-server) to create a **Python WSGI application**

# Application URL
- Public IP **35.199.122.175**
- <a href='http://35.199.122.175/catalog/' target='_blank'>Catalog item web app</a> 

**Notice:** The app is hosted on a [Google Cloud](https://cloud.google.com/), and may be unavailable after the trial period

# Requirements
1.  Use cloud service to create an instance of the Ubuntu Linux server. 
    - I am  using the [Google Cloud service](https://cloud.google.com/)  
1.  Follow the Linux SSH configuration instructions.
1.  Update all installed packages
1.  Change the default port SSH **22** to **2200** add the firewall rule.
    - add the firewall rule in **VPN NETWORK** option on **Google AppEngine**
1.  Configure **Uncomplicated Firewall (UFW)** to only allow connections:
    - SSH **2200**
    - HTTP **80**
    - NPT **123**
1. Create a **grader user** account
1. Give **sudo permission** to user grader
1. Create SSH keys to **grader** using the **ssh-keygen** tool
1. Set the local time zone for **UTC**
1. Install the apache server to serve **mod_wsgi Python application**
1. Install **PostgreSQL** and create user catalog with limited permissions to the application database.
1. Installing **Git**
1. Cloning and Configuring the **Project Item Catalog**
1. Configure the server so that it works correctly by visiting the ip address of your server in a browser. And do not allow the git directory to be publicly accessible through a browser

# Steps

## 1 - Google Cloud Shell

1. Update package information
    - `sudo apt-get update`
1. Install package updates  
    - `sudo apt-get upgrade`
1. Create user **grader** 
    - `sudo adduser grader`
1. Give sudo permission to user **grader**
    - `sudo cp /etc/sudoers.d/google_sudoers /etc/sudoers.d/grader`
    - `sudo nano /etc/sudoers.d/grader` rename google_sudoers to grader
1. Log in with user **grader** 
    - `su grader`
1. In the user's directory **grader**, create an .ssh folder to store the public key.
    - `mkdir .ssh`
1. Generate SSH keys for the **grader** in your local environment
    - `ssh-keygen grader` with the name grader
1. Create the **authorized_key** file in the **.ssh** folder save the public key
    - `sudo nano .ssh/authorized_keys` and paste content the file grader.pub 
1. Change permissions .ssh
    - `sudo chmod 744 .ssh`
1. Change permissions for authorized_keys
    - `sudo chmod 644 .ssh/autthorized_keys`
1. Close the **Google Cloud Shell** and log in to your local environment with a previously generated key
    - `ssh grader@35.199.122.175 -i grader` grader is the private key 
## 2 - Security

 **ATTENTION:** In the Google compute engine, change the default **default-allow-ssh** port to **2200**
      - If you do not do this, you will not be able to communicate through ssh

1. Configure o **Uncomplicated Firewall (UFW)**
```
sudo ufw default deny incoming 
sudo ufw default allow incoming 
sudo ufw allow  2200/tcp
sudo ufw allow web
sudo ufw allow 123/tcp 
```
2. Activate the firewall 
    - `sudo ufw enable`
2. Change the ssh port to **2200** in the configuration file and disable **SSH Root Login**
    - `sudo nano /etc/ssh/sshd_config`
```
# The strategy used for options in the default sshd_config shipped with
# OpenSSH is to specify options with their default value where
# possible, but leave them commented.  Uncommented options override the
# default value.

Port 2200

#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::
...
#LogLevel INFO

# Authentication:

#LoginGraceTime 2m

PermitRootLogin no
...
```	
4. Restart the **SSH** service
- `sudo service ssh restart`

## 3 - Implement the project

### 1 - The Apache server

1. Set the time for **UTC**,
    - `sudo dpkg-reconfigure tzdata` select none of above and **UTC**.
1. Install apache and wsgi
    - `sudo apt-get install apache2`
    - `sudo apt-get install libapache2-mod-wsgi`
1. Install postgresql
    - `sudo apt-get install postgresql`
### 2 - Postgresql
1. Install the Postgresql
    - ` sudo -u postgres psql postgres`
1. Create the database and user catalog
    - `create database catalog;`
    - `create user catalog with password 'somePass'`

### 2 - Create the Catalog Item wsgi application
1. Install git
    - `sudo apt-get install git`
1. create CatalogApp folder
    - `sudo mkdir\var\www\ CatalogApp`
1. Enter the folder
    - `cd \var\www\CatalogApp`
1. Cloning the Project Catalog Item
    - `sudo git clone https://github.com/denmarksdev/catalog.git`
1. Install Viral Python Environment
    - `sudo virtualenv venv --always-copy`
1. Activate the virtual environment
    - `source venv/bin/activate`
1. Install the application packages
    - `sudo pip install -r catalog/requirements.txt`
1. Installing the PostgreSQL Provider
    - `pip install psycopg2-binary`
1. Move the app folder to /var/www/CatalogApp
    - `sudo mv /catalog/app  ./`
1. Move the config.py folder to /var/www/CatalogApp
    - `sudo mv /catalog/config.py  ./`
1. In the configuration file change the provider of the bank and the public address
```
SQLALCHEMY_DATABASE_URI = 'postgresql://catalog:somePass@localhost/catalog'
PUBLIC_URL = "http://35.199.122.175"
```
12. Changing **static Image folder** permissions
    - `sudo ssh chmod 747 app\static\images`
13. Create the application's **WSGI** file
    - `sudo nano catalogapp.wsgi`
```
#!/usr/bin/python

import sys
import logging
import os

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/CatalogApp/")

from app import app as application
from app.sample_data import create as create_sample_data

create_sample_data()
```
14. Create **WSGI** configuration file
    - `sudo nano /etc/apache2/sites-available/CatalogApp.conf`
```
<VirtualHost *:80>
                ServerName 35.199.122.175
                ServerAdmin grader@test.com
                WSGIScriptAlias / /var/www/CatalogApp/catalogapp.wsgi
                <Directory /var/www/CatalogApp/app/>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/CatalogApp/app/static
                <Directory /var/www/CatalogApp/app/static/>
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

15. And finally restart the apache server
    - `sudo service apache2 restart`

# Third-party resources
   
  - [Google Cloud](https://cloud.google.com/)
  - [git](https://git-scm.com/) the version control system  
  - [Flask with apache](http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/)
  - [How To Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
  
