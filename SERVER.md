## Linux Debian 9 (minimal server)
Debian 9 Server:
* Minimal Distribution
* SSH Server

### Server Package
```
tar zxvf server.tgz
mv server /server
```

### Dependencies

#### Build Deps
```
apt-get update
apt-get install -y build-essential
apt-get install -y autoconf make automake cmake libtool subversion git mercurial checkinstall bison flex unzip
```
#### MySQL Deps
```
apt-get install -y libncurses-dev libssl-dev
```
#### Apache Deps
```
apt-get install -y libapr1 libapr1-dev libaprutil1 libaprutil1-dev libpcre3-dev
```
#### PHP Deps
```
apt-get install -y libpng-dev libjpeg-dev libbz2-dev libcurl4-gnutls-dev libxml2 libxml2-dev libzip-dev libmcrypt-dev libfreetype6-dev libxpm-dev libwebp-dev
```

### Localization
```
dpkg-reconfigure locales
```
```
locale-gen en_US.UTF-8
```

### VIM Editor (FIX)
```
echo "set nocompatible" > ~/.vimrc
echo "set backspace=indent,eol,start" >> ~/.vimrc
```

### CURL Libraries (FIX)
```
cd /usr/include
ln -s x86_64-linux-gnu/curl
```

### Environment
`vi /etc/environment`
```
LANGUAGE=en_US.UTF-8
LANG=en_US.UTF-8
LC_ALL=en_US.UTF-8
JAVA_HOME=/server/java/jdk8
MYSQL_HOME=/server/mysql
PHP_HOME=/server/php
```

`vi /etc/bash.bashrc`
```
# permanently update global paths for all users
export PATH=$PATH:$JAVA_HOME/bin:$PHP_HOME/bin
```

### Security

#### Linux (Version / Release)
```
hostnamectl
```

#### SSH Certificate
```
ssh-keygen -t rsa -b 4096 -C "jalopezsuarez@gmail.com"
secure_rsa_jalopezsuarez_private.key
secure_rsa_jalopezsuarez_server.pub
```

#### SSH Username / Password (only certificate)
Generate random password (https://passwordsgenerator.net):
```
passwd
```
SSH Server disable password:
```
cat /etc/ssh/sshd_config | grep PasswordAuthentication
```
`vi /etc/ssh/sshd_config`
```
PasswordAuthentication no
```

#### SSH Certificates
Private Key (client side):
```
secure_rsa_jalopezsuarez_private.key
```
Public Key (server side):
```
cat secure_rsa_jalopezsuarez_server.pub >> ~/.ssh/authorized_keys
```

### MySQL

#### MySQL Server
```
rm -rf /server/mysql/data
mkdir /server/mysql/data
chmod 750 /server/mysql/data
```

```
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

```
cd /server/mysql
bin/mysqld --initialize --user=mysql --datadir=/server/mysql/data
bin/mysql_ssl_rsa_setup --datadir=/server/mysql/data
bin/mysqld_safe --user=mysql &
```

```
bin/mysql -u root -p
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
bin/mysqladmin -u root -p shutdown
```

#### MySQL Service
```
cp /server/mysql/support-files/mysql.server /etc/init.d/mysql
chmod +x /etc/init.d/mysql
```
```
systemctl start mysql.service
systemctl status mysql.service
```

#### MySQL Status
```
cd /server/mysql
bin/mysqladmin variables -p
bin/mysqladmin --help
bin/mysqld --help --verbose | grep my.cnf
```

#### MySQL References
```
Create user to access externally:
CREATE USER 'server'@'%.%.%.%' IDENTIFIED BY 'MYPASSWORD'; 
GRANT ALL PRIVILEGES ON *.* TO 'server'@'%.%.%.%' IDENTIFIED BY 'MYPASSWORD';
FLUSH PRIVILEGES;

Change user root password (1):
ALTER USER 'root'@'localhost' IDENTIFIED BY 'MYPASSWORD';
FLUSH PRIVILEGES;

Change user root password (2):
SELECT CURRENT_USER();
UPDATE user SET Password=PASSWORD('MYPASSWORD') WHERE User='root';
FLUSH PRIVILEGES;

Change user root password (3):
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('MYPASSWORD');
FLUSH PRIVILEGES;

Permissions to root to access from outside:
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' IDENTIFIED BY 'MYPASSWORD';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'127.0.0.1' IDENTIFIED BY 'MYPASSWORD';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'192.168.0.%' IDENTIFIED BY 'MYPASSWORD';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%.%.%.%' IDENTIFIED BY 'MYPASSWORD';

CREATE USER 'MYUSER'@'192.168.%.%' IDENTIFIED BY 'MYPASSWORD'; 
GRANT ALL PRIVILEGES ON *.* TO 'MYUSER'@'192.168.%.%' IDENTIFIED BY 'MYPASSWORD';
GRANT SELECT ON *.* TO 'MYUSER'@'192.168.%.%' IDENTIFIED BY 'MYPASSWORD';
```

### Apache

#### Apache Service
```
cp /server/apache/bin/apachectl /etc/init.d/apache
chmod +x /etc/init.d/apache
```

`vi /etc/init.d/apache`
```
# Comments to support LSB init script conventions
### BEGIN INIT INFO
# Provides: apache
# Required-Start: $local_fs $network $remote_fs $syslog
# Required-Stop: $local_fs $network $remote_fs $syslog
# Default-Start: 2 3 4 5
# Default-Stop: 0 1 6
# Short-Description: start and stop Apache
# Description: Apache is a open source HTTP web server.
### END INIT INFO
```

```
systemctl start apache.service
systemctl status apache.service
```

#### Apache SSL (https/443)
```
mkdir /server/apache/conf/ssl
apidox_net_private.key
apidox_net_certificate.cer
apidox_net_intermediate.cer
```

`vi /server/apache/conf/httpd.conf`
```
# ========================================================
# MultiHost
# ========================================================
<VirtualHost *:80>
    DocumentRoot /server/apache/htdocs
    ServerAdmin webmaster@localhost
    DirectoryIndex index.php index.phtml index.phps index.html index.htm
</VirtualHost>
<VirtualHost *:80>
    DocumentRoot /server/apache/htdocs/apidox_net
    ServerName apidox.net
    ServerAlias www.apidox.net
    ServerAdmin webmaster@localhost
    DirectoryIndex index.php index.phtml index.phps index.html index.htm
    RewriteEngine On
    RewriteCond %{HTTPS} off [OR]
    RewriteCond %{HTTP:X-Forwarded-Proto} !https
    RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]
