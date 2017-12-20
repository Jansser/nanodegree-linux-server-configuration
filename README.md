# Linux Server Configuration

A project to Full Stack Web Developer Nanodegree program. The goal here is to set up and secure an Amazon Lightsail baseline installation of Linux server to run a web application. The configuration steps to secure the server, set up the database and deploy the project are below.

## Server details

**IP Address**
18.216.231.41

**SSH port**
2200

**URL to web application**
http://http://18.216.231.41/

## Summary

### Software installed
  * Apache2
  * mod_wsgi
  * PostgreSQL
  * git
  * python3
  * python-pip
  * psycopg2
  * virtualenv
  * SQLAlchemy
  * Flask
  * oauth2client
  * httplib2

#### Secure your server
* Update all currently installed packages.

    `sudo apt-get update`

    `sudo apt-get upgrade`

    * Setup automatic updates

      `sudo apt install unattended-upgrades`

      Edit this file to enable

      `sudo nano /etc/apt/apt.conf.d/20auto-upgrades`

      ```
      APT::Periodic::Update-Package-Lists "1";
      APT::Periodic::Download-Upgradeable-Packages "1";
      APT::Periodic::AutocleanInterval "7";
      APT::Periodic::Unattended-Upgrade "1";
      ```

* Change the SSH port from `22` to `2200`.
  * Configure the Lightsail firewall to allow port `TCP 2200` in the Networking tab.
  * Change the line "Port 22" to "Port 2200".

    `sudo vi /etc/ssh/sshd_config`

  * Restart ssh service.

    `sudo service ssh restart`

* Configure Firewall
    * Set the firewall to block any connections coming in, allow everything outgoing.
    Then allow our `2200` SSH port, `80` port to our web server and `123` NTP port and enable the firewall.
    ```
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 2200/tcp
    sudo ufw allow 80/tcp
    sudo ufw allow 123/udp
    sudo ufw enable
    ```

  * Disable remote root login.

### Give grader access.
  * Create a new user account named grader.

    `sudo adduser grader`

  * Give grader the permission to sudo.
    * Create grader file on sudoers.d

      `sudo touch /etc/sudoers.d/grader`

    * Edit the file to add the permissions.

      `sudo nano /etc/sudoers.d/grader`

      Add this line `grader ALL=(ALL) NOPASSWD:ALL`

  * Create an SSH key pair for grader using the ssh-keygen tool.
    * Run `ssh-keygen` on the local machine to create the key pair.
      create `grader_key` file on `.ssh` directory.

    * Place the public key on the server
      * Create directory `.ssh`
      * Create `authorized_keys` file

        `touch .ssh/authorized_keys`
      * Copy the public key from the local machine `grader_key.pub` and paste in the `authorized_keys` file

        `sudo nano .ssh/authorized_keys`

      * Set permissions of the `.ssh` directory and the `authorized_keys` file

        `sudo chmod 700 .ssh`

        `sudo chmod 644 .ssh/authorized_keys`

    * Disable password authentication
      * Change PasswordAuthentication to no.

        `sudo nano /etc/ssh/sshd_config`

      * Restart ssh servie

        `sudo service ssh restart`

      * Now we can connect with the grader user

        `sudo ssh grader@18.216.231.41 -p 2200 -i .ssh/grader_key`

### Prepare to deploy your project.
  * Configure the local timezone to UTC

    `sudo dpkg-reconfigure tzdata` select 'None of the above' and then UTC

  * Install and configure Apache

    `sudo apt-get install apache2`

  * Install and configure Apache to serve a Python mod_wsgi application

    `sudo apt-get install libapache2-mod-wsgi-py3`

    * Enable wsgi

    `sudo a2enmod wsgi`

  * Install git

    `sudo apt-get install git`

  * Install and configure PostgreSQL

    `sudo apt-get install postgresql postgresql-contrib`

    * Do not allow remote connections

      This is the current default when installing PostgreSQL from the Ubuntu repositories.

      We can double check that no remote connections are allowed by looking in the host based authentication file:

      `sudo nano /etc/postgresql/9.5/main/pg_hba.conf`

      ```
      local   all             postgres                                peer
      local   all             all                                     peer
      host    all             all             127.0.0.1/32            md5
      host    all             all             ::1/128                 md5
      ```

    * Create a new database user named catalog

      `sudo -i -u postgres`

      `psql`

      `CREATE USER catalog WITH CREATEDB PASSWORD 'catalog';`

      `CREATE DATABASE catalog WITH OWNER catalog;`

