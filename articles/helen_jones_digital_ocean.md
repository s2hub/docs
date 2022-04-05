# Beginners’ guide to installing Silverstripe CMS on DigitalOcean
Helen Jones

## Introduction

### Caveat
There are many different ways to set up a server. This guide shows one way to set up a digital ocean droplet server that will work for a Silverstripe CMS project. You are advised to always ensure you are working with up to date versions of software to have the latest security updates.

The versions of software given in the guide were the latest versions available at the time it was written.

### How to use guide:
This guide will walk you through setting up a digital ocean server with CentOS 7 operating system, PHP v 7.4 (including extensions required by Silverstripe CMS), Apache v 2.4, and MariaDB v 10.6. If you already have a suitable server set up you can skip those parts and use only the part on installing and setting up Silverstripe CMS.

XXX Things that need to be typed or will be seen as responses/outputs in terminal are shown on a grey background. Red font indicates things which need to be typed, black font indicates things that will be shown as an output/response/prompt. Some text files will need to be created or edited directly on the droplet/server. Nano is used as the text editor program on the server. Things to be typed in nano are shown as blue font on a grey background. Every time a file is created of edited in nano, you need to press `Ctrl+o` to save the file and then `Ctrl+x` to exit the nano program. Notes to clarify things in the terminal or nano sections are shown in green font.

For the purposes of this guide, the fictional person Katniss Everdeen is used as the developer and she uses password-primrose, password-peeta, and password-cinna as passwords. You should use your own suitable usernames and secure passwords for all steps. Katniss’s website has the domain name mywebsite.url so you should replace this with your own domain name wherever this is used in the guide.

## Brief guide for those who already know how to set up the droplet/server

Create a new Silverstripe CMS project called website1 in the webserver folder:

```bash
> cd /var/www/
> composer create-project silverstripe/installer website1
```

The 2 warnings about depreciated packages can be ignored

XXX Pro tip: if you want to install a specific version of Silverstripe CMS rather than the latest one available, XXX

IMPORTANT: You will need to make sure there is an apache virtual host pointing to the public subfolder of the website1 folder that is created. See section 3.2 for details.

You now need to modify some file and folder owners and permissions, particularly the selinux settings, to allow Silverstripe CMS to run.

```bash
> cd website1
> sudo chown -R katniss:silverstripe public
> sudo semanage fcontext -a -t httpd_sys_rw_content_t “/var/www/website1/public(/.*)?”
> sudo restorecon -R -v /var/www/website1/public
> cd public
> sudo chown -R apache:silverstripe assets
```

You also need to create a .env file containing settings for your Silverstripe CMS project

/var/www/website1/.env
```yaml
SS_DATABASE_CLASS=”MySQLDatabase”
SS_DATABASE_NAME=”website1”
SS_DATABASE_SERVER=”localhost”
SS_DATABASE_USERNAME=”katnisswebsite”
SS_DATABASE_PASSWORD=”password-peeta”
SS_DEFAULT_ADMIN_USERNAME=”katniss-default”
SS_DEFAULT_ADMIN_PASSWORD=”password-cinna”
SS_ENVIRONMENT_TYPE=”dev”
```
The rest of the setup of the Silverstripe CMS project is done online. In chrome go to:

