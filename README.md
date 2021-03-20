# centos-8-install

Change password :
```console
sudo passwd
```

- I log in as root :

Update :
```console
serverIp=151.80.58.150
domains=$(curl https://raw.githubusercontent.com/pierreminiggio/miniggio/main/index.php | grep "'*.*'," | sed -e 's/^[ \t]*//' | cut -c 2- | rev | cut -c 3- | rev | egrep -v 'é|œ')

# Update
yum update -y

# Install Apache
yum install httpd -y

# Install Firewall
yum install firewalld -y

# Start the Firewall
systemctl enable firewalld --now

# Allow http & https services
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload

# Create Vhosts
for domain in $domains;
do
domain=$domain;
folder=$(cut -d '.' -f1 <<<"$domain")
echo "
<VirtualHost *:80>
ServerName $domain
ServerAlias www.$domain
ServerAdmin pierre@miniggiodev.fr
DocumentRoot /var/www/html/$folder/
ErrorLog /var/www/logs/$folder_error.log
CustomLog /var/www/logs/$folder_access.log combined
<Directory "/var/www/html/$folder/">
    AllowOverride All
</Directory>
</VirtualHost>" >> /etc/httpd/conf.d/vhost.conf
! [ -d "/var/www/html/$folder" ] && mkdir /var/www/html/$folder && echo "test $domain" > /var/www/html/$folder/index.html;
done;

# Start Apache
mkdir /var/www/logs
chown -R apache /var/www/logs
setsebool -P httpd_unified 1
systemctl enable httpd --now

# Enable the EPEL & Remi repository
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y
yum install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm -y

# Install PHP
yum install -y yum-utils
dnf module enable php:remi-8.0 -y
yum install php php-cli php-common php-xml -y
systemctl restart httpd

# Check if PHP is installed
php --version

# Install wget
yum install wget -y

# Install MySQL
wget https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
rpm -Uvh mysql80-community-release-el7-3.noarch.rpm
yum install mysql-server -y
yum install php-mysqlnd -y
systemctl start mysqld.service
systemctl enable mysqld
mysql_secure_installation
```

```console
rm -f mysql80-community-release-el7-3.noarch.rpm

# Install unzip
yum install unzip -y

# Install Nano
yum install nano -y

# Install Certbot
yum install certbot python3-certbot-apache mod_ssl -y

# Install phpMyAdmin
cd /var/www/html
wget https://files.phpmyadmin.net/phpMyAdmin/5.1.0/phpMyAdmin-5.1.0-all-languages.zip
unzip phpMyAdmin-5.1.0-all-languages.zip
mv phpMyAdmin-5.1.0-all-languages phpmyadmin
chown -R apache:apache /var/www/html/phpmyadmin
rm -f phpMyAdmin-5.1.0-all-languages.zip

# Configure phpMyAdcd ../min
cd /var/www/html/phpmyadmin
cp config.sample.inc.php config.inc.php

# Set a blowfish secret
cat config.inc.php | sed -e "s/blowfish_secret'\] \= ''/blowfish_secret'\] \= '$(openssl rand -base64 32 | sed 's/\=/3/g')'/g" > config2.inc.php
mv config2.inc.php config.inc.php -f

# Add the following lines :
echo "\$cfg['TempDir'] = '/tmp/phpmyadmin';" >> config.inc.php
echo "\$cfg['Servers'][\$i]['hide_db'] = 'information_schema|mysql|performance_schema|phpmyadmin|sys';" >> config.inc.php

# Create vhost
echo 'Alias /phpmyadmin /var/www/html/phpmyadmin

<Directory /var/www/html/phpmyadmin/>
   AddDefaultCharset UTF-8

   <IfModule mod_authz_core.c>
     # Apache 2.4
     <RequireAny>
      Require all granted
     </RequireAny>
    </IfModule>
    <IfModule !mod_authz_core.c>
      # Apache 2.2
      Order Deny,Allow
      Deny from All
      Allow from 127.0.0.1
      Allow from ::1
    </IfModule>
</Directory>

<Directory /var/www/html/phpmyadmin/setup/>
   <IfModule mod_authz_core.c>
     # Apache 2.4
     <RequireAny>
       Require all granted
     </RequireAny>
   </IfModule>
   <IfModule !mod_authz_core.c>
     # Apache 2.2
     Order Deny,Allow
     Deny from All
     Allow from 127.0.0.1
     Allow from ::1
   </IfModule>
</Directory>
' > /etc/httpd/conf.d/phpmyadmin.conf

# Import phpMyAdmin tables
mysql < sql/create_tables.sql -u root -p
```

```console
# Restart Apache
systemctl restart httpd

# Install DNS
yum install bind bind-utils -y
systemctl start named

# Configure DNS
cp /etc/named.conf /etc/named.bak
cat /etc/named.conf | sed -e 's/listen-on port 53 { 127.0.0.1; };//g' > /etc/named2.conf
mv /etc/named2.conf /etc/named.conf -f
cat /etc/named.conf | sed -e 's/listen-on-v6 port 53 { ::1; };//g' > /etc/named2.conf
mv /etc/named2.conf /etc/named.conf -f
cat /etc/named.conf | sed -e 's/allow-query     { localhost; };//g' > /etc/named2.conf
mv /etc/named2.conf /etc/named.conf -f
cat /etc/named.conf | sed -e 's/recursion yes;/recursion no;/g' > /etc/named2.conf
mv /etc/named2.conf /etc/named.conf -f

# Restart DNS
systemctl restart named

# Allow DNS calls
firewall-cmd  --add-service=dns --zone=public  --permanent
firewall-cmd --permanent --add-port={53/udp,53/tcp}
firewall-cmd --reload

# Add domains into the DNS config
for domain in $domains;
do
domain=$domain;
echo "
zone \"$domain\" IN {
    type master;
    file \"$domain.db\";
    allow-update { none; };
};
" | sed -e "s/DOMAIN/$domain/g" >> /etc/named.conf; echo "\$TTL 86400
\$ORIGIN DOMAIN.
@ IN SOA ns1.DOMAIN. hostmaster.DOMAIN. (
    1           ;Serial
    3600        ;Refresh
    1800        ;Retry
    604800      ;Expire
    86400       ;Minimum TTL
)

;Name Server Information
 IN NS ns1.DOMAIN.

;IP address of Name Server
ns1 IN A $serverIp

;A - Record HostName To Ip Address
@ IN A $serverIp

;CNAME record
www IN CNAME DOMAIN." | sed -e "s/DOMAIN/$domain/g" > /var/named/$domain.db; chown named:named /var/named/$domain.db; named-checkconf; named-checkzone $domain /var/named/$domain.db;
done;

# Restart DNS
systemctl restart named

# Install Composer
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
mv composer.phar /usr/local/bin/composer
php -r "unlink('composer-setup.php');"

# Install Git
yum install git -y

# Install Python
yum install python3 -y

# Install NodeJS
yum install nodejs -y
npm cache clean -f
npm install -g n
n lts
npm install -g npm@latest
hash -r
lastVersion=$(ls /usr/local/n/versions/node | tac | grep -m1 "")
ln -sf /usr/local/n/versions/node/$lastVersion/bin/node /usr/bin/node

# FTP
yum install vsftpd -y
systemctl enable vsftpd --now

# Puppeteer prerequisites
yum install gcc-c++ -y make
yum install libXcomposite libXcursor libXdamage libXext libXi libXtst libmng libXScrnSaver libXrandr libXv alsa-lib cairo pango atk at-spi2-atk gtk3 -y

# Voice prerequisites
pip3 install gTTS
```
