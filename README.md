# Linux Server Configuration
These are a set of instructions that specify how a Linux Server was deployed to host a Python web application with a PostgreSQL database, along with it's configurations. The server was deployed using AWS LightSail. Software and packages used include:

- PostgreSQL
- Apache2
- Flask
- mod_wsgi

## Server Details
Public IP: 18.191.240.195

SSH Port: 2200

Web Application URL (both work): 
- http://18.191.240.195.xip.io/
- http://ec2-18-191-240-195.us-east-2.compute.amazonaws.com

## Configuration steps

### Updating to latest packages
1. Run `sudo apt-get update`
1. Run `sudo apt-get upgrade`

### Configure the firewall
1. The default SSH port is change from `22` to `2200` by accessing `/etc/ssh/sshd_config file` and changing the port number
1. In order to avoid being locked out of the server ssh must be restarted: `sudo service ssh restart`
1. Incoming firewall connection were denied and all out going connections were allowed:
- `sudo ufw default deny incoming`
- `sudo ufw default allow outgoing`
1. SSH is enabled: `sudo ufw allow ssh`
1. SSH is reconfigured to port 2200: `sudo ufw allow 2200/tcp`
1. Since SSH is being used on another port, port 22 is blocked: `sudo ufw denny 22`
1. HTTP is enabled: `sudo ufw allow www`
1. NTP is enabled: `sudo ufw  allow 123/udp`
1. Firewall is enabled: `sudo ufw enable`
1. Firewall status can be checked to ensure only ports specified above are following corresponding rules: `sudo ufw status`


### Connecting from local machine
Since AWS LightSail uses port 22 to allow access to the server and port 22 has been blocked, it is necessary to connect to the server
from a remote machine. The instructions below use Mac OS Terminal.
1. From the AWS LighSail instance download a default key and save the key in: `~/.ssh/`
1. For security purposes on the AWS side the permissions for the key should be changed: `chmod 600 ~/.ssh/keyname`
1. Connect to the remote server with SSH on the specified SSH port: `ssh -i ~/.ssh/keyname ubuntu@PUBLICIP`

### Creating user `grader` and granting remote access
1. Enter `sudo adduser grader` and enter corresponding information
1. Give grader sudo permissions by creating `sudo vi /etc/sudoers.d/grader` and writing `grader ALL=(ALL) NOPASSWD:ALL`
1. On your local machine run `ssh-keygen` enter a filename and a passphrase (this key will be used when wanting to log in as grader)
1. Switch to the grader user by running `su grader` and entering the password
1. On the home directory create a .ssh directory: `.ssh` and set the following permissions `chmod 700 .ssh`
1. Within .ssh create the following `vi authorized_keys` and copy the content from the `~/.ssh/keyname.pub` file found on the local machine
1. Change permission of authorized_keys: `chmod 644 authorized_keys`
1. Force key based authentication by setting `PasswordAuthentication yes` to  `PasswordAuthentication no` on `/etc/ssh/sshd_config` on virtual machine
1. Login as grader from a remote machine can now be done as `ssh -i ~/.ssh/grader_key -p 2200 grader@IPADDRESS`

### Configure the local timezone to UTC
1. Run `sudo dpkg-reconfigure tzdata`, enter `None of the Above`, select `UTC`

### Install and configure Apache
1. Run `sudo apt-get install apache2` to install Apache
1. Visit the AWS LightSail IP address shown above, the default apache page should display

### Install necessary software
1. Install the mod_wsgi and python-dev: `sudo apt-get install libapache2-mod-wsgi python-dev`
1. Install PostgreSQL by running `sudo apt-get install postgresql`
1. Enter: `python` to ensure python is installed

	
### Create a new PostgreSQL user named `catalog` with limited permissions
1. PostgreSQL creates a Linux user `postgress` during installation
1. Switch to user: `sudo su - postgres` (this should only be done when dealing with PostgreSQL software)
1. Connect to psql by running `psql`
1. Create the `catalog` user: `CREATE ROLE catalog WITH LOGIN;`
1. Enable catalog to create databases: `ALTER ROLE catalog CREATEDB;`
1. Give the `catalog` user a password by running `\password catalog`
1. Exit psql by running `\q`
1. Enter: `exit` to logout of postgres user


### Create PostgreSQL database
1. Create a new user `catalog`
1. Give `catalog` sudo permissions
1. Log in as catalog and run `sudo su catalog`
1. Create database: `createdb catalog`
1. Enter: `exit` to logout of catalog user    


