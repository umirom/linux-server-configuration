# Final project: Linux server configuration

## Project Details:

**Project overview as defined by www.udacity.com:**

"You will take a baseline installation of a Linux server and prepare it to host your web applications. You will secure your server from a number of attack vectors, install and configure a database server, and deploy one of your existing web applications onto it."

This document lists chronologically my steps taken to complete the project.

## 1. Create Lightsail instance & server details

I created an instance of AWS Lightsail to host my application by following the instructions given on https://lightsail.aws.amazon.com.

**Instance blueprint:** OS only - Ubuntu 16.04 LTS

**Instance name:** copper-ocean

*The following instance details will allow the Udacity Grader to access the deployment and the server:*

**Public IP:** 18.184.74.41

**Accessible SSH-Port:** 2200

**Application URL:** http://18.184.74.41

<hr/>

## 2. Connect to the server from my local client

Download default key pair for my region from AWS Lightsail

Store the file *file-with-lightsail-key.pem* locally as ``~/.ssh/[file-with-lightsail-key.pem]`` and modify the file permissions:

``$ cd ~/.ssh``

``$ chmod 600 file-with-lightsail-key.pem``

Connect to server from my local terminal: 

``$ ssh ubuntu@public-ip -i ~/.ssh/file-with-lightsail-key.pem``

## 3. Update and upgrade server, install finger for ease of user management

``$ sudo apt-get update``

``$ sudo apt-get upgrade``

``$ sudo apt-get install finger``

## 4. Add user *grader* and give it sudo permission

``$ sudo adduser grader``

``$ sudo visudo``

Add grader permissions to visudo in section *#user privilege specification*:

``grader ALL=(ALL:ALL) ALL``

Login as grader to check if everything worked:

``$ sudo login grader``

## 5. Change SSH port from default 22 to 2200

**In AWS Lightsail's web interface:**

Go to tab *Networking* --> section *Firewall*

*Add another* to firewall and set value *Custom* to *TCP 2200*

**On server:**

``$ sudo nano /etc/ssh/sshd_config``

Modify file at section:

````
#What ports, IPs and protocols we listen for
#Port 22
Port 2200
````


Then restart sshd:

``$ sudo service sshd restart``


Now I am able to log in from my local terminal:

``$ ssh ubuntu@[public-ip] -i ~/.ssh/[file-with-lightsail-key.pem] -p 2200``

## 6. Generate ssh keypair for user *grader*

**On local machine**

``$ ssh-keygen``

Store on local machine as ``~/.ssh/[name-of-key-gen-file]``

**On server, log in as user *grader* and create a new directory for the file *authorized keys* **

``$ sudo login grader``

``$ mkdir /home/grader/.ssh``

``$ sudo nano /home/grader/.ssh/authorized_keys``

**Paste content of locally generated and stored public key document into the *authorized_keys* file.**

Now I am able to log in as *grader* from my local terminal without providing a password:

``$ ssh grader@[public-ip] -i ~/.ssh/[name-of-key-gen-file] -p 2200``

## 7. Configure Firewall (UFW)

``$ sudo ufw status``

``$ sudo ufw default deny incoming``

``$ sudo ufw default allow outgoing``

``$ sudo ufw allow 2200/tcp``

``$ sudo ufw allow www``

``$ sudo ufw allow 123/udp``

``$ sudo ufw allow ndp``

``$ sudo ufw enable``

## 8. Configure Timezone to UTC

``$ sudo dpkg-reconfigure tzdata``

Select *None of the above* and then *UTC*

## 9. Install Apache and WSGI

``$ sudo apt-get install apache2``

``$ sudo apt-get install libapache2-mod-wsgi python-dev``

**Enable WSGI:**

``$ sudo a2enmod wsgi``

**Restart Apache:**

``$ sudo apache2ctl restart``

Now, when visiting the public IP in a browser, the Apache default page is on display.

## 10. Install Postgresql, create and configure database and database user

``$ sudo apt-get install postgresql postgresql-contrib``

Make sure that no remote connections are allowed:

``$ sudo nano /etc/postgresql/9.5/main/pg_hba.conf``

Log in to psql als postgres default user:

