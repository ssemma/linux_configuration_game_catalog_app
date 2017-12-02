# Linux Server Configuration
This project takes a clean install of Ubuntu 16.04 and configures it to host a web application Game Catalog.
These configuration secures the server from various forms of attacks.

#IP address and SSH
1. [http://18.217.95.198](http://18.217.95.198)
or [http://ec2-18-217-95-198.us-east-2.compute.amazonaws.com](http://ec2-18-217-95-198.us-east-2.compute.amazonaws.com)

SSH - Port 2200

# Steps of Configuration
## Get server
1. Use [Amazon Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home/instances)
2. Download the private key from your account
3. Login to server using 
    ```
        ssh ubuntu@18.217.95.198 -p 22 -i **the location of the private key you downloaded**
    ```
## Secure server
1. Update all currently installed packages
```
    sudo apt-get update
    sudo apt-get upgrade
```
2. Change the SSH port from 22 to 2200
```
    sudo vim /etc/ssh/sshd_config
```
In the file, change port 22 to 2200.
Restart the ssh service
```
    sudo service ssh restart
```
3.Configure the uncomplicated firewall
```
    sudo ufw status
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow ssh
    sudo ufw allow 2200/tcp
    sudo ufw allow www
    sudo ufw allow 123/udp
    sudo ufw deny 22
    sudo ufw enable
    sudo ufw status
```
Make sure on lightsail website, the firewall allows port 2200, port 80 and port 123. And not allow port 22.
## Give grader access
1. Create a new user account
```
    sudo adduser grader
```
create a password and fill in the information for the grader
2. Give grader the permission to sudo
```
    sudo touch /etc/sudoers.d/grader
    sudo vim /etc/sudoers.d/grader
```
In the file, write
```
    grader ALL=(ALL) NOPASSWD=ALL
```
switch user
```
    su grader
```
test if the grader can or cannot sudo
```
    sudo cat /etc/passwd
```
3.Create an SSH key pair for grader using the ssh-keygen tool
In the local machine,
```
    ssh-keygen
    /c/Users/Shanshan/.ssh/[name]
    cat /c/Users/Shanshan/.ssh/[name].pub
```
copy the [name].pub file
In grader,
```
    mkdir .ssh
    nano .ssh/authorized_keys   #and paste the pub file
    chmod 700 .ssh
    chmod 644 .ssh/authorized_keys
```
Now, you can login as
```
    ssh grader@18.217.95.198 -p 2200 -i ** the location of the key**
```
4. Check to secure the web by not allowing password authentication and no root login
```
    sudo vim /etc/ssh/sshd_config
```
In the file, check there is no password authentication allow and 
change **PermitRootlogin** to no.
and then restart ssh service
```
    sudo service ssh restart
```
## Prepare to deploy project
1. Configure the local timezone to UTC
```
    sudo dpkg-reconfigure tzdata
```
and select UTC
Third party source: 
[askubuntu](https://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt)
2. Install and configure Apache to serve a python mod_wsgi application
```
    sudo apt-get install apache2
    sudo apt-get install libapache2-mod-wsgi python-dev
    sudo a2enmod wsgi
    sudo service apache2 start
```
3. Install and configure PostgreSQL
```
    sudo apt-get install postgresql
```
4. Do not allow remote connections
```
    sudo vim /etc/postgresql/9.5/main/pg_hba.conf
```
In the file, check the setup is the same in the following
```
    local   all             postgres                                peer
    local   all             all                                     peer
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5
```
Third party source:
[digitalocean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
5.Create a new database user named catalog that has limited permission to your catalog application database
```
    sudo -i -u postgres
    psql
    CREATE ROLE catalog WITH LOGIN;
    ALTER ROLE catalog WITH CREATEDB;
    CREATE DATABASE catalog WITH OWNER catalog;
    \c catalog
    REVOKE ALL ON SCHEMA public FROM public;
    GRANT ALL ON SCHEMA public TO catalog;
    \password catalog   # create password
    \du
```
It should show up the following:
```
                              List of roles
 Role name |                   Attributes                   | Member of 
-----------+------------------------------------------------+-----------
 catalog   | Create DB                                    | {}
 postgres  | Superuser, Create role, Create DB, Replication | {}
```

Third party source:
[digital ocean](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
[digital ocean](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)
6. Install git
```
    sudo apt-get install git
```
7. Install pgloader
```
    sudo apt-get install pgloader
    pgloader gamedatabasewithusers.db postgresql://catalog:12345@localhost/catalog
```
Change __init__.py, lotsofgames.py and database_setup.py's 
```
    engine = create_engine('sqlite:///gamedatabasewithusers.db')
```
to 
```
    engine = create_engine('postgresql://catalog:12345@localhost/catalog')
```
Third party source:
[sqlalchemy](http://docs.sqlalchemy.org/en/latest/core/engines.html)
[pgloader](https://github.com/dimitri/pgloader)
[udacity forum](https://discussions.udacity.com/t/converting-from-sqlite-to-postgres/246787)
## Deploy game catalog project
1. Clone and setup the project
```
    cd /var/www/
    sudo mkdir game
    sudo chown -R grader:grader game
    cd game
    git clone https://github.com/ssemma/game_catalog_webapp.git game
    cd game
    mv application.py __init__.py
    nano __init__.py
```
In the __init__.py,
change the 
```
app.run(host='0.0.0.0', port=8000) 
```
to 
```
app.run()
```
2. Install Flask, pip and virtualenv
```
    sudo apt-get install python-pip
    sudo pip install virtualenv
    sudo virtualenv Virtenv
    source Virtenv/bin/activate
    sudo pip install Flask
```
3. Install modules that can make the app run normally
```
    sudo pip install httplib2
    sudo pip install oauth2client
    sudo pip install sqlalchemy
    sudo pip install psycopg2
    sudo pip install requests
    sudo pip install Flask_HTTPAuth
    sudo pip install sqlalchemy_utils
    
```
4. In __init__.py,
Change the following line 
```
    CLIENT_ID = json.loads(
                       open('client_secrets.json',
                            'r').read())['web']['client_id']
```
to 
```
    APP_PATH = '/var/www/game/game/'
    CLIENT_ID = json.loads(
                       open(APP_PATH + 'client_secrets1.json',
                            'r').read())['web']['client_id']
```
Go to google developer account,
 
edit the game app oauth 2.0 client_id,
add http://18.217.95.198
http://ec2-18-217-95-198.us-east-2.compute.amazonaws.com
to authorized javascript origins
add http://ec2-18-217-95-198.us-east-2.compute.amazonaws.com/login
http://ec2-18-217-95-198.us-east-2.compute.amazonaws.com/gconnect
http://ec2-18-217-95-198.us-east-2.compute.amazonaws.com/gconnect
to the authorized redirect URIs.
5. check the app is running
```
    sudo python /var/www/game/game/__init__.python
```
The following will show up
```
    * Running on http://127.0.0.1:8000/ (Press CTRL+C to quit)
```
6. deactivate the virtual environment
```
    deactivate
```
7. create a virtual host file
```
    sudo nano /etc/apache2/sites-available/game.conf
```
The game.conf should be similar to below
```
<VirtualHost *:80>
        ServerName 18.217.95.198
        ServerAdmin admin@18.217.95.198
        WSGIScriptAlias / /var/www/game/game.wsgi
        <Directory /var/www/game/game/>
            Order allow,deny
            Allow from all
        </Directory>
        Alias /static /var/www/game/game/static
        <Directory /var/www/game/game/static/>
            Order allow,deny
            Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Enable the virtual host
```
    sudo a2ensite game
```
Inside of /var/www/game/ directory, create a game.wsgi file
```
    sudo nano /var/www/game/game.wsgi
```
The game.wsgi should look like below
```
    #!/usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/game/")

    from Flask import app as application
    application.secret_key = 'super_secret_key'
```
disable default page
```
    a2dissite 000-default.conf
```

Apply changes to Apache2
```
    sudo apachectl restart
```
Third party source:
[Reverse DNS lookup](https://remote.12dt.com/lookup.php)
[Udacity forum for converting database](https://discussions.udacity.com/t/converting-from-sqlite-to-postgres/246787)
[Udacity forum for oath error](https://discussions.udacity.com/t/oath-error-origin-mismatch/221485/3)
[Deploy app](https://devops.profitbricks.com/tutorials/deploy-a-flask-application-on-ubuntu-1404/)
[digital ocean](https://www.digitalocean.com/community/questions/block-default-apache-page-on-ubuntu-14-04)
[udacity linux server configuration course](https://classroom.udacity.com/nanodegrees/nd004/parts/ab002e9a-b26c-43a4-8460-dc4c4b11c379)



