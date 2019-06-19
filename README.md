# Linux-Server-Configuration

Udacity Linux Server Configuration

## AWS Lightsail

First you need a AWS account to login into AWS Lightsail. Then you should create a an instance

Create a server instance via AWS Lightsail (follow the instructions).

IP address : 18.195.216.23

Domain: http://andruska.me

SSH port : 2200

## Configuration steps

### Download your private key and connect with ubuntu user

You have to download your private key in order to connect via ssh to your instance. This key can be downloaded fom AWS Lightsail and it should look like this:

`LightsailDefaultKey-{your-country}.pem` (i.e. `LightsailDefaultKey-eu-central.pem`)

Go into the directory where u place your private keys

```
cd ~/.ssh/
```

and run (you could also rename the file to something more readable (e.g. udacity_rsa.rsa):

```
chmod 600 LightsailDefaultKey-eu-central.pem
```

Now you connect via ssh to your remove server with the `ubunutu` user:

```
ssh -i udacity_key.rsa ubuntu@3.121.183.105
```

You are connected, yay 🎉🎉🎉

### Create a grader user

To ensure that your server is properly protected against hackers your should setup a new user and disable `root` login.

Create a new user called grader:

```
sudo adduser grader
```

Create a `grader` file to handle permissions for this user:

```
sudo nano /etc/sudoers.d/grader
```

Now you can give `grader` sudo permissions by adding this line:

```
grader ALL=(ALL:ALL) ALL
```

For security reasons we also want to have `ssh` connection for `grader`. Therefore we have to generate a key pair with `ssh-keygen` on your local machine.

Just follow this steps here:

1. Create ssh key pair:

```
ssh-keygen
```

2. Name it grader_key

```
grader_key
```

3. Copy your public key:

```
sudo cat ~/.ssh/grader_key.pub
```

4. Connect to your server with `ubuntu` and go to this directory:

```
cd /home/grader/
```

5. Create a new directory called .ssh

```
sudo mkdir .ssh
```

6. Paste the public key to the `authorized_keys` file:

```
sudo nano .ssh/authorized_keys
```

7. and adjust the permissions to:

```
sudo chmod 700 /home/grader/.ssh
```

```
sudo chmod 644 /home/grader/.ssh/authorized_keys
```

7. Change the owner:

```
sudo chown -R grader:grader /home/grader/.ssh
```

8. Enfore key-based `ssh` authentication open this file:

```
sudo nano /etc/ssh/sshd_config
```

**Change config settings:**

`PasswordAuthentication` to `no`

Probably this line is commented in that case just remove the hashtag and change the `Port` to `2200`.

`PermitRootLogin` to `no`

To apply your changes restart it:

```
 sudo service ssh restart
```

The last thing is that you have to go to your AWS Lightsail account and add the `2200` port to allow connections via this port.

### Configure Firewall

To only allow connections for `ssh` (port 2200), `http` (port 80), and `ntp` (port 123) you have to do:

```
sudo ufw default deny incoming
```

```
sudo ufw default allow outgoing
```

```
sudo ufw allow 2200/tcp
```

```
sudo ufw allow 80/tcp
```

```
sudo ufw allow 123/udp
```

```
sudo ufw enable
```

### Connect as grader via ssh

```
ssh -i ~/.ssh/grader_key grader@18.195.216.23 -p 2200
```

### Update packages

It will create a list of packages which can be updated:

```
sudo apt-get update
```

Now you can upgrade this by having your list in place with:

```
sudo apt-get upgrade
```

### Install Apache and mod_wsgi

Apache2

```
sudo apt-get install apache2
```

mod_wsgi

```
sudo apt-get install libapache2-mod-wsgi python-dev
```

Enable mod_wsg:

```
sudo a2enmod wsgi
```

Start your apache2 server

```
sudo service apache2 start
```

You can double check it in your browser:

```
http://18.195.216.23
```

It should show you the default apache2 page.

### Copy your app to your remote server

In my case I wanted to put my data remotely on my server:

```
scp -v -P 2200 -i ~/.ssh/grader_key -r movieflix/ grader@3.121.183.105:~
```

Go to:

```
cd /var/www
```

create a `movieflix` directory:

```
sudo mkdir movieflix
```

Change owner:

```
sudo chown -R grader:grader movieflix
```

then move the `movieflix` folder to:

```
sudo mv /movieflix /var/www/movieflix/
```

Your folder structure should look like this:

```
├── var/www
|   └── movieflix/
|   |    └── movieflix/
|   |    |   |-- populate_db.py
|   |    |   |-- database_setup.py
|   |    |   |-- app.py
|   |    |   └── ...
```

### Install pip and virtualenv

Just install the internet (in the following directory: `/var/www/movieflix/):

```
sudo apt-get install python-pip
```

```
sudo pip install virtualenv
```

```
sudo virtualenv venv
```

Ajust permissions to your `venv` folder:

```
sudo chmod -R 777 venv
```

Activate virtualenv

```
source venv/bin/activate
```

Since your app has `requirements.txt` with all your dependencies you can run:

```
sudo pip install -r movieflix/requirements.txt
```

Install pythons PostgresSql adapter:

```
sudo apt-get install python-psycopg2
```

### Configure your virtual host

Create and open this file:

```
sudo nano /etc/apache2/sites-available/movieflix.conf
```

and paste this Block into the file:

```
<VirtualHost *:80>
   ServerName 18.195.216.23
   ServerAlias andruska.me
   ServerAdmin grader@18.195.216.23
   WSGIDaemonProcess movieflix python-path=/var/www/movieflix:/var/www/movieflix/venv/lib/python2.7/site-packages
   WSGIProcessGroup movieflix
   WSGIScriptAlias / /var/www/movieflix/movieflix.wsgi
   <Directory /var/www/movieflix/movieflix/>
       Order allow,deny
       Allow from all
   </Directory>
   Alias /static /var/www/movieflix/movieflix/static
   <Directory /var/www/movieflix/movieflix/static/>
       Order allow,deny
       Allow from all
   </Directory>
   ErrorLog ${APACHE_LOG_DIR}/error.log
   LogLevel warn
   CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

As you can see, there's not much here. We will customize the items here for our `domain` and add some additional directives. This virtual host section matches any requests that are made on port 80, the default HTTP port.

### Setup the wsgi file

Create and open a new file called `movieflix.wsgi`:

```
sudo nano /var/www/movieflix/movieflix.wsgi
```

Add following code to this file:

```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/movieflix/")

from movieflix import app as application
application.secret_key = 'secret'
```

### Adjust movieflix application

Since we don't have a `localhost` enviroment anymore we need to adjust our application to the real world:

1. Go to your root directory of your application:

```
cd /var/www/movieflix/movieflix
```

2. Rename `app.py` to `__init__.py`

```
sudo mv app.py __init__.py
```

3. Adjust relative path to absolute one for the `client_secrets.json` in the `__init__.py` file:

```
sudo nano __init__.py
```

You need to change your path to:

```
/var/www/movieflix/movieflix/client_secrets.json
```

### Install and configure PostgreSql

Install more stuff from the internet:

```
sudo apt-get install libpq-dev python-dev
```

For more information you can have a look here:

https://packages.debian.org/de/sid/libpq-dev

https://packages.debian.org/de/sid/python-dev

Install PostgreSql:

```
sudo apt-get install postgresql postgresql-contrib
```

Change to the `postgres` user:

```
sudo -u postgres psql
```

Now you can perform database operations.

1. Connect to the database:

```
\c movieflix
```

2. Create a user with a password:

```
CREATE USER movieflix WITH PASSWORD 'movieflix';
```

3. Give your user permission to alter the database:

```
ALTER USER movieflix CREATEDB;
```

4. Create your database `movieflix`:

```
CREATE DATABASE movieflix WITH OWNER movieflix;
```

5. Remove access privileges

```
REVOKE ALL ON SCHEMA public FROM public;
```

6. Grant all permissions to movieflix role:

```
GRANT ALL ON SCHEMA public TO movieflix;
```

7. Close session with PostgreSql:

```
\q
```

Now we need to adjust our application again.

You have to edit these files in order to connect to our postgres database:

`database_setup.py`

`populate_db.py`

`__init__.py`

In these files you have to find this code snippet:

```
engine = create_engine('sqlite:///movie-catalog.db',
                       connect_args={'check_same_thread': False})
```

and replace it with:

```
create_engine('postgresql://movieflix:movieflix@localhost/movieflix')
```

### Activate movieflix virtual host

Now you can enable this virtual host:

```
sudo a2ensite movieflix.conf
```

### Run your application

Now you can setup your database by (please ensure that you are in this directory `/var/www/movieflix/movieflix`):

```
python database_setup.py
```

Populate your database with some data:

```
python populate_db.py
```

Now you can restart apache:

```
sudo service apache2 restart
```

### Debugging

If you're reading this something went wrong right? I received many times this weird

```
Internal 500 Server error
```

Which was not really helpful for me, but there is a way how you can debug your server to find your bug:

```
sudo cat /var/log/apache2/error.log
```