### Deploy the Item Catalog project
  * Clone and setup your Item Catalog project from the Github repository

    `sudo git clone https://github.com/Jansser/nanodegree-catalog-items.git catalog`

    * Create wsgi file

    `sudo touch catalog.wsgi`

    ```
    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/catalog/")

    from catalog import app as application
    application.secret_key = 'slimShady'
    ```

    * Setup a virtual environment and install web app dependencies

      * Install virtualenv

        `sudo apt-get install python3-pip`

        `sudo pip3 install virtualenv`

      * Create virtual env

        `sudo virtualenv venv --python=python3.5`

      * Change the owner of virtual env folder to allow pip install the dependencies

        `sudo chown -R ubuntu:ubuntu venv`

      * Activate enviroment

        `source venv/bin/activate`

      * Install app dependencies

        `pip install -r requirements.txt`

        `pip install psycopg2`

      * Deactivate the virtual environment

        `deactivate`

      `sudo service apache2 restart`

    * Edit catalog app to change the database from Sqlite3 to PostgreSQL catalog created.
      `sudo nano database.py`

      `sudo nano load_data.py`

      `sudo nano app.py`

      Change the engine to postgres.

      `engine = create_engine("postgresql://catalog:catalog@localhost/catalog")`

    * Setup a virtual host

    `sudo nano /etc/apache2/sites-available/catalog.conf`

    ```
    <VirtualHost *:80>
      ServerName 18.216.231.41
      ServerAdmin admin@18.216.231.41
      WSGIDaemonProcess catalog python-home=/var/www/catalog/venv/lib/python3.5/site-packages
      WSGIProcessGroup catalog

      WSGIScriptAlias / /var/www/catalog/catalog.wsgi
      <Directory /var/www/catalog/>
          Order allow,deny
          Allow from all
      </Directory>

      Alias /static /var/www/catalog/static

      <Directory /var/www/catalog/static/>
        Order allow,deny
        Allow from all
      </Directory>

      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```

    `sudo a2ensite catalog`

    Activate new configuration

    `sudo service apache2 reload`

  * Add `http://18.216.231.41` to Authorized Javascript origins on google project to authorization works.

  * Edit app.py so the server can find the `client_secrets.json`

  `sudo nano app.py`

  ```
  PROJECT_ROOT = os.path.realpath(os.path.dirname(__file__))
  json_url = os.path.join(PROJECT_ROOT, 'client_secrets.json')
  CLIENT_ID = json.load(open(json_url))['web']['client_id']
  ```

  * Fix requests on line 81

  ```
  h.request(url, "GET")[1].decode('utf-8')
  ```

  * Configure app image upload with flask_uploads

    Change the path of flask_uploads and the set the url with the server ip.

    `sudo nano app.py`

    ```
    app.config["UPLOADS_DEFAULT_DEST"] = "/var/www/catalog/static"
    app.config["UPLOADS_DEFAULT_URL"] = "http://18.216.231.41:80/static"
    app.config["UPLOADS_IMAGES_DEST"] = "/var/www/catalog/static"
    app.config["UPLOADS_IMAGES_URL"] = "http://18.216.231.41:80/static"
    ```

    Change the owner of images folder to apache user

    `mkdir static/images`

    `sudo chown www-data:www-data static/images`

## Resources
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps

https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps

https://www.postgresql.org/docs/9.5/static/app-createuser.html

http://modwsgi.readthedocs.io/en/develop/user-guides/quick-configuration-guide.html

https://br.godaddy.com/help/como-alterar-a-porta-ssh-para-seu-servidor-linux-7306

https://wiki.apache.org/httpd/13PermissionDenied
