# Linux Server Configuration

This project is a guide to secure and set up a Linux server. This file also 
contains instructions that help to have one of web applications running live on a secure 
a web server.

#### Prerequisites
- Linux server instance ([Amazon Lightsail](https://aws.amazon.com/lightsail/)
 is recommended)
 
### Server Configuration
Following are the steps to configure the server.


##### Get your server
1. Start a new Ubuntu Linux server instance on Amazon Lightsail. There are 
full details on setting up your Lightsail instance on the next page.
2. Follow the instructions provided to SSH into your server.

##### Secure your server
1. Update all currently installed packages
```
sudo apt-get update
sudo apt-get upgrade
```

It may take some time to upgrade the packages.

2. Change the SSH port from 22 to 2200. Make sure to configure the Lightsail 
firewall to allow it. [Reference](https://www.knownhost.com/wiki/security/misc/how-can-i-change-my-ssh-port)
```
sudo vi /etc/ssh/sshd_config
```
Change port from 22 to 2200
```
sudo service ssh restart
```
**Add 2200 port under the Networking tab on the Amazon Lightsail interface. 
This allows you to connect to the 2200 port via the Amazon firewall. 
Otherwise, you won’t be able to connect to the instance using ssh.**

3. Configure the Uncomplicated Firewall (UFW) to only allow incoming 
connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/tcp
sudo ufw enable
```

Run the following command to verify.
```
sudo ufw status
```
You should see the following:
```
Status: active

To                         Action      From
--                         ------      ----
2200/tcp                   ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
123/tcp                    ALLOW       Anywhere                  
2200/tcp (v6)              ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
123/tcp (v6)               ALLOW       Anywhere (v6)
```

4. Create a new user account named grader.
```
sudo adduser grader
```
Enter a password (a_secure_password), name and other details if needed.

5. Give grader the permission to sudo.
```
sudo vi /etc/sudoers.d/grader
```

Add the following line of text:
```
grader ALL=(ALL) NOPASSWD:ALL
```

Login to the grader account
```
sudo su - grader
```

6. Create an SSH key pair for grader using the ssh-keygen tool.
```
ssh-keygen -t rsa -b 4096
Enter a filename and passphrase : a_secure_passphrase
```

The above command generates a private key and public key. You will need to copy 
the generated public keygen to the authorized keys file for the key to be 
authorized.
```
sudo cat ~/.ssh/id_rsa_grader.pub >>  ~/.ssh/authorized_keys
```

### Project Deployment

1. Configure the local timezone to UTC
```
sudo dpkg-reconfigure tzdata

Select ‘None of the above’
Select ‘UTC’
```

You’ll see the following output
```
Current default time zone: 'Etc/UTC'
Local time is now:      Sat Mar 17 20:45:13 UTC 2018.
Universal Time is now:  Sat Mar 17 20:45:13 UTC 2018.

```

2. Install and configure Apache to serve a Python mod_wsgi application
```
sudo apt-get install apache2
```
For Python 3 projects, install the Python 3 mod_wsgi package on your server:
```
sudo apt-get install libapache2-mod-wsgi-py3
```

3. Install and configure PostgreSQL: [Reference](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)
```
sudo apt-get install python3
sudo apt-get install python3-setuptools
sudo apt-get install postgresql postgresql-contrib
```

Do not allow remote connections [Reference](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

Open pg_hba.conf file
```
sudo vi /etc/postgresql/9.5/main/pg_hba.conf
```

Comment the following lines
```
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

Create a new database user named catalog that has limited permissions to your 
catalog application database.
```
sudo -u postgres createuser --interactive with password 'a_secure_password'
Enter name of role to add: catalog
Shall the new role be a superuser? (y/n) n
Shall the new role be allowed to create databases? (y/n) n
Shall the new role be allowed to create more new roles? (y/n) n
```

Create DB
```
sudo -i -u postgres psql
postgres=# CREATE DATABASE catalog_db WITH OWNER catalog;
```

4. Install git.
```
sudo apt-get install git
```

5. Clone and setup a project from the Github repository you created earlier.
```
cd /var/www/
sudo mkdir epl_teams
cd epl_teams
sudo git clone https://github.com/ahdeshpande/EPL-Teams.git epl_teams
```
pip installation (if needed)
```
sudo easy_install3 pip
```
Setup environment by installing packages from requirements.txt
```
cd epl_teams
sudo pip install -r requirements.txt

```

6. Set it up in your server so that it functions correctly when visiting your 
server’s IP address in a browser. Make sure that your .git directory is not 
publicly accessible via a browser. [Reference](https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-ubuntu-16-04)


###### Create a wsgi file

The file name should be the same as the cloned folder name. The location of 
this file is ```/var/www/epl_teams/```

```
sudo vi epl_teams.wsgi
```

The contents of the file should be as follows:
```
#!/usr/bin/python

import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/epl_teams/")

from epl_teams.server import app as application
```

###### Create a \__init\__.py file
Create a ```\__init\__.py``` in the ```epl_teams``` folder that contains the 
code files. This is an initialization file and is required for the ```
.wsgi``` file. Keep the file blank.

###### Disable the .git, static, template and other folder access
Add ```.htaccess``` file in each of the folders. The file contents should be 
as follows:
```
Order allow,deny
Deny from all
Options -Indexes
```

###### Apache server configuration
Create a configuration file for the site (application) to be deployed. The 
best way is to copy the default configuration and edit it.
```
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/epl_teams.conf
```

The contents of the file without the comments should be as follows:
```
<VirtualHost *:80>

        ServerAdmin your_emailj@address.com
        ServerName xx.xx.xx.xx
        WSGIScriptAlias / /var/www/epl_teams/epl_teams.wsgi
        <Directory /home/vagrant/flask/>
                WSGIProcessGroup epl_teams
                WSGIApplicationGroup %{GLOBAL}
                WSGIScriptReloading On

                Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
        
        <Directory /var/www/epl_teams/epl_teams>
                Order allow,deny
                Allow from all
        </Directory>
        
        Alias /static /var/www/epl_teams/epl_teams/static
        
        <Directory /var/www/epl_teams/epl_teams/static/>
                Order allow,deny
                Allow from all
                Options -Indexes
        </Directory>
</VirtualHost>

```
The above contents contain additional code to the one copied using the 
default site.

Disable the default site and enable the site:
```
sudo a2dissite 000-default.conf
sudo a2ensite epl_teams.conf
```

Restart the Apache server
```
sudo service apache2 restart
```

To check for Apache errors:
```
sudo cat /var/log/apache2/error.log
```

###You have now successfully configured the Amazon Lightsail instance and deployed your application on it.


##### Note: Google OAuth doesn't work for public IPs. So you may want to use a domain name.