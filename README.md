# Linux Server Configuration


A baseline installation of a Linux distribution on a virtual machine and prepare it to host web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.


# Server Details

    Public IP address : 35.154.231.1
    SSH port : 2200
    URL : http://ec2-35-154-231-1.ap-south-1.compute.amazonaws.com


# Server Configuration Steps

### 1. Creating an AWS Lightsail instance. Download the private key to your local machine.

### 2. SSH into the server
    1. Moved the primary key to the .ssh directory.
    2. Apply RW owner rights on the key
        $ chmod 600 .ssh/LightsailDefaultPrivateKey-ap-south-1.pem
    3. SSH into the instance
        $ ssh ubuntu@35.154.231.1 -i ~/.ssh/LightsailDefaultPrivateKey-ap-south-1.pem
        * Make sure you have a good and a stable internet connection to ssh into the server

### 3. Create a user grader
    $ sudo adduser grader
    $ sudo nano /etc/sudoers.d/grader
    Add "grader ALL=(ALL:ALL) ALL" to the newly created file and save

### 4. Setup SSH keys for user grader
    Source : https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2

### 5. Change SSH port to 2200
    $ sudo nano /etc/ssh/sshd_config
    Make sure PasswordAuthentication is set to no
    Change port to 2200 from 22
    Change PermitRootLogin to no
    $ sudo service ssh restart
    Change timezone to UTC using $ sudo timedatectl set-timezone UTC

### 6. Update and upgrade all packages
    $ sudo apt-get update
    $ sudo apt-get upgrade

### 7. Configure the firewall (UFW)
    $ sudo ufw default deny incoming
    $ sudo ufw default allow outgoing
    $ sudo ufw allow 2200/tcp
    $ sudo ufw allow www
    $ sudo ufw allow ntp
    $ sudo ufw enable

### 8. Install apache2, mod-wsgi and git
    $ sudo apt-get install apache2 libapache2-mod-wsgi git
    $ sudo a2enmod wsgi

### 9. Install and configure PostgreSQL
    1. Installing python dependencies and PostgreSQL
        $ sudo apt-get install libpq-dev python-dev
        $ sudo apt-get install postgresql postgresql-contrib
    2. Log into PostgreSQL shell
        $ sudo su - postgres
        $ psql
    3. Create a new user and database named catalog. Connect to the db, revoke rights,
    lock down permissions only to user catalog.
        CREATE USER catalog WITH PASSWORD 'password';
        CREATE DATABASE catalog WITH OWNER catalog;
        \c catalog
        REVOKE ALL ON SCHEMA public FROM public;
        GRANT ALL ON SCHEMA public TO catalog;
        \q
        $ exit

### 10. Install Flask
    $ sudo apt-get install python-pip
    $ sudo pip install Flask
    $ sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils

### 11. Clone the Item Catalog application from Github repo
     $ sudo mkdir /var/www/catalog
     $ sudo chown -R grader:grader /var/www/catalog
     $ git clone https://github.com/AvneeshAFC/Item_Catalog.git /var/www/catalog/catalog

### 12. Make a catalog.wsgi file
    1. $ touch catalog.wsgi && nano catalog.wsgi
    2. Add the following lines and save

        import sys
        import logging
        logging.basicConfig(stream=sys.stderr)
        sys.path.insert(0, "/var/www/catalog/")

        from project import app as application
        application.secret_key = 'super_secret_key'
        
    3. Inside the files project.py, database_setup.py, lotsofmenus.py make the following changes for
    correct database connection :
        Change engine = create_engine('sqlite:///restaurantmenuwithusers.db') to
        engine = create_engine('postgresql://catalog:password@localhost/catalog')

### 13. Initialize the db schema and populate the db
    $ cd /var/www/catalog/catalog/
    $ python database_setup.py
    $ python lotsofmenus.py

### 14. Update the Google OAuth settings
    Fill in the client_id and client_secret fields in the file client_secrets.json. Also change the javascript_origins field to the IP address and AWS assigned URL of the host. In this instance that would be: "javascript_origins":["http://ec2-35-154-231-1.ap-south-1.compute.amazonaws.com"]
    These addresses also need to be entered into the Google Developers Console -> API Manager -> Credentials, in the web client under "Authorized JavaScript origins".

### 15. Configure apache2 to serve the app
    $  sudo nano /etc/apache2/sites-available/000-default.conf

    Add the following lines and save

    <VirtualHost *:80>
        ServerName 35.154.231.1
        ServerAdmin itasaini68@gmail.com
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi
        <Directory /var/www/catalog/>
            Order allow,deny
            Allow from all
        </Directory>
        Alias /static /var/www/catalog/catalog/static
        <Directory /var/www/catalog/static/>
            Order allow,deny
            Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>

### 16. Restart apache to launch the application!
    $ sudo service apache2 restart

Source : https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps




