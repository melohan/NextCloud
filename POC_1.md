# SRW3 - POC 1

The purpose of this report is to document the configuration of a GNU/Linux server on which NextCloud has been installed.
This document will list the steps that will allow to :

1. Configure a secure connection from a client computer.
2. Install and configure a PHP and MariaDB server
3. Install and configure NextCloud (without native errors)

## Authors

- Ohan Mélodie
- Tesfazghi Robiel

## Table of contents

- [Introduction](#introduction)
- [Part I : Working from mutiple PC's](#part-i--working-from-mutiple-pcs)
    - [Port forwarding configuration](#port-forwarding-configuration)
    - [Install and configure SSH](#install-and-configure-ssh)
- [Part II: Apache, PHP and MySQL installation](#part-ii-apache-php-and-mysql-installation)
    - [Install PHP](#1-install-php)
    - [Install Apache2](#2-install-apache2)
    - [Install MariaDB server](#3-install-mariadb-server)
- [Part III: NextCloud without errors](#part-iii-nextcloud-without-errors)
    - [Download and unzip NextCloud](#download-and-unzip-nextcloud)
    - [Configure NextCloud](#configure-nextcloud)
    - [Start NextCloud](#start-nextcloud)
    - [Fix NextCloud errors](#fix-nextcloud-errors)
- [Sources](#sources)
   
## Introduction: Server configuration

Download this operating system: [Ubuntu server 20.04.3](https://ubuntu.com/download/server)


For this project we use the $ and # signs for shell code presentation.

- *$ : The command is run by simple user*
- *\# : The command is run by superuser*

> In this document, the host machine (the one on which the vm is installed) has the ip address `10.229.33.170` and the VM has the IP address `192.168.17.132`.

### 1) VmWare prerequisites

1) Install GNU/Linux Ubuntu server later manually, for now, you just have to specify `Linux Ubuntu 64 bits`.

2) Then define the material according to the table below:

|Hardware|| 
|----|----|
|Processor Configuration|2 processors, 1 core per processor|
|Memory|4096MB|
|Network Connection |NAT|
|SCSI Controller|LSI Logic|
|Virtual disk type|SCSI|
|Disk size|20 GB|

3) Once the virtual machine is created, right click on the VM and go to `Settings`
4) We recommand to remove the following hardware items: `USB Controller`, `Sound Card` and `Printer`.
5) Go to `CD/DVD (SATA)` and set your ISO file Ubuntu server.

### 2) Install Ubuntu Server

You can now start your vm.

|Settings||
|----|----|
|Language|English|
|Keyboard configuration|French, Switzerland|
|Network connections|default|
|Domain|srw3|
|Proxy|-|
|Mirror address|default|
|Storage configuration|Use an entire disk (`/dev/sda local disk 20.000G`)|
|Storage configuration|default|

You can skip additional install. We will do required installation in further steps.

### 3) First start up

Assure that the server is up to date with this command ```# apt update && sudo apt upgrade -y```

## Part I : Working from mutiple PC's

--- 

### Port forwarding configuration

> Here you will prepare your environnment for Part II and Part III

1) Ensure that the VM is connected to NAT. From VMware:

    - Right click on the virtual machine and go to `settings`
    - From `Hardware` tab, go to `Network Adapter` and select `NAT: Used to share the host's IP address`

2) From your Ubuntu server, get your VM IP address with the command `ip a`.

3) In VMware Workstation go to `Edit` > `Virtual Network Editor`.
    - Clic on VMnet 8 (NAT external connection)
    - Clic on `NAT Settings`
    - As Gateway IP set up your VM IP address
        - Set this configuration:

         |Configuration|Value|
         |----|----|
         |Host port|8080|
         |Type|TCP|
         |Virtual machine IP address|[YOUR_IP]|
         |Virtual machine port|80|

    - Apply changes and reboot your Ubuntu server with `$ reboot` command.

### Install and configure SSH

1) Install SSH package 
   ``` 
   # apt install openssh-server -y
   ```

2) Verify SSH status with
   ```
   $ systemctl status ssh
   ```

3) Set SSH port and add under `Include /etc/ssh/sshd_config.d/*.conf` a port 
   
   ```
   # nano /etc/ssh/sshd_config
   ```
   ```
   Port 332
   ```

4) Restart ssh service with
   ```
   # systemctl restart sshd
   ```   

5) Test SSH connection. Connect from another machine on the network with this
   command `ssh user@serveripaddress -p [PORT]`
   ```
   ssh cpnv@192.168.17.132 -p 332
   ```

## Part II: Apache, PHP and MySQL installation

--- 

### 1) Install PHP

   ``` 
   # apt install php7.4 php7.4-gd php7.4-mysql php7.4-curl php7.4-mbstring php7.4-intl php7.4-gmp php7.4-bcmath php7.4-xml php7.4-zip php-imagick php-apcu -y
   ```

### 2) Install apache2

   ```
   # apt install apache2 apache2-data apache2-utils -y
   ```

### 3) Install MariaDB server

   ```
   # apt install mariadb-server -y
   ```

1) You also need a little configuration after the installation. Type and enter this command:

   ```
   # mysql_secure_installation
   ```

2) Make sure to configure your MariaDB server as follow

   > **This is important don't set password directly, follow these sequence:**

    - Enter current password for root (enter for none): **enter**
    - Set root password [Y/n]: **Y**
    - New password : cpnv
    - Remove anonymous users? : y
    - Disallow root login remotely : y
    - Remove test database : y
    - Reload privilege tables now : y