[https://mywebsite.url/dev/build](https://mywebsite.url/dev/build)

You should see a long webpage which shows the database being set up and lots of tables being created.

Next go to:

[http://mywebsite.url/?flush=all](http://mywebsite.url/?flush=all)

You should now see the built in homepage with a welcome message.

Click the link to log in to the CMS. You can now create a user account for yourself and give it full administrative rights. Then you can edit the .env file on the server to remove the 2 lines related to the default admin username and password.

## Full guide including setup of the droplet/server

### 1.  Buy and set up a droplet
Buy droplet of type: Basic plan; regular intel; $10/mo, 2GB ram / 1 CPU, 50GB SSD disk; 2TB transfer.

Select initial setup as: 
- authentication via SSH key
- CentOS 7 x64 operating system
- add monitoring

Create an SSH key for the authentication. You need to create an SSH key using the specific computer you will be using to connect to the server.

Caution: If you already use SSH key to connect to something else you need to take care not to overwrite the existing key.

In terminal enter:
XXX


#### 1.1.  Log in to droplet and do initial setup
Login to the droplet via the console on the website then find out the ECDSA key fingerprint:

```bash
> cd /etc/ssh
> ssh-keygen -lf ssh_host_ecdsa_key.pub
```

Take a screenshot of the output, or copy/paste it to a temporary text file on your computer

Login to the droplet from your computer using terminal

```bash
> ssh root@{ipaddress}
```

It will show you the ECDSA fingerprint. Make sure that this matches the one you got from the digital ocean console. You will only need to do this the first time you log in.

Setup a new user to work with instead of root

```bash
> adduser katniss
> passwd katniss
# password-primrose - this won’t display as you type it
> gpasswd -a katniss wheel
```

Update all the included software

```bash
> yum update
```

install the nano text editor program

```bash
> yum install nano
```

Setup the public key so the new user can also use it.

```bash
> su katniss
> cd ~/
> mkdir .ssh
> cd .ssh
> nano authorized_keys
```
Now paste the public key from your computer (get it from terminal by entering cat ~/.ssh/id_rsa.pub) and press `Ctrl+o` (to save the file) then `Ctrl+x` (to exit nano)

```bash
> chmod -R go= ~/.ssh
```

VERY IMPORTANT STEP!! Use a new terminal window to make sure the new user can log in.

All future work will be done as the new katniss user. As long as all went well, disable the root user from being able to login.

```bash
> sudo nano /etc/ssh/sshd_config
# press Ctrl+w to enter search mode and search for PermitRootLogin
# change from yes to no
# press Ctrl+o and then Ctrl+x
> sudo systemctl reload sshd
```

Create a new group of users and add users to it
```bash
> sudo groupadd silverstripe
> sudo usermod -a -G silverstripe katniss
> sudo usermod -a -G silverstripe apache
```

#### 1.2.  Install helper software and additional repositories
Wget (for installing things from the internet)

```bash
> sudo yum install wget
```

Repositories (places where software can be installed from)

```bash
# EPEL
> sudo yum install epel-release

# CodeIT
> cd /etc/yum.repos.d
> sudo wget https://repo.codeit.guru/codeit.el`rpm -q --qf "%{VERSION}" $(rpm -q --whatprovides redhat-release)`.repo

# Remi
> sudo yum install [https://rpms.remirepo.net/enterprise/remi-release-7.rpm](https://rpms.remirepo.net/enterprise/remi-release-7.rpm)
> sudo yum-config-manager --enable remi-php74
> sudo yum update

# IUS
> sudo yum install [https://repo.ius.io/ius-release-el7.rpm](https://repo.ius.io/ius-release-el7.rpm)
> sudo yum-config-manager --enable ius
> sudo yum update

# MariaDB repo
> cd /etc/yum.repos.d
> sudo nano MariaDB.repo
```

/etc/yum.repos.d/MariaDB.repo
```yaml
[mariadb]
name=MariaDB
baseurl=http://yum.mariadb.org/10.6/centos7-amd64
gpgkey=https://yum.mariadb.org.RPM-GPG-KEY-MariaDB
gpgcheck=1
```

### 2. Install software required to run Silverstripe CMS

Most of the software will be installed using the in-built program yum from the various repositories installed in step 1.2. It’s a good idea to check the version that will be installed and where from first to ensure all the repo setups worked correctly.

Use the following 2 commands to check the version and then install each program by replacing {package name} with the name as given in the table below

```bash
> yum info {package name}
> sudo yum install {package name}
```

All items in this table need to be installed

| Software           | Package name         | Repository | Version |
|--------------------|----------------------|------------|---------|
| Apache             | httpd                | CodeIT     | 2.4.48  |
| MariaDB            | MariaDB-client       | MariaDB    | 10.6.4  |
| MariaDB            | MariaDB-server       | MariaDB    | 10.6.4  |
| PHP                | php                  | remi-php74 | 7.4.23  |
| PHP-MySqlconnector | php-mysqlnd          | remi-php74 | 7.4.23  |
| PHP-intl           | php-intl             | remi-php74 | 7.4.23  |
| PHP-mbstring       | php-mbstring         | remi-php74 | 7.4.23  |
| PHP-xml            | php-xml              | remi-php74 | 7.4.23  |
| PHP-ZendOpcache    | php-opcache          | remi-php74 | 7.4.23  |
| Imagick            | ImageMagick7-libs    | remi-safe  | 7.1.0.5 |
| Imagick            | php-pecl-imagick-im7 | remi-php74 | 3.5.1   |
| Git                | git224               | IUS        | 2.24.4  |

Composer 1 and Composer 2 need to be installed in a way that allows them both to be used without needing to use the pseudo-root function.

For composer 2:

```bash
> cd /usr/bin

# Go to [https://getcomposer.org/download](https://getcomposer.org/download) and 
# copy the code in the first code box on the page (it starts with php -r "copy( and 
# ends with setup.php');"). Paste and enter the code and then enter

> sudo cp composer.phar /usr/local/bin/composer
```

For composer 1:

```bash
> cd /usr/local/bin

# Paste the code from the website as for composer 2 BUT edit line 3 by adding the # version number for composer 1 so that it says:

> sudo php composer-setup.php --version=1.10.22
```

Enter the edited code and then enter

```bash
> sudo cp composer.phar composer1
```

### 3.  Setup software
  
  PHP, Apache, and MariaDB all need to be setup ready to install Silverstripe CMS. Set up of git is also included here but if your working method will upload your custom code to the droplet/server in a different way then you can omit this step.

#### 3.1.  MariaDB
    
First MariaDB needs to be turned on, set so that it automatically turns on if the server needs to be restarted, and then checked to ensure it is running ok.

```bash
> sudo systemctl start mariadb
> sudo systemctl enable mariadb
> sudo systemctl status mariadb
```

Check the output to make sure MariaDB is running. Next some settings need to be modified to make MariaDB more secure

```bash
> sudo mariadb-secure-installation
# Respond to the questions/prompts given as follows:

switch to unix-socket authentication n
change root password password-primrose
remove anonymous users y
disallow root login remotely y
remove rest database and access to it y
reload privilege tables now y

> mysqladmin -u root -p version
> sudo nano /etc/my.cnf.d/server.cnf
```

Go to the section `[mysqlnd]` and add:

/etc/my.cnf.d/server.cnf
```yaml
[mysqlnd]
Bind-address=127.0.0.1
Local-infile=0
Log=/var/log/mariadb-logfile
```

```bash
> mariadb -u root -p
# password-primrose this won’t display as you type it

mariadb> select user, host, password from mysql.user;
mariadb> rename user ‘root’@’localhost’ to ‘katnissroot’@’localhost’;
mariadb> sh privileges;
mariadb> exit

> mariadb -u katnissroot -p
# password-primrose this won’t display as you type it

mariadb> show grants for ‘katnissroot’@’localhost’;
mariadb> create user ‘katnisswebsite’@localhost identified by ‘password-peeta’;
mariadb> grant all privileges on *.* to ‘katnisswebsite’@’localhost’;
mariadb> flush privileges;
mariadb> exit

> sudo systemctl restart mariadb
```
  
#### 3.2.  Apache

Apache also needs to be turned on and set so that it automatically turns on if the server needs to be restarted.

```bash
> sudo systemctl start httpd.service
> sudo systemctl enable httpd.service
```

The easiest way to check if Apache is running correctly is to make a very simple webpage and check that it can be viewed online.

```bash
> cd /var/www/html
> sudo nano index.html
```

/var/www/html/index.html
```html
<!DOCTYPE html>
 <html>
	 <body>
	 	<h1>May the odds be ever in your favour!</h1>
	 </body>
 </html>
```

Go to the server’s IP address in chrome to make sure you can see the page you just made.

Next you need to set up an Apache virtual host that will allow the internet to connect to the Silverstripe CMS project you are going to create. You can make multiple virtual hosts (one per project) if you are going to use the droplet to create more than one Silverstripe CMS website.

```bash
> sudo mkdir /var/www/website1
> sudo chown -R katniss:katniss /var/www/website1
> sudo mkdir /etc/httpd/sites-available
> sudo mkdir /etc/httpd/sites-enabled
> sudo nano /etc/httpd/conf/httpd.conf
```

go to the end of the file and add

/etc/httpd/conf/httpd.conf
```conf
# tell apache to look for virtual hosts in the sites-enabled directory

IncludeOptional sites-enabled/*.conf
```

```bash
> sudo nano /etc/httpd/sites-available/website1.conf
```

/etc/httpd/sites-available/website1.conf
```conf
<VirtualHost *:80>
 ServerName mywebsite.url
 ServerAlias [www.mywebsite.url](http://www.mywebsite.url)
 DocumentRoot “/var/www/website1/public”

<Directory “/var/www/website1/public”>
 Options Indexes FollowSymLinks MultiViews
 AllowOveride All
 Require all granted
 </Directory>

ErrorLog /var/log/httpd_website1/error.log
CustomLog /var/log/httpd_website1/requests.log combined

</VirtualHost>
```

It’s good to also set up a default virtual host so that if anyone points their domain name to your server it will not provide them any information.

```bash
sudo nano /etc/httpd/sites-available/aaa-default.conf
```

/etc/httpd/sites-available/aaa-default.conf
```conf
<VirtualHost _default_:80>
 Redirect 404 /
<VirtualHost>
```

The folder for the log files needs to be created, the 2 virtual hosts created need to be activated, and then apache needs to be restarted so the changes take effect.

```bash
> sudo mkdir /var/log/httpd_website1
> sudo ln -s /etc/httpd/sites-available/website1.conf /etc/httpd/sites-enabled/website1.conf
> sudo ln -s //etc/httpd/sites-available/aaa-default.conf /etc/httpd/sites-enabled/aaa-default.conf
> sudo systemctl restart httpd.service
```
  

#### 3.3.  PHP
    
The memory thingy that Tim and nightjarnz helped me sort out on slack (and I already forgot what it was – oops!)
XXX

#### 3. 4.  Git

### 4.  Install Silverstripe CMS

Create a new Silverstripe CMS project called website1 in the webserver folder:

```bash
> cd /var/www/
> composer create-project silverstripe/installer website1
```

The 2 warnings about depreciated packages can be ignored
Pro tip: if you want to install a specific version of Silverstripe CMS rather than the latest one available, XXX

IMPORTANT: You will need to make sure there is an apache virtual host pointing to the public subfolder of the website1 folder that is created. See section 3.2 for details.

You now need to modify some file and folder owners and permissions, particularly the selinux settings, to allow Silverstripe CMS  to run.

```bash
> cd website1
> sudo chown -R katniss:silverstripe public
> sudo semanage fcontext -a -t httpd_sys_rw_content_t “/var/www/website1/public(/.*)?”
> sudo restorecon -R -v /var/www/website1/public
> cd public
> sudo chown -R apache:silverstripe assets
```

You also need to create a .env file containing settings for your Silverstripe CMS project


```bash
cd /var/www/website1
nano .env
```

```yaml
SS_DATABASE_CLASS=”MySQLDatabase”
SS_DATABASE_NAME=”website1”
SS_DATABASE_SERVER=”localhost”
SS_DATABASE_USERNAME=”katnisswebsite”
SS_DATABASE_PASSWORD=”password-peeta”
SS_DEFAULT_ADMIN_USERNAME=”katniss-default”
SS_DEFAULT_ADMIN_PASSWORD=”password-cinna”
SS_ENVIRONMENT_TYPE=”dev”
```

The rest of the setup of the Silverstripe CMS project is done online. In chrome go to:

[https://mywebsite.url/dev/build](https://mywebsite.url/dev/build)

You should see a long webpage which shows the database being set up and lots of tables being created.

Next go to:

[http://mywebsite.url/?flush=all](http://mywebsite.url/?flush=all)

You should now see the built in homepage with a welcome message.

Click the link to log in to the CMS. You can now create a user account for yourself and give it full administrative rights. Then you can edit the .env file on the server to remove the 2 lines related to the default admin username and password.

## Conclusion

Congratulations on setting up your first Silverstripe CMS project!

Where to next?

You may want to read some of the beginners guides on this site, and do some of the lessons on the Silverstripe CMS website.

We particularly recommend:

XXX …..some specific guides

XXX …..the specific basic lessons in the series on silver stripe website

Once you have played around with your silver stripe project and are ready to create a real website you will definitely want to look into setting up your project to work on https once you have set up your SSL certificate on your server. XXX.... other things that should be done for real websites...