### Install git and clone web application
1. Run `sudo apt-get install git`
1. Create a directory called `shoeCatalog`in the `/var/www/` and change to that directory
1. Clone project: `sudo git clone https://github.com/ortizmauricio/shoecatalog.git`
1. Change `shoeCatalog` ownership to grader from `/var/www`: sudo chown -R ubuntu:ubuntu shoeCatalog/

### Adapting Web Application From Local Machine Configuration To Actual Server
1. Change directory to `/var/www/shoeCatalog/shoecatalog`
1. Change final_project.py file to `__init__.py` by running `mv final_project.py __init__.py`
1. Within `__init__.py` change `app.run(host='0.0.0.0', port=8000)`  to `app.run()`
1. Delete, rename, or move the database_setup.py file to another directory

1. In database_setup.py and `__init__.py` change `engine = create_engine('sqlite:///shoecatalogwithusers.db')` to:
`engine = create_engine('postgresql://catalog:DATABASEPASSWD@localhost/catalog')`
1. Instruction for modifying the client_secrets.json files are found in the Additional notes section

### Configure environment and necessary dependencies
1. Install virtualenv `sudo apt-get install python-virtualenv`
1. Change to the /var/www/shoeCatalog/shoecatalog/ directory; choose a name for a temporary environment: `virtualenv catalogenv`

1. Activate environtment: `. catalog/bin/activate`

1. With the virtual environment active, install the following dependenies with `pip install`:
	- httplib2
	- flask
	- sqlalchemy
	- psycopg2
	- --upgrade oauth2client

1. Install `sudo apt-get install libpq-dev`
1. Run `python __init__.py` to ensure everything was installed properly
1. Exit environment


### Set up and enable a virtual host
1. Enter `vi /etc/apache2/sites-available/shoeCatalog.conf;
1. Add the following into the file:

	```
	<<VirtualHost *:80>
            ServerName 18.191.240.195
            ServerAdmin yourEmail@email.com
            WSGIScriptAlias / /var/www/shoeCatalog/shoeCatalog.wsgi
            <Directory /var/www/shoeCatalog/shoecatalog/>
                    Order allow,deny
                    Allow from all
                    Options -Indexes
            </Directory>
            Alias /static /var/www/shoeCatalog/shoecatalog/static
            <Directory /var/www/shoeCatalog/shoecatalog/static/>
                    Order allow,deny
                    Allow from all
                    Options -Indexes
            </Directory>
            ErrorLog ${APACHE_LOG_DIR}/error.log
            LogLevel warn
            CustomLog ${APACHE_LOG_DIR}/access.log combined
	</VirtualHost>

	```

1. Enable virtual host `sudo a2ensite shoeCatalog` to 
1. Run `sudo service apache2 reload`


### Write a .wsgi file
1. Apache runs Flask application with .wsgi file; create shoeCatalog.wsgi in /var/www/shoeCatalog and add:
1. Add the following to the file:

	```
		activate_this = '/var/www/shoeCatalog/shoecatalog/catalogenv/bin/activate_this.py'
		execfile(activate_this, dict(__file__=activate_this))

		#!/usr/bin/python
		import sys
		import logging
		logging.basicConfig(stream=sys.stderr)
		sys.path.insert(0,"/var/www/shoeCatalog/")

		from shoecatalog import app as application
		application.secret_key = '12345'

	```
1. Restart Apache: `sudo service apache2 restart`
1. Disable the default Apache site `sudo a2dissite 000-default.conf`
1. Enter `sudo service apache2 reload`
1. In the /var/www directory, run: `sudo chown -R www-data:www-data shoeCatalog/`

## Access Server
1. Visit:
	- http://18.191.240.195.xip.io/
	- http://ec2-18-191-240-195.us-east-2.compute.amazonaws.com

## Additional Notes
### OAuth Redirect URI via Google API
1. In the Google API console the Redirect URI and Javascript origins should be set to XXX.XXX.XXX.XXX.xip.io and http://ec2-18-191-240-195.us-east-2.compute.amazonaws.com
1. Downloade the JSON client secrets file and replace the content ini `/var/www/shoeCatalog/shoecatalog/client_secrets.json` with the content that was just downloaded

The Google API does not allow IP addresses to be used in either the Redirect URI or Javascript origins so a wildcard DNS such as .xip.io can be used 


## Third Party Resources
- Providing WildCard DNS: http://xip.io/
- Integrating Apache with Flask: https://www.jakowicz.com/flask-apache-wsgi/
- Upgrading all packages: https://serverfault.com/questions/262751/update-ubuntu-10-04/262773#262773
- Udacity Linux Security: https://www.udacity.com/course/configuring-linux-web-servers--ud299




