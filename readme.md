# Linux Server Configuration

Linux server setup for hosting Python webapp.

## General info

_IP adress_: 34.201.252.43

_URL_: http://34.201.252.43/

_SSH port_: 2200


## Setup walkthrough

### 1. Create new User: *grader*

Once I was logged into my Ubuntu server -provided by Amazon Lightsail-, add a new user.
```sh
$ sudo adduser grader
```
Give *grader* sudo permissions by creating a new file on the sudoers directory.

```sh
$ sudo nano /etc/sudoers.d/grader
```
Inside that file, include grader:
```sh
# /etc/sudoers.d/grader
grader ALL=(ALL:ALL) ALL
```

### 2. Update Packages

```sh
$ sudo apt-get update
```
```sh
$ sudo apt-get upgrade
```

### 3. Configure localtime

Select time configuration. In my case, CEST.
```sh
$ sudo dpkg-reconfigure tzdata
```
### 4. Give *grader* a rsa key

Since I'm a windows user, I had to use [PuTTy](http://www.putty.org/) to create the rsa key pair.

After the key is created, you need to convert it to OpenSSH.
```
Conversions > Export OpenSSH
```

Then, you create a .ssh directory, and inside, create a *authorized_keys*, and paste the outcome of the key generator in there.
```sh
# /home/grader
$ sudo mkdir .ssh
$ sudo touch .ssh/authorized_keys
# Past output in keys
$ sudo nano .ssh/authorized_keys
```

*grader* can now login directly to the server:
```sh
$ ssh -i [PuTTy file] grader@[serverIp]
```

### 5. Change port to 2200

Modify port from 22 to 2200.
```sh
$ sudo nano /etc/ssh/sshd_config
```

Then restart ssh:
```sh
$ sudo service ssh restart
```

Also, opening that port was needed on Lightsail, in nextwork tab.

### 6. Disable ssh login for root user and avoid password authentication

Inside /etc/ssh/sshd_config:
```
PermitRootLogin no
PasswordAuthentication no
```
Then restart the service:
```sh
$ sudo service ssh restart
```

### 7. Firewall
Checking that firewall is inactive
```sh
$ sudo ufw status
```
Allow ssh 2200 port, http 80 port and 123 udp port.
```sh
$ sudo ufw allow 2200/tcp
```
```sh
$ sudo ufw allow 80/tcp
```
```sh
$ sudo ufw allow 123/udp
```
```sh
$ sudo ufw enable
```

### 8. Install Apache and mod_wsgi
First, install apache2:
```sh
$ sudo apt-get install apache2
```
Then, install mod_wsgi, that makes Apache serve python Flask applications
```sh
$ sudo apt-get install libapache2-mod-wsgi python-dev
```
Finally, enable mod_wsgi and run apache2 server.
```sh
$ sudo a2enmod wsgi
$ sudo service apache2 start
```

### 9. Configure git
Lightsail already includes git, so you only have to include your name and your email.
```sh
$ git config --global user.name Pablo Egido
$ git config --global user.email pgarciaegido@gmail.com
```

### 10. Clone repository
In */var/www* create a new folder catalog, and change owner.
```sh
$ sudo mkdir catalog
$ sudo chown -R grader:grader catalog
```
Inside that folder, clone the repo.
```sh
$ git clone https://github.com/pgarciaegido/python_catalog
```
Create a *catalog.wsgi* file, with the following config:
```python
# /var/www/catalog
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from python_catalog import app as application
application.secret_key = 'super_secret_key'
```

### 11. Install project dependencies and virtual enviroment
First of all, install Pip:
```sh
$ sudo apt-get install python-pip
```
Install virtual enviroment:
```sh
$ sudo pip install virtualenv
```

Create new virtual enviroment into the catalog folder, and activate it:
```sh
# /var/www/catalog
$ sudo virtualenv venv
$ source venv/bin/activate
```
Install dependencies with pip:
```sh
$ sudo pip install Flask httplib2 request oauth2client sqlalchemy python-psycopg2
```

### 12. Configure virtual host
Create a virtual host config file:
```sh
$ sudo nano /etc/apache2/sites-available/catalog.conf
```
With following code:
```
<VirtualHost *:80>
    ServerName 34.201.252.43
    ServerAlias ec2-34-201-252-43.us-west-2.compute.amazonaws.com
    ServerAdmin admin@34.201.252.43
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/python_catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/python_catalog/app/static
    <Directory /var/www/catalog/python_catalog/app/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Then enable it:
```sh
$ sudo a2ensite catalog
```

### 13. Install Postgresql
First, install all necessary dependencies:
```sh
$ sudo apt-get install libpq-dev python-dev
$ sudo apt-get install postgresql postgresql-contrib
```
Login with the postgres user and connect to database:
```sh
$ sudo su - postgres
$ psql
```
Create new user with a password and give it CREATEDB powers:
```
# CREATE USER catalog WITH PASSWORD 'udacity';
# ALTER USER catalog CREATEDB;
```
Create database and revoke permissions:
```
# CREATE DATABASE catalog WITH OWNER catalog;
# \c catalog
# REVOKE ALL ON SCHEMA public FROM public;
# GRANT ALL ON SCHEMA public TO catalog;
```
Logout from postgres and come back to grader:
```
# \q
$ exit
```

Now, I had to change session_setup.py file:
```python
# /var/www/catalog/python_catalog/app/models/session_setup.py
def export_db_session():
    engine = create_engine('postgresql://catalog:udacity@localhost/catalog')
    Base.metadata.bind = engine
    DBSession = sessionmaker(bind=engine)
    return DBSession()
```

Then, create databases runing models file:
```sh
python /var/www/catalog/python_catalog/app/models/models.py
```

### 14. Getting ready!
It was necessary to modify some import routes inside the python modules. Checking the apache logs was essential for the debugging process. They can be found here:
```sh
$ sudo less /var/log/apache2/error.log
```
Also, from Google and Facebook developers console, the new url needs to be included.

Restart apache server and check if everything works!
```sh
$ sudo service apache2 restart
```

Resources used to acomplish the project:
* Ubuntuforums
* Udacity forums
* Askubuntu
* Digital Ocean's community tutorials