``$ sudo su postgres``

``$ psql``

**Create new user named "catalog" (connected as postgres-user)**

``postgres=# CREATE USER catalog WITH PASSWORD 'catalog';``

**Create new database named "catalog" and associate it with user "catalog" (connected as postgres-user)**

``postgres=# CREATE DATABASE catalog WITH OWNER catalog;``

**Revoke rights from user public (connected as postgres-user)**

``postgres=# REVOKE ALL ON SCHEMA public FROM public;``

**Grant rights to user catalog (connected as postgres-user)**

``postgres=# GRANT ALL ON SCHEMA public TO catalog;``

**Log out as postgres-user and become ubuntu-user again**

``postgres=# \q``

``exit``

## 11. Create application directory

``$ cd /var/www``

``$ sudo mkdir catalog``

Make grader the owner of the new diretory:

``$ sudo chown -R grader:grader catalog``

## 12. Install Flask and App Dependencies

````
$ sudo apt-get install python-pip python-flask python-sqlalchemy python-psycopg2
$ sudo pip install oauth2client requests httplib2
````

## 13. Install git and clone application repo to the new directory

``$ sudo apt-get install git``

``$ cd /var/www/catalog``

``$ git clone [URL to my github repository]``

Now the folder with all my project files sits inside the newly created application directory.

**Rename the repository and the file that contains the application logic for ease of use

In my case, the cloned repository had the name *itemsCatalog* and the file with the main application logic was called *recipes.py*.

Make sure to be inside the application directory:

``$ cd /var/www/catalog``

Rename the directory that holds the repository:

``$ sudo mv ./itemsCatalog ./catalog`` 

Rename the file with the application logic:

````
$ cd /var/www/catalog/catalog
$ sudo mv recipes.py __init__.py
````

Now, my file structure looks as follows:

|var
||www
 ||catalog
  ||catalog
   ||__init__.py
   ||static
   ||teplates
   ||*etc* *etc*

## 14. Create WSGI file inside the new directory

Make sure to create the file in the top-level catalog directory:

``$ sudo nano /var/www/catalog/catalog.wsgi``

Now, my file structure looks as follows:

|var
||www
 ||catalog.wsgi
 ||catalog
  ||catalog
   ||__init__.py
   ||static
   ||teplates
   ||*etc* 

Paste the following content inside the catalog.wsgi file and verify that the path and the module name for the import are correct and that the application secret key is identical to the one in __init__.py:

````#WSGI File
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super-secret_key'
````
## 15. Create Virtual Host File and enable it

Create a new conf-file for the application inside /etc/apache2/sites-available

``$ sudo nano /etc/apache2/sites-available/catalog.conf``

Add code to the catalog.conf file and verify that ServerName, ServerAdmin and all indicated paths are correct:

````
<VirtualHost *:80>
	ServerName 18.184.74.41
	ServerAdmin lenz.sophie@gmail.com
	WSGIScriptAlias / /var/www/catalog/catalog.wsgi
	<Directory /var/www/catalog/catalog>
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
````

Enable virtual host for my application:

``$ sudo a2ensite catalog.conf``

Disable default Apache site:

``$ sudo a2dissite 000-default.conf``

Restart Apache:

``$ sudo service apache2 restart``

## 16. Update project files with new engine connection

Open the project file /var/www/catalog/catalog/database_setup.py and update the engine connection:

``engine = create_engine('postgresql://catalog:catalog@localhost/catalog')``

Then, open the project file with the python logic /var/www/catalog/catalog/__init__.py and update the engine connection here as well:

``engine = create_engine('postgresql://catalog:catalog@localhost/catalog')``

Create database schema by running database_setup.py

``$ sudo python database_setup.py``

## 17. Update project files with new path for oauth client login

Open the project file with the python logic /var/www/catalog/catalog/__init__.py and update the paths in the client id and the oauth_flow:

````
CLIENT_ID = json.loads(
	open('/var/www/catalog/catalog/client_secrets.json', 'r').read())['web']['client_id']
````

````
oauth_flow=
flow_from_clientsecrets('/var/www/catalog/catalog/client_secrets.json', scope='')
````




