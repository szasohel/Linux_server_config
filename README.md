# Linux Server Configuration

A flask application is being deployed on to AWS Light Sail. At the time when this document is written, the instance was ubuntu 16.04.30 LTS. Here is the final website link:(http://52.15.239.207/)

IP address: 52.15.239.207
Port: 2200


Please view this (https://github.com/szasohel/restaurant_menu.git) to use the application

To deploy a web app you have to go through these following steps:

## Updating the server
Since you are running ubuntu, its only two commands:

```
sudo apt-get update
sudo apt-get upgrade
```

## Creating the New user with sudo permissions
You are login into the instance. Lets create a new user called grader:
```
sudo adduser grader
```
It will prompt for a password. Add any password to that field.
After the user gets created, run this command to give the grader sudo priviledges:
```
sudo visudo
```
Once your in the visudo file, go to the section user privilege specification
```
root    ALL=(ALL:ALL) ALL
```
You've noticed the root user in the file. You need add the grader user to same section.  It should look like this:
```
grader  ALL=(ALL:ALL) ALL
```
If you want to login into the account, check to see if works: `sudo login grader`.
We've created our new user, lets configure our ssh to non-default port

## Configuring SSH to a non-default port

SSH default port is 22. We want to configure it to non default port.  Since we are using LightSail, we need to open the non default port through the web interface. If you don't do this step, you'll be locked out of the instance. Go to the networking tab on the management console. Then, the firewall option

![LightSail firewall](/images/lightsailfirewall.png)

Go into your sshd config file: `sudo nano /etc/ssh/sshd_config`
Change the following options. # means commented out

```
# What ports, IPs and protocols we listen for
# Port 22
Port 2200

# Authentication:
LoginGraceTime 120
#PermitRootLogin prohibit-password
PermitRootLogin no
StrictModes yes

# Change to no to disable tunnelled clear text passwords
PasswordAuthentication No
```
Once these changes are made, restart ssh: `sudo service restart ssh`
Any user can login into the machine using a specify port: `sudo user@PublicIP -p 2200`
### Creating the ssh keys
```
LOCAL MACHINE:~$ ssh-keygen
```
Two things will happen:
1. You can name the file whatever you want.  Keep in mind when you name the file it will give you a private key and public key
2. It will prompt you for a pass phrase. You can enter one or leave it blank

Once the keys are generated, you'll need to login to the user account aka grader here:
You'll make a directory for ssh and store the public key in an authorized_keys files

```
mkdir .ssh
cd .ssh
```
When you are in the directory, create an authorized_keys file. This where you paste the public key that you generated on your local machine.
```
sudo nano authorized_keys
```
Please double check your path by pwd. You should be in your `/home/grader/.ssh/` when creating the file. You should be login with grader account using the private key from local machine into the server: `ssh user@PublicIP -i ~/.ssh/whateverfile_id_rsa`

## Configuring Firewall rules using UFW
We need to configure firewall rules using UFW. Check to see if ufw is active: `sudo ufw status`. If not active, lets add some rules
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow 123/udp
```
Now, you enable the rules: `sudo ufw enable` and re check the status to see what rules are activity

## Configure timezone for server
```
sudo dpkg-reconfigure tzdata
```
Choose none of the above and choose UTC.  The server by default is on UTC.

## Install Apache, Git, and flask

### Apache
We will be installing apache on our server. To do that:

```
sudo apt-get install apache2
```
If apache was setup correctly, a welcome page will come up when you use the PublicIP. We are going to install mod wsgi, python setup tools, and python-dev
```
sudo apt-get install libapache2-mod-wsgi python-dev
```
We need to enable mod wsgi if it isn't enabled: `sudo a2enmod wsgi`

Let's setup wsgi file and sites-available conf file for our application.
Create the WSGI file in `path/to/the/application directory`
```
WSGI file

import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/catalog/restaurant_menu')

from views import app as application
application.secret_key='super_secret_key'
```
Please note views.py is my python file. Wherever I wrote application logic.

To setup a virtual host file: `cd /etc/apache2/sites-available/catalog.conf`:
```
Virtual Host file
<VirtualHost *:80>
     ServerName  PublicIP
     ServerAdmin email address

     WSGIScriptAlias / /var/www/catalog/catalog.wsgi

     <Directory /var/www/catalog/restaurant_menu>
          Order allow,deny
          Allow from all
     </Directory>
     #Allow Apache to deploy static content
     <Directory /var/www/catalog/restaurant_menu/static>
        Order allow,deny
        Allow from all
     </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel info
      CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```

### Disable default configuration file and enable catalog site

You have to disable difault site configuration file

```
cd /etc/apache2/sites-available/
sudo a2dissite 000-default.conf
sudo a2ensite catalog.conf
service apache2 reload
```

### Git
`sudo apt-get install git`

Clone repository into the apache directory. In the `cd /var/www/catalog`

```
sudo git clone https://github.com/szasohel/restaurant_menu.git
```

### Flask
Do these commands:

```
sudo apt-get install python-pip python-flask python-sqlalchemy python-psycopg2
sudo pip install oauth2client requests httplib2
```

## PostGreSql
`sudo apt-get install postgresql postgresql-contrib`

To ensure that remote connections to PostgreSQL are not allowed, I checked
that the configuration file `/etc/postgresql/9.3/main/pg_hba.conf` only
allowed connections from the local host addresses `127.0.0.1` for IPv4
and `::1` for IPv6.

Create a PostgreSQL user called `catalog` with:

`sudo -u postgres createuser -P catalog`

You are prompted for a password. This creates a normal user that can't create
databases, roles (users).

Create an empty database called `catalog` with:

`sudo -u postgres createdb -O catalog catalog`

Now we should run the `data.py` to create the database and populate the database initially. Note that inside these files your created engine should point to the new databse now :   
```
engine = create_engine('postgresql://catalog:sayed@localhost/catalog')
```

The basic syntax of this statement is:   

```
postgresql://username:password@host:port/database

In **_gconnect():_**

app_token = json.loads(
	open(r'/var/www/catalog/restaurant_menu/client_secret.json', 'r').read())['web']['client_id']

CLIENT_ID = json.loads(
    open('/var/www/catalog/restaurant_menu/client_secret.json', 'r').read())['web']['client_id']

oauth_flow = flow_from_clientsecrets('/var/www/catalog/restaurant_menu/client_secret.json', scope='')
```

# Bibiliography

1. [Abhishek Ghosh](https://github.com/ghoshabhi/P5-Linux-Config)
2. [Oauth Error on apache2](https://discussions.udacity.com/t/oauth-error-on-apache2-linux-server/235498/7)
3. [Steven Wooding](https://github.com/SteveWooding/fullstack-nanodegree-linux-server-config)
4. [Flask Config](http://flask.pocoo.org/docs/0.12/config/)
