## Linux Configuration

This is a basic reference guide for securing the ports on Ubuntu, I utilize
the Catalog app as a reference, which can be uploaded to S3 and allocated to the instance from there.

NOTE :
- Requires allocation of an EC2 instance of Ubuntu Server. Instructions not included
- Ensure that the IAM security role allocated to the instance has the appropriate permissions to permit ssh, http, and https.

### Summary: 

1. Login via ssh as root through port 2200.
2. Updated system
```
apt-get update
apt-get upgrade
```
3. Changed [Time Zone to UTC][2]
```
dpkg-reconfigure tzdata
```
4. Set password authorization in /etc/ssh/sshd_config to yes
5. Change ssh port to 2200
6. Restart the ssh server
```
service ssh restart
```

6. Via root access, add user grader and grant sudo priveleges
through the addition of a grader file to /etc/sudoers.d
Contents of /etc/sudoers.d/grader:
```
grader ALL=(ALL:ALL)
```
7. Log in as grader locally using the password set when user grader 
was created.
```
ssh grader@127.0.0.1 -p 2200
```

8. Use the grader account to create the .ssh directory
and .ssh/authorized_keys file, setting appropriate permissions
for the two.
```
mkdir .ssh
touch .ssh/authorized_keys
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```

9. Copy the key permitting login via host computer from the root
.ssh/authorized_keys file to that belonging to grader.

10. Generate a new rsa key for login locally to the grader 
account. This key should be saved to the root .ssh directory as grader_rsa with public as 
grader_rsa.pub. The public key is then copied to the 
.ssh/authorized_keys file of user grader. Set example passphrase as 
*grader*.
```
ssh-keygen
```

11. Modify /etc/ssh/sshd_config, setting password authentication back 
to no and restart the ssh service. Verify that grader can be 
logged into both locally and from host machine.

[Sudo: unable to resolve host][1] issue in AWS may necessitate the 
addition of the following to /etc/hosts 
```
127.0.1.1 ip-10-20-0-127
```

#### Firewall setup

12. Using ufw as the primary firewall, implement built-in limits on the 
use of ssh on port 2200 to block IPs with more than 6 login 
attempts.
```
sudo ufw deny incoming
sudo ufw allow outgoing
sudo ufw deny 22
sudo ufw limit 2200/tcp
sudo ufw allow ntp
sudo ufw allow www
sudo ufw enable
sudo ufw status numbered
```

13. Install [fail2ban][8] as an extra line of defense. May keep most of 
the default settings in place and turn on the service to watch 
apache. Ssh monitoring was changed from port 22 to 2200.

### Installing Catalog/Setting up site
Use the [Catalog](https://github.com/whiskeyromeo/Catalog) repository to get the site running. 

#### Apache/Site setup
Installation:
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi
sudo apt-get install python-setuptools
sudo apt-get install python-psycopg2
sudo apt-get install git
sudo apt-get install python-pip
```
1. Create catalog.wsgi in /var/www/html. 
2. Clone Catalog project into /var/www
3. Modify the [SQLAlchemy URI][9] in the Catalog application to use the  
postgres "Catalog" database via user catalog.
```
SQLALCHEMY_DATABASE_URI = 'postgresql://catalog:catalog@localhost/Catalog'
```

4. Follow the steps outlined in the Catalog app README in order to 
create the first migrations to the postgres database and seed the 
Category list.

5. Modify catalog.wsgi to call the create_app method in catalog
in order to serve the application.

6. Restart the apache server
```
sudo service apache2 restart
```
7. Verify the application was serving.

#### Postgres setup
Installation:
```
sudo apt-get install postgresql
```
1. Create a new user by the name of catalog after making a 
modification to the postgres pg_hba.conf file allowing login locally
via md5 instead of peer. Set the 
password for user catalog was as *catalog*. 

2. As su postgres, create the "Catalog" database and [place restrictions][3]
limiting the role catalog to SELECT, INSERT, UPDATE, and DELETE.
Add restrictions to all other users in oder to prevent 
tampering with the public schema. Since all tables in Catalog are 
created by postgres, they cannot be dropped except by postgres.


Remote access to postgres [is disabled by default][4]


#### Cron Scripts and automation.

1. For basic security automation use [unattended upgrades][5] to
allow automatic stable upgrades.
```
dpkg-reconfigure unattended-upgrades
```

2. To provide for regular updating/upgrading of apt-get use
[crontab][6]. This must be done as root, in the repo I did this with a schedule set to run 
every Sunday at 3:05 am.

#### Monitoring

System monitoring is performed via [glances][7].

This requires the installation of the python-dev and build-essential
packages via apt-get
To run, simply type:
```
glances
```

#### Notes

Any implementation of this done on a local machine [should utilize either virtualenv or anaconda.][10]
This was done some time ago on AWS, so things have changed a bit. 


[1]: https://forums.aws.amazon.com/thread.jspa?messageID=699718
[2]: https://hlp.ubuntu.com/community/UbuntuTime
[3]: http://dba.stackexchange.com/questions/33943/granting-access-to-all-tables-for-a-user
[4]: https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
[5]: https://help.ubuntu.com/community/AutomaticSecurityUpdates
[6]: https://help.ubuntu.com/community/CronHowto
[7]: http://askubuntu.com/questions/293426/system-monitoring-tools-for-ubuntu
[8]: https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04
[9]: http://killtheyak.com/use-postgresql-with-django-flask/
[10]: https://pythontips.com/2013/07/30/what-is-virtualenv/
