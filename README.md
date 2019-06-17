# Linux-Server-Configuration
Udacity Linux Server Configuration

## AWS Lightsail

First you need a AWS account to login into AWS Lightsail. Then you should create a an instance

Create a server instance via AWS Lightsail (follow the instructions).

IP address : 3.121.183.105

SSH port : 2200


## Configuration steps

### Private key
Download your private key `LightsailDefaultKey-{your-country}.pem` (i.e. `LightsailDefaultKey-eu-central.pem` to enable the connection via ssh.

Go into the directory where u place your private keys
```
cd ~/.ssh/
```

and run (you could also rename the file to something more readable (e.g. udacity_rsa.rsa):

```
chmod 600 LightsailDefaultKey-eu-central.pem
```
Now you connect via ssh to your remove server:

```
ssh -i udacity_key.rsa ubuntu@3.121.183.105
```

You are connected, yay...

### Update your stuff

To ensure to have the latest packages installed on your server run following commands:

```
sudo apt-get update
sudo apt-get upgrade
```

### Change default port 22 to 2200

Since you established a connection to your remote server you can now do some security by changing the ssh (`22`) port to `2200`

```
sudo nano /etc/ssh/sshd_config
```

Change port from `22` to `2200`


Restart SSH with `sudo service ssh restart`


### Step 5: Configure the Uncomplicated Firewall (UFW)

Configure the default firewall for Ubuntu to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
  ```
  sudo ufw status                  # The UFW should be inactive.
  sudo ufw default deny incoming   # Deny any incoming traffic.
  sudo ufw default allow outgoing  # Enable outgoing traffic.
  sudo ufw allow 2200/tcp          # Allow incoming tcp packets on port 2200.
  sudo ufw allow www               # Allow HTTP traffic in.
  sudo ufw allow 123/udp           # Allow incoming udp packets on port 123.
  sudo ufw deny 22                 # Deny tcp and udp packets on port 53.
  ```

Turn UFW on: `sudo ufw enable`. The output should be like this:

### Add grader user
Usally you setup a server and add some users to it and give them only the permessions they. Another important thing is you also deactivate the default root login to protect your server from crazy hackers.

1. Create a user called `grader`
```
sudo adduser grader
```

Create a directory to enable sudo right for the `grader` user
```
sudo nano /etc/sudoers.d/grader
```

Now just add the following line to enable sudo access:

```
grader ALL=(ALL:ALL) ALL
```
Close and save the file.



You need to create a ssh keypair to enable ssh connection for your user. Therefore I use `ssh-keygen` on my local machine. It automatically generates for your a keypair for your local enviroment and one key for your server.

```
ssh-keygen
```

follow the instructions and and setup a name for your key: `udacity_key`

copy your public key (e.g. `udacity_key.pub`) and connect to your server:

Create a new directory called ~/.ssh (mkdir .ssh)

Create a new directory:

```
mkdir .ssh
```

Paste your public key into the `authorized_keys` file:
```
sudo nano ~/.ssh/authorized_keys
```

To secure even more you can disable Password authentication to allow only connections via ssh for this user:

```
sudo nano /etc/ssh/sshd_config
```

and set `PasswordAuthentication` to `no`

Now you should do a restart and grab a coffee:

```
sudo service ssh restart
```

### Connect with grader

Now you can connect via ssh with our new user `grader` on the remove server

```
ssh -i ~/.ssh/udacity_key -p 2200 grader@3.121.183.105
```

### Install and configure Apache to serve a Python mod_wsgi application

While logged in as `grader`, install Apache: `sudo apt-get install apache2`.
Enter public IP of the Amazon Lightsail instance into browser.

Run `sudo apt-get install python-setuptools libapache2-mod-wsgi` to install mod-wsgi module


### Install and configure PostgreSQL

- While logged in as `grader`, install PostgreSQL:
 `sudo apt-get install postgresql`.
- PostgreSQL should not allow remote connections. In the  `/etc/postgresql/9.5/main/pg_hba.conf` file, you should see:
  ```
  local   all             postgres                                peer
  local   all             all                                     peer
  host    all             all             127.0.0.1/32            md5
  host    all             all             ::1/128                 md5
  ```

- Switch to the `postgres` user: `sudo su - postgres`.
- Open PostgreSQL interactive terminal with `psql`.
- Create the `catalog` user with a password and give them the ability to create databases:
  ```
  postgres=# CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
  postgres=# ALTER ROLE catalog CREATEDB;
  ```

- List the existing roles: `\du`. The output should be like this:
  ```
                                     List of roles
   Role name |                         Attributes                         | Member of 
  -----------+------------------------------------------------------------+-----------
   catalog   | Create DB                                                  | {}
   postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
  ```

- Exit psql: `\q`.
- Switch back to the `grader` user: `exit`.
- Create a new Linux user called `catalog`: `sudo adduser catalog`. Enter password and fill out information.
- Give to `catalog` user the permission to sudo. Run: `sudo visudo`.
- Search for the lines that looks like this:
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  ```

- Below this line, add a new line to give sudo privileges to `catalog` user.
  ```
  root    ALL=(ALL:ALL) ALL
  grader  ALL=(ALL:ALL) ALL
  catalog  ALL=(ALL:ALL) ALL
  ```

- Save and exit using CTRL+X and confirm with Y.
- Verify that `catalog` has sudo permissions. Run `su - catalog`, enter the password, run `sudo -l` and enter the password again. The output should be like this:

  ```
  Matching Defaults entries for catalog on ip-172-26-13-170.us-east-2.compute.internal:
      env_reset, mail_badpass,
      secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
  
  User catalog may run the following commands on ip-172-26-13-170.us-east-2.compute.internal:
      (ALL : ALL) ALL
  ```

- While logged in as `catalog`, create a database: `createdb catalog`.
- Run `psql` and then run `\l` to see that the new database has been created. The output should be like this:
  ```
                                    List of databases
     Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
  -----------+----------+----------+-------------+-------------+-----------------------
   catalog   | catalog  | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
   postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
   template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
             |          |          |             |             | postgres=CTc/postgres
   template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
             |          |          |             |             | postgres=CTc/postgres
  (4 rows)
  ```
- Exit psql: `\q`.
- Switch back to the `grader` user: `exit`.

### Copy your app to your remote server
In my case I wanted to put my data remotely on my server:

```
scp -v -P 2200 -i ~/.ssh/grader_key -r movieflix/ grader@3.121.183.105:~
```

then I moved the `movieflix` folder to:

```
sudo mv /movieflix /var/www/
```

From the `/var/www` directory, change the permissions of the movieflix directory to grader using: sudo chown -R grader:grader movieflix/.

### Adjust your app

You have to rename your `app.py` (or something else) to `__init__.py`


Remove params of your `run()` fn withn `app.py`:
app.run()

Replace `sqlite` with our `postgresql` database:
# engine = create_engine("sqlite:///movieflix.db")
engine = create_engine('postgresql://catalog:PASSWORD@localhost/movieflix')

## Install all requirements for your application

```
sudo apt-get install python-pip
```

Install venv
```
sudo virtualenv venv
```
and activate it:
```
source venv/bin/activate
```
Adjust permissions for the `venv` folder:

```
sudo chmod -R 777 venv
```

```
pip install Flask httplib2 request oauth2client sqlalchemy python-psycopg2
```
