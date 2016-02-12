## Linux Configuration

### Access: 

**IP:** 52.46.46.127
**SSH:** Port 2200
**Site URL:** http://ec2-52-36-46-127.us-west-2.compute.amazonaws.com/

### Summary: 

Login via ssh as root through port 2200.
Updated system
```
apt-get update
apt-get upgrade
```
Changed [Time Zone to UTC][2]
```
dpkg-reconfigure tzdata
```
Set password authorization in /etc/ssh/sshd_config to yes
Changed ssh port to 2200
Restarted the ssh server
```
service ssh restart
```

Via root access, user grader was added and granted sudo priveleges
through the addition of a grader file to /etc/sudoers.d
Contents of /etc/sudoers.d/grader:
```
grader ALL=(ALL:ALL)
```
Logged in as grader locally using the password set when user grader 
was created.
```
ssh grader@127.0.0.1 -p 2200
```

Used the grader account to create the .ssh directory
and .ssh/authorized_keys file, setting appropriate permissions
for the two.
```
mkdir .ssh
touch .ssh/authorized_keys
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```

I copied the key permitting login via my host computer from the root
.ssh/authorized_keys file to that belonging to grader.

I also Generated a new rsa key for login locally to the grader 
account (I realize this was not necessary but I wanted to make sure 
I understood all of the elements involved with ssh login). This key 
was saved to the root .ssh directory as grader_rsa with public as 
grader_rsa.pub. The public key was then copied to the 
.ssh/authorized_keys file of user grader. Passphrase was set as 
grader.
```
ssh-keygen
```

Modified /etc/ssh/sshd_config, setting password authentication back 
to no and restarted the ssh service. Verified that grader could be 
logged into both locally and from host machine.

[Sudo: unable to resolve host][1] issue in AWS necessitated the 
addition of the following to /etc/hosts 
```
127.0.1.1 ip-10-20-0-127
```

#### Firewall setup

Used ufw as the primary firewall, implemented built-in limits on the 
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

Also installed [fail2ban][8] as an extra line of defense. I kept most of 
the default settings in place and turned on the service to watch 
apache as well. Ssh monitoring was changed from port 22 to 2200.

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
Created catalog.wsgi in /var/www/html. 
Cloned Catalog project into /var/www
Modified the [SQLAlchemy URI][9] in the Catalog application to use the  
postgres "Catalog" database via user catalog.
```
SQLALCHEMY_DATABASE_URI = 'postgresql://catalog:catalog@localhost/Catalog'
```

Followed the steps outlined in the Catalog app README in order to 
create the first migrations to the postgres database and seed the 
Category list.

Modified catalog.wsgi to call the create_app method in catalog
in order to serve the application.

Restarted the apache server
```
sudo service apache2 restart
```
Verified the application was serving.

#### Postgres setup
Installation:
```
sudo apt-get install postgresql
```
I created a new user by the name of catalog after making a 
modification to the postgres pg_hba.conf file allowing login locally
via md5 instead of peer. I kept running into peer authentication
difficulties until this post brought me to that solution. The 
password for user catalog was set as catalog. 

As su postgres I created the "Catalog" database and [placed restrictions][3]
limiting the role catalog to SELECT, INSERT, UPDATE, and DELETE.
Restrictions were placed on all users to prevent 
tampering with the public schema. Since all tables in Catalog were 
created by postgres, they cannot be dropped except by postgres.

Remote access to postgres [seemed to be disabled by default][4], so 
no changes were made.


#### Cron Scripts and automation.

For basic security automation I used [unattended upgrades][5] to
allow automatic stable upgrades.
```
dpkg-reconfigure unattended-upgrades
```

To provide for regular updating/upgrading of apt-get I used
[crontab][6]. This was done as root with a schedule set to run 
every Sunday at 3:05 am.

#### Monitoring

System monitoring is performed via [glances][7].
This required the installation of the python-dev and build-essential
packages via apt-get
To run, simply type:
```
glances
```

#### Notes

A great number of the commands were performed as root on this 
machine, this is not something I would normally ever do. I
appreciate the very valid reasons why this is a bad idea, however
with this project being very temporary I chose to go this route. 

Normally I would also have run the installations for the Catalog
website within a virtualenv instead of directly to the machine. 
However, with the terminal nature of the project I chose to
not do so. If this was a machine I was going to use in the future,
I would implement the use of a virtual environment without 
hesitation.


[1]: https://forums.aws.amazon.com/thread.jspa?messageID=699718
[2]: https://hlp.ubuntu.com/community/UbuntuTime
[3]: http://dba.stackexchange.com/questions/33943/granting-access-to-all-tables-for-a-user
[4]: https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
[5]: https://help.ubuntu.com/community/AutomaticSecurityUpdates
[6]: https://help.ubuntu.com/community/CronHowto
[7]: http://askubuntu.com/questions/293426/system-monitoring-tools-for-ubuntu
[8]: https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04
[9]: http://killtheyak.com/use-postgresql-with-django-flask/
