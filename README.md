# Linux Server Configuration

## Project Overview
> Basic deployment steps for catalog flask application with OAuth support.

## Additional details
- FQDN: 13.250.9.102.xip.io
- SSH Port: 2200

## Configuration Steps
0. Buy Short term server to get complete access and deploy application for everyone
> Used LightSail 512MB RAM Ubuntu 18.04 Server, there are many many opitons though

1. Allow Port 2200 via webconsole, Services such as LightSail or AWS have this option

2. Update packages
```
sudo apt-get update && sudo apt-get upgrade 
sudo apt-get install unattended-upgrades

```
3. Install tools and services
```
$ apt-get install python3 python3-dev python3-pip postgresql-server postgresql-server-dev-10 postgresql-contrib ntp apache2 git libapache2-mod-wsgi-py3 tmux 
```

4. Setup Database
```
$ systemctl start postgresql
$ su - postgresql
$ psql
postgres=# CREATE USER catalog WITH PASSWORD 'password';
postgres=# CREATE DATABASE catalog WITH OWNER catalog;
postgres=# \c catalog;
postgres=# REVOKE ALL ON SCHEMA public FROM public;
postgres=# GRANT ALL ON SCHEMA public TO catalog;
postgres=# \q
$ exit
```

5. Edit PG_HBA file to allow db access based on password authentication
```
$ sudo echo "local catalog catalog md5" >> /etc/postgresql/10/main/pg_hba.conf
```

6. Create `grader` user and add to sudoers
```
$ sudo adduser grader
$ sudo echo "grader ALL=(ALL:ALL) ALL" > /etc/sudoers.d/grader
```

7. Create ssh key using ssh-keygen on local system and copy newly generated public key to deployment server
```
$ ssh-keygen -t rsa
$ ssh-copy-id -i newly-generated-key.pub grader@xxx.xxx.xxx.xxx
```

8. Check if able to ssh using keys, if the following command executes then key based access is working
```
$ ssh -i newly-generated-private-key grader@xxx.xxx.xxx.xxx -- date
Sun Nov  3 01:37:10 UTC 2019
```

9. Change SSH Port number, deny passowrd based logins and disable root login
> Edit /etc/ssh/sshd_config and Modify followings
```
Port 2200
PasswordAuthentication no
PermitRootLogin no
```

10. Set timezone to UTC
```
$ sudo timedatectl set-timezone UTC
```

11. Restart NTP and SSH services
```
$ sudo systemctl restart ntp
$ sudo systemctl restart ssh
```

12. Install python packages
```
pip3 install flask sqlalchemy sqlalchemy_utils psycopg2 requests oauth2client httplib2
```

13. Clone Catalog project to `/var/www/` directory
```
$ cd /var/www/
$ mkdir catalog
$ sudo chown -R grader:grader /var/www/catalog
$ git clone https://github.com/rubinsaifi/udacity-catalog-project.git catalog
```

14. Create WSGI file and set apache2 configurations
> Content of proj.wsgi
```
#!/usr/bin/env python3
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog")

from project import app as application
application.secret_key = 'Qa#f}Ww|]+6FuYc_web[#yV'
```
> Enable WSGI mod
```
$ sudo a2enmod wsgi
```
> > Apache2 Configuration changes in `/etc/apache2/sites-enabled/000-default.conf`
```
<VirtualHost *:80>
	ServerName 13.250.9.102
	ServerAdmin webmaster@localhost
	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
	WSGIScriptAlias / /var/www/catalog/proj.wsgi
	<Directory /var/www/catalog>
		Require all granted
	</Directory>
	Alias /static /var/www/catalog/static
</VirtualHost>
```

15. Setup Uncomplicated Firewall
```
$ sudo ufw disable
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 123/udp
$ sudo ufw allow www
$ sudo ufw allow http
$ sudo ufw enable
```

16. Setup database connector in `project.py`
> Modify connector and use username/password created for database catalog
```
engine = create_engine('postgresql://user:password@localhost/catalog')
```
17. Populate database tables
```
$ python3 database_setup.py
$ python3 lotsofmenus.py
```

18. Restart apache service
```
$ sudo systemctl restart apache2
```