</VirtualHost>
<VirtualHost *:443>
    DocumentRoot /server/apache/htdocs/apidox_net
    ServerName apidox.net
    ServerAlias www.apidox.net
    ServerAdmin webmaster@localhost
    DirectoryIndex index.php index.phtml index.phps index.html index.htm
    SSLEngine on
    SSLCertificateFile /server/apache/conf/ssl/apidox_net_certificate.cer
    SSLCertificateKeyFile /server/apache/conf/ssl/apidox_net_private.key
    SSLCertificateChainFile /server/apache/conf/ssl/apidox_net_intermediate.cer
</VirtualHost>
# ========================================================
# /MultiHost
# ========================================================
```

### Gearman

#### Gearman Service
`vi /etc/systemd/system/gearman.service`
```
[Unit]
Description=Gearman Service

[Service]
ExecStart=/server/java/jdk8/bin/java -jar /server/gearman/gearman-server-0.8.11-20150731.182506.jar
WorkingDirectory=/server/gearman/
Restart=always

[Install]
WantedBy=multi-user.target
```

```
systemctl enable gearman.service
systemctl daemon-reload
systemctl start gearman.service
```

### Services
```
update-rc.d mysql defaults 70 30
update-rc.d apache defaults 80 20
```
```
ls -l /etc/rc*.d/ | grep mysql*
update-rc.d mysql remove
ls -l /etc/rc*.d/ | grep apache*
update-rc.d apache remove
```

### Firewall (IPTABLES)

#### IPTables Firewall Server Security

Common rules for firewall server security:
```
iptables -A INPUT -m state --state INVALID -j DROP
iptables -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT

iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

iptables -P INPUT DROP
iptables -P FORWARD DROP
```

Additional rules to allow traffic IP/Ports:
```
iptables -A INPUT -s 192.168.0.0/24 -j ACCEPT
iptables -D INPUT -s 192.168.0.0/24 -j ACCEPT

iptables -A INPUT -p tcp --dport 3306 -s 188.78.107.149 -j ACCEPT
iptables -D INPUT -p tcp --dport 3306 -s 188.78.107.149 -j ACCEPT
```

#### IPTables Persistent (v4/v6)
```
apt-get install -y iptables-persistent
```

Clear all IPTables rules to defaults:

`vi /etc/iptables/rules.v4`
```
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
COMMIT
```

`vi /etc/iptables/rules.v6`
```
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
COMMIT
```

Save IPTables configuration (persistent):
```
iptables-save > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6
```

Common rules for firewall server security:

`vi /etc/iptables/rules.v4`
```
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state INVALID -j DROP
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
COMMIT
```

`vi /etc/iptables/rules.v6`
```
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
COMMIT
```
