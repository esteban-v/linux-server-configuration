# Linux Server Configuration
This is the configuration procedure to enable a linux server using Amazon Lightsail with an Ubuntu 16.04 LTS instance running.
* Current URL to web application: <http://18.217.104.67.xip.io> (once this project is reviewed this url wil be disabled - april 2018)
* IP address and SSH port to be used by reviewer: **IP** 18.217.104.67 - **PORT** 2200

## Securing server
1. Update all currently installed packages:
   ```
   sudo apt-get update
   sudo apt-get upgrade
   ```
2. Change the SSH port from **22** to **2200** and configure other ports needed
   * On your Lightsail Firewall enable port **2200** for SSH, port **80** for HTTP and port **123** for NTP.
   * Run `sudo nano /etc/ssh/sshd_config` and look for title `# What ports, IPs and protocols we listen for` replace Port 22 for Port 2200, save changes.
3. Configure the Uncomplicated Firewall (UFW)
    To configure your UFW, run the following commands:
    ```
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 2200/tcp
    sudo ufw allow www
    sudo ufw allow 123/udp
    sudo ufw enable
    sudo ufw status
    ```

## Creating a user named grader and giving access to this user
1. To create new user run:
    `sudo adduser grader` you'll be prompted to assign a password for this user.
2. Give grader user the permission to sudo:
    `sudo nano /etc/sudoers.d/grader`
    add to this file the following content `grader ALL=(ALL) NOPASSWD:ALL` and save changes.
3.  Create an SSH key pair for grader using the ssh-keygen tool.
    * On your local machine create a key pair running the command `ssh-keygen`, you’ll be asked to assign a file name for the key pair and a passphrase (in this case the word **grader** was used).
    * Back on your ssh session switch to the grader user running `su - grader`, you'll be asked to provide this user's password then run the following:
        ```
        mkdir .ssh
        touch .ssh/authorized_keys
        nano .ssh/authorized_keys
        ```
        Explore and copy the contents of the public key (file with .pub extension) your created with the ssh-keygen tool and paste that content (long string) into **authorized_keys**, make sure it's all one big line when you paste it, save changes.
        
    * Finally set permissions to the dir and file you just created:
        ```
        chmod 700 .ssh
        chmod 644 .ssh/authorized_keys
        ```
    * You can now exit your ssh session and login as user **grader** using the created private key.

## Preparing to deploy a Flask web application
1. Configure the local timezone to UTC, running the following:
    `sudo timedatectl set-timezone UTC`
2. Install and configure Apache to serve a Python mod_wsgi application.
    * Run `sudo apt-get install apache2` to install apache.
    * Install mod_wsgi `sudo apt-get install libapache2-mod-wsgi`
3. Install and configure PostgreSQL:
    `sudo apt-get install postgresql`
    * Make sure remote connections are not allowed exploring the following file:
    `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`
    * Login as postgres user and access the PostgreSQL shell:
        ```
        sudo su - postgres
        psql
        ```
    * Create a new database user named catalog that has limited permissions to your catalog application database:
        ```
        postgres=# CREATE DATABASE application;
        postgres=# CREATE USER catalog;
        postgres=# ALTER ROLE catalog WITH PASSWORD 'password';
        postgres=# GRANT ALL PRIVILEGES ON DATABASE application TO catalog;
        ```
    * You can now exit PostgreSQL shell and logout:
        ```
        postgres=# \q
        exit
        ```

## Deploy your Flask web application
* Update all your application files were a database connection is made, replacing `engine = create_engine('sqlite:///your_database.db')` for `engine = create_engine('postgresql://catalog:password@localhost/your_database')`
* Install Flask and all required dependencies:
    ```
    sudo apt-get install python-flask
    sudo apt-get install python-pip
    sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils
    sudo pip install requests flask-sqlalchemy bleach
    ```
    
* Create a dir for your application inside `/var/www`
    ```
    cd /var/www
    sudo mkdir catalog_app
    ```
* If you had a Github repository for your application you can now clone that repository into the dir you just created or if you have your application files on your local machine (which was my case) you can upload them to the same dir using :
    `scp -i /path/to/.ssh/private_key path/to/local/file <username>@<IP address>:/destination/path`
* After you have all your files in place use your corresponding file (database_setup.py) to setup your database:
`python database_setup.py`
* Create myapp.wsgi file to serve your application inside /var/www/catalog_app
    ```
    cd /var/www/catalog_app
    sudo nano myapp.wsgi
    ```
    Write the following content to that file:
    ```
    import sys
    import logging
    sys.path.insert(0, "/var/www/catalog_app/")
    logging.basicConfig(stream=sys.stderr)
    
    from application import app as application
    application.secret_key = 'super_secret_key'
    ```
* Create your Apache configuration file and enable your application
    Create file `sudo nano /etc/apache2/sites-available/flask_app.conf`
    Write the following content to that file:
    ```
    <VirtualHost *>
        ServerName flask_example.com
    
        WSGIDaemonProcess application user=ubuntu threads=5
        WSGIScriptAlias / /var/www/catalog_app/myapp.wsgi
    
        <Directory /var/www/catalog_app>
            WSGIProcessGroup application
            WSGIApplicationGroup %{GLOBAL}
            Order deny,allow
            Allow from all
        </Directory>
    </VirtualHost>
    ```
      
    Enable your application running:
    ```
    sudo a2dissite 000-default.conf
    sudo a2ensite flask_app.conf
    ```
    Restart Apache running `sudo service apache2 restart`

## References
   * <http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/>
   * <http://alonavarshal.com/flask/aws/python/2017/10/13/deploying-a-flask-web-app-on-lightsail-aws.html>
