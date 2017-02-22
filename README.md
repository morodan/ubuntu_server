#How To Setup a Ubuntu Server with Apache/PHP/FTP

##we have access on server via ssh, so how to install from zero an Ubuntu server is out of our scope here

##make first an update 

`sudo su`

`apt-get update`

`apt-get upgrade`

`reboot`

##you must disable AppArmor

```
service apparmor stop
update-rc.d -f apparmor remove
apt-get remove apparmor apparmor-utils
```

##Synchronize the System Clock
It is a good idea to synchronize the system clock with an NTP server

`apt-get -y install ntp ntpdate`

##Install MariaDB, rkhunter and binutils

`apt-get install mariadb-client mariadb-server openssl rkhunter binutils`

Now we set a root password in MariaDB. Run: `mysql_secure_installation`

You will be asked these questions:

```
Enter current password for root (enter for none): <-- press enter
Set root password? [Y/n] <-- y
New password: <-- Enter the new MariaDB root password here
Re-enter new password: <-- Repeat the password
Remove anonymous users? [Y/n] <-- y
Disallow root login remotely? [Y/n] <-- y
Reload privilege tables now? [Y/n] <-- y
```

Then we restart MariaDB: `service mysql restart`

##Install Apache, PHP, SuExec, Pear, and mcrypt

`apt-get install apache2 apache2-doc apache2-utils libapache2-mod-php php7.0 php7.0-common php7.0-gd php7.0-mysql php7.0-imap php7.0-cli php7.0-cgi libapache2-mod-fcgid apache2-suexec-pristine php-pear php-auth php7.0-mcrypt mcrypt imagemagick libruby libapache2-mod-python php7.0-curl php7.0-intl php7.0-pspell php7.0-recode php7.0-sqlite3 php7.0-tidy php7.0-xmlrpc php7.0-xsl memcached php-memcache php-imagick php-gettext php7.0-zip php7.0-mbstring`

Then run the following command to enable the Apache modules suexec, rewrite, ssl, actions, and include (plus dav, dav_fs, and auth_digest if you want to use WebDAV):

`a2enmod suexec rewrite ssl actions include cgi`

`a2enmod dav_fs dav auth_digest headers`

To ensure that the server can not be attacked trough the HTTPOXY vulnerability, you must disable the HTTP_PROXY header in apache globally.

`nano /etc/apache2/conf-available/httpoxy.conf`

Paste this content to the file:

```
<IfModule mod_headers.c>
    RequestHeader unset Proxy early
</IfModule>
```

Enable the config file by running: `a2enconf httpoxy`

Restart Apache afterward: `service apache2 restart`

###PHP Opcode cache

APCu is a free PHP opcode cacher for caching and optimizing PHP intermediate code. It is strongly recommended to have one of these installed to speed up your PHP page.

APCu can be installed as follows:

`apt-get install php7.0-opcache php-apcu`

Now restart Apache: `service apache2 restart`

##Install BIND DNS Server

BIND can be installed as follows:

`apt-get install bind9 dnsutils haveged`

##Install Jailkit

Jailkit is needed only if you want to chroot SSH users. It can be installed as follows:

`apt-get install build-essential autoconf automake1.11 libtool flex bison debhelper binutils`

```
cd /tmp
wget http://olivier.sessink.nl/jailkit/jailkit-2.19.tar.gz
tar xvfz jailkit-2.19.tar.gz
cd jailkit-2.19
./debian/rules binary
```

You can now install the Jailkit .deb package as follows:

```
cd ..
dpkg -i jailkit_2.19-1_*.deb
rm -rf jailkit-2.19*
```

##Install fail2ban

This is optional but recommended:

`apt-get install fail2ban`

To make fail2ban monitor vsftpd, create the file /etc/fail2ban/jail.local:

`nano /etc/fail2ban/jail.local`

and paste into it:

```
[vsftpd]
enabled  = true
port     = ftp
filter   = vsftpd
logpath  = /var/log/syslog
maxretry = 3
```

##Install vsftpd

```
add-apt-repository ppa:thefrontiergroup/vsftpd
apt-get update
apt-get install vsftpd
```

###Create an FTP user

If you take a peek at /etc/vsftpd.chroot_list, you'll see the following:

```
# vsftpd userlist
# If userlist_deny=NO, only allow users in this file
# If userlist_deny=YES (default), never allow users in this file, and
# do not even prompt for a password.
# Note that the default vsftpd pam config also checks /etc/vsftpd/ftpusers
# for users that are denied.
root
bin
daemon
adm
lp
sync
shutdown
halt
mail
news
uucp
operator
games
nobody
```

This is basically saying, "Don't allow these users FTP access." vsftpd will allow FTP access to any user not on this list.

So, in order to create a new FTP account, you may need to create a new user on your server.

```
adduser advice
passwd advice 
```

###Restricting users to their home directories

At this point, your FTP users are not restricted to their home directories. That's not very secure, but we can fix it pretty easily.

Edit your vsftpd conf file again by typing:

`nano /etc/vsftpd.conf`

Un-comment out the line:

`chroot_local_user=YES`

###Changing a user's FTP home directory

restrict their FTP access to a specific folder, such as /var/www. In order to do this, you'll need to change the user's default home directory:

`usermod -d /var/www/ advice`

`sudo usermod -a -G www-data advice`

Edit and add the following lines to vsftpd.conf:

`nano /etc/vsftpd.conf`

```
utf8_filesystem=YES
connect_from_port_20=YES
pasv_enable=YES
pasv_min_port=1024
pasv_max_port=1048
pasv_address=35.156.115.9
chown_username=www-data
chmod_enable=YES
local_umask=0002
```

and do not forget to OPEN in ec2 instance the ports:

`80, 20, 21, 1024-1048`

At this point we cannot connect vis sftp to server so we need to make another changes:

`groupadd sftponly`

`nano /etc/ssh/sshd.config`

add the lines:

```
AllowUsers ubuntu advice

Match Group sftponly
    ChrootDirectory /var/www
    ForceCommand sftp
    AllowTcpForwarding no
```

save it and restart the service: `service sshd restart`

To restrict user access to just var/www/html folder we need to make a directory

`mount --bind /var/www /home/advice`

block shell access: `usermod -s /bin/false advice`


And ... that's all.