3) Connect as root user
   ``` 
   # mysql -u root -p 
   ```

4) Create our MySQL user. Keep `cpnv` as user name:
   ```MYSQL 
   create user 'cpnv'@'localhost' identified by 'cpnv';
   CREATE DATABASE nextcloud;
   GRANT ALL PRIVILEGES on nextcloud.* to 'cpnv'@'localhost';
   FLUSH PRIVILEGES;
   exit
   ```

## Part III: NextCloud without errors

---

### Download and unzip Nextcloud

1) Install Lynx In order to download Nexcloud and to ensure that our nextcloud server will be later correctly configured
   and installed, we will need to download the lynx web browser.

   ```shell
   # apt install lynx -y
   ```

2) Install unzip

   ```shell
   # apt install unzip
   ```

3) Download from home [Nextcloud 23.0.0](https://download.nextcloud.com/server/releases/nextcloud-23.0.0.zip) with this command:

   ```
   # wget https://download.nextcloud.com/server/releases/nextcloud-23.0.0.zip
   ```

4) Unzip the downloaded file with:
   ```
   # unzip nextcloud-23.0.0.zip -d /var/www
   ```

5) Change recursively owner with
   ```
   # chown -R www-data:www-data /var/www/nextcloud/
   ```

6) Enable module headers rewrite into apache2 and restart
   ``` 
   # a2enmod headers env dir mime rewrite
   # systemctl restart apache2
   ```

8) You can check its status with `service apache2 status`

### Configure nextcloud

1) Save the demo config file
   ```
   # mv /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/000-default.conf.txt
   ```

2) Then you will update `000-default.conf` configuration. Change its content as follow:
   ```
   # nano /etc/apache2/sites-available/000-default.conf
   ```
   ```
    <VirtualHost *:80>
    
     ServerName cloud.yourdomain.com
        DocumentRoot /var/www/nextcloud
    
        <Directory /var/www/nextcloud/>
            Require all granted
            AllowOverride All
            Options FollowSymLinks MultiViews
    
            <IfModule mod_dav.c>
                Dav off
            </IfModule>
    
            RewriteEngine On
            RewriteRule ^/\.well-known/carddav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
            RewriteRule ^/\.well-known/caldav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
            RewriteRule ^/\.well-known/host-meta https://%{SERVER_NAME}/public.php?service=host-meta [QSA,L]
            RewriteRule ^/\.well-known/host-meta\.json https://%{SERVER_NAME}/public.php?service=host-meta-json [QSA,L]
            RewriteRule ^/\.well-known/webfinger https://%{SERVER_NAME}/public.php?service=webfinger [QSA,L]
    
        </Directory>
    
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
    
    </VirtualHost>
   ```

3) Restart apache 2 server with
   ```
   # service apache2 restart
   ```  

### Start Nextcloud

1) In `Part I` you have configured port forwarding. Now connect you have connect to your NextCloud from your client PC
   at `http://localhost:8080/`.

2) At first you have to define your admin account. Then as database configuration set what you defined in `Part II`

   |Nextcloud configuration||
   |----|----|
   |Administrator|cpnv|
   |Password|cpnv|
   |Data directory|`/var/www/nextcloud/data`|  
   |MySQL user|cpnv|
   |Password|cpnv|
   |Database|nextcloud|
   |Host|localhost|

> App installation can take a few minutes.

### Fix Nextcloud errors

Once Nextcloud is installed, go to your user profile > `settings` > `overview`.

> Once the Security & setup warnings are loaded, you may encounter some errors. We will correct them in this section.

 
#### PHP memory limit fix

> The PHP memory limit is below the recommended 512 MB.

1) We will edit php.ini file. Execute this command 
`# nano /etc/php/7.4/apache2/php.ini`
2) Press `ctrl` + `_` and search `memory_limit`
3) Set `memory_limit = 512M`
4) Save changes
5) Restart apache2 with `# service apache2 restart`

#### Set default region prefix

> Your installation does not have a default region prefix.

Update config file with this command

```
# nano /var/www/nextcloud/config/config.php
```

Update configuration as follow:

- Add this line just above `datadirectory`

```php
'default_phone_region' => 'CH',
```

Stay in this file the next step add new lines to fix cache problems.

#### Fix cache problems

> No cache configured. To improve performance, please configure a memcache, if available.

Right after `installed` add theses lines:

```php
'memcache.local' => '\\OC\\Memcache\\APCu',
'updater.secret' => '$2y$10$0LJFZGD6iJ78Q0CohdF18OCDgCh3GikVytJto6pFyj.SBrMNizAlW',
'maintenance' => false,
'theme' => '',
'loglevel' => 2, 
```

Apply change and save. Then restart server with this command:

```shell.
# service apache2 restart
```

#### Fix php-imageick error message

> The php-imagick module has no SVG support in this instance. For better compatibility, it is recommended to install it.

1) Remove imagemagick
   ```
   # apt remove imagemagick-6-common -y
   # apt remove php-imagick
   # apt autoremove -y
   ```

2) Re-install php-imageick
   ```
   # apt install php-imagick imagemagick -y
   ```

3) Restart apache2

   Execute this command `# service apache2 restart`

> At the end of this document there should be only one error left about accessing site insecurely via HTTP

--- 
## Sources

- [techguide.yt (To install and configure nextcloud)](https://techguides.yt/guides/how-to-install-and-configure-nextcloud-hub-21/)
- [docs.nextcloud.com](https://docs.nextcloud.com/server/19/admin_manual/contents.html)
