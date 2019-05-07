# Udacity-Linux-Server-Configuration

### Project Overview
> Baseline installation of a Linux server and prepare it to host your web applications. Secure a server from a number of attack vectors, install and configure a database server, and deploy one of your existing web applications onto it.

### What will I learned
> Learned how to access, secure, and perform the initial configuration of a bare-bones Linux server. How to install and configure a web and database server and actually host a web application.

# IP & Hostname
> + URL: www.savionj.com 
> + Host Name: http://35.173.115.27 
> + IP Adress: 35.173.115.27
> + Port: 2200 

# Prerequisites 
> + [AWS Account](https://aws.amazon.com/lightsail/)
> + [Vagrant](https://www.vagrantup.com/)
> + Ubuntu server
> + [PostgreSQL](https://www.postgresql.org/)
> + Web Application

# Get your server
### Start a new Ubuntu Linux server
+ Create an account and into Amazon Lightsail 
+ Click Create instance
+ Choose Linux/Unix platform, OS Only and Ubuntu 16.04 LTS.
+ Choose a instance plan
+ Used Lightsail defaults name provided by AWS or rename your instance.
+ Click the 'Create' button to create the instance.
+ Wait for the instance to start up
+ Attach a static IP
<img src="Screenshots/Lightsail1.jpg">

# SSH into your server
#### Within your Lightsail terminal 
+ Download default key
+ Rename to LightsailDefaultKey.pem
+ Copy to shared vagrant folder 

#### Within Vagrant Box
+ cd /vagrant
+ sudo mv LightsailDefaultKey.pem ~/.ssh/
+ sudo chmod 600 ~/.ssh/LightsailDefaultKey.pem
+ sudo ssh -i /home/vagrant/.ssh/LightsailDefaultKey.pem ubuntu@__your static ip__

# Secure your server
### Update all currently installed packages
#### Within your Lightsail terminal
+ sudo apt-get update
+ sudo apt-get upgrade
+ sudo apt-get autoremove

# Change the SSH port from 22 to 2200
> When you change the SSH port, the Lightsail instance will no longer be accessible through the web app 'Connect using SSH' button. 

#### Within Lightsail terminal

+ sudo nano /etc/ssh/sshd_config
+ uncomment and change the port number on line 41 from 22 to 2200
  <img src="Screenshots/portChange.jpg">
+ confirm line that says PasswordAuthentication is set to 'no'
+ sudo service ssh restart
+ log out of Lightsail terminal

#### While on the Lightsail website
1. Click __networking__ within your AWS Instance
2. Under Firewall, click 'Add another'
3. Add a custom TCP with port 2200

<img src="Screenshots/Firewall.jpg">

#### Within Vagrant Box
ssh -i /home/vagrant/.ssh/LightsailDefaultKey.pem ubuntu@(__your static ip__) -p 2200

> You can __**NO LONGER**__ connect via Lightsail SSH which uses __Port 22__

# Configure the Uncomplicated Firewall (UFW)

+ sudo ufw status                                
+ sudo ufw default deny incoming 
+ sudo ufw default allow outgoing
+ sudo ufw allow 2200/tcp
+ sudo ufw allow www
+ sudo ufw allow 123/udp
+ sudo ufw enable

# Create a grader user
### Within lightsail remote VM

+ sudo apt install finger
+ sudo adduser grader

# Give the grader permission to sudo
### Within Lightsail remote VM

+ sudo touch /etc/sudoers.d/grader
+ sudo nano /etc/sudoers.d/grader 
> type in grader ALL=(ALL:ALL) ALL 

> verify grader has sudo access
+ su grader
+ sudo -l

# Create an SSH key pair for grader
#### Within Vagrant Box

+ ssh-keygen -f ~/.ssh/*linuxCourse*
+ cat ~/.ssh/linuxCourse.pub

> Copy the key to clipboard

<img src="Screenshots/Create private Key.jpg">

### Within Lightsail remote VM

+ cd /home/grader
+ sudo mkdir .ssh
+ sudo touch .ssh/authorized_keys
+ sudo nano .ssh/authorized_keys

> Paste the key and save

+ Sudo chmod 700 .ssh
+ Sudo chmod 644 .ssh/authorized_keys
+ sudo chown -R grader:grader /home/grader/.ssh
+ sudo service ssh restart
+ exit

#### Within Vagrant Box
+ ssh -i /home/vagrant/.ssh/linuxCourse grader@__your static ip__ -p 2200
>  Can log into remote server as grader

# Preration for project deployment 
## Configure local timezone
#### Within Vagrant Box
> sudo dpkg-reconfigure tzdata

 <img src="Screenshots/Set time.jpg">
  
# Install Apache
+ sudo apt install apache2
+ sudo apt-get install libapache2-mod-wsgi-py3
+ sudo a2enmod wsgi
+ sudo service apache2 restart

# Install Postgres
+ sudo apt-get install postgresql
> Edit the pg_hba.conf
sudo nano /etc/postgresql/9.5/main/pg_hba.conf <br/>
> psql by default disables the remote connections, but we still need to make sure <br/>

<img src="Screenshots/verifiy database config.jpg">

# Create a catalog user
+ sudo adduser catalog (pwd:catalog)
+ sudo touch /etc/sudoers.d/catalog
+ sudo nano /etc/sudoers.d/catalog 
> type in 'catalog ALL=(ALL:ALL) ALL'

# Create catalog database role and privileges

+ sudo -u postgres psql
> The prompt should say postgres=# 
``` postgres
  CREATE USER catalog WITH PASSWORD 'catalog'; 
  ALTER User catalog CREATEDB; 

  \q
```
# Install Git
+ sudo apt-get install git
## Clone app from GitHub

+ cd /var/www
+ sudo mkdir catalog 
+ cd /var/www/catalog/
+ sudo git clone https://github.com/*YourGithHubProfileName*/*TheGitHubRepository*.git catalog 
> change the owner of the catelog folder
+ sudo chown -R grader:grader /var/www/catalog

# Create virtual environment and load database
+ sudo apt-get install python3-pip
+ sudo apt-get install python-virtualenv
+ cd /var/www/catalog/catalog 
+ sudo virtualenv -p python3 venv3 
+ sudo chown -R grader:grader venv3/

+ source venv3/bin/activate

> Installs 
``` ssh 
> pip3 install httplib2 
> pip3 install requests
> pip3 install --upgrade oauth2client 
> pip3 install sqlalchemy 
> pip3 install flask 
> pip3 install sqlalchemy-utils 
> sudo apt-get install libpq-dev 
> pip3 install psycopg2 
> pip3 install Flask-SQLAlchemy 
```
> Execute all three programs, one at a time
> The first will create the database, the second will insert data, and the third is the main program which will be executed later.
```
> python3 /var/www/catalog/catalog/_CreateDatabase.py
> python3 /var/www/catalog/catalog/_InsertSampleData.py
```
+ sudo -u postgres psql

> Run test query
``` postgres
\c itemcatalog 
Select * from category 
\q
```
<img src="Screenshots/Test Query.jpg">

> Deactivate the virtual environment
+ deactivate

# Edit the wsgi.conf to use python 3
> Add the following line in /etc/apache2/mods-enabled/wsgi.conf file to use Python 3
```
#WSGIPythonPath directory|directory-1:directory-2:...
WSGIPythonPath /var/www/catalog/catalog/venv3/lib/python3.6/site-packages
```
# Create or edit catalog.conf
+ sudo nano into catalog.conf file

> Add or edit the text below 
``` 
<VirtualHost *:80> 
    ServerName [*your static ip*] 
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi 
    <Directory /var/www/catalog/catalog/> 
        Order allow,deny 
        Allow from all 
    </Directory>  
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/> 
        Order allow,deny
        Allow from all  
    </Directory>  
    ErrorLog ${APACHE_LOG_DIR}/error.log  
    LogLevel warn 
    CustomLog ${APACHE_LOG_DIR}/access.log combined 
</VirtualHost>  
```
# Disable the default Apache site
 + sudo a2dissite 000-default.conf

# Activate project app
 + sudo a2ensite catalog
 + sudo service apache2 reload

# Create /var/www/catalog/catalog.wsgi file 
> it should read like below 
```python
activate_this = '/var/www/catalog/catalog/venv3/bin/activate_this.py' 
with open(activate_this) as file_: 
exec(file_.read(), dict(__file__=activate_this)) 

#!/usr/bin/python 
import sys
import logging 
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog/") 
sys.path.insert(1, "/var/www/catalog/")

from catalog import app as application 
application.secret_key = "super_secret_key" 
```
> restart apache 
+ sudo service apache2 restart

# Launch the web app  
> Change the ownership of the project directories 
+ sudo chown -R www-data:www-data catalog/
+ sudo service apache2 restart

> Go to the browser and enter www.savionj.com

# Fixing Google Login 
+ Edit Google app settings
+ Download and save new cliet_secrets.json
+ Edit GoDaddy domain DNS to point to IP address

<img src="Screenshots/Google OAuth.jpg">

#### Edit the client_secrets.json
```
sudo nano /var/www/catalog/catalog/client_secrets.json
```
+ Delete all of the file contents
+ Copy and paste the new info into the file and save

+ Launch the app again using the DNS and test the login button

> Google oauth 2.0 credentials no longer accept xip.io and I could not find any way around not 
having a public top level domain for google to accept for oauth. So I register my own [domain](www.savionj.com) and pointed my domaian's DNS to my Lightsail server IP address. 

<img src="Screenshots/Setu DNS.jpg">

# References 
> + https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-ubuntu-16-04
> + https://www.youtube.com/watch?v=HcwK8IWc-a8
> + https://ubuntuforums.org/showthread.php?t=1053137
> + https://libraries.io/github/golgtwins/Udacity-P7-Linux-Server-Configuration

