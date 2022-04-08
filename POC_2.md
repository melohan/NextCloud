# SRW3 - POC 2

The purpose of this document is to secure an Ubuntu Linux server on which NextCloud has been installed. 

## Authors

- Ohan Mélodie
- Tesfazghi Robiel

## Table of contents
- [Introduction](#introduction)
- [Securing Apache2](#securing-apache2)
   - [fail2ban config](#fail2ban-config)
   - [SSL OK](#ssl-ok)
   - [Well-know present](#well-know-present)
   - [Apache config](#apache-config)
      - [Set server name](#set-server-name)
      - [HTTPS Anytime](#https-anytime-force-secure-connections)
      - [XSS Protection](#xss-protection)
      - [Clickjacking protection](#clickjacking-protection-x-frame-options)
      - [ETags off](#etags-off)
      - [HTTP Trace](#http-trace)
      - [Signature off & Tokens Prod](#signature-off--tokens-prod)
      - [Directory Index off](#directory-index-off)
- [Sources](#sources)
   
      
## Introduction 

For this project we use the $ and # signs for shell code presentation.

- $ : The command is run by simple user*
- \# : The command is run by superuser*

> In this document, the host machine (the one on which the vm is installed) has the ip address `10.229.33.170` and the VM has the IP address `192.168.17.132`.

## Securing Apache2

### fail2ban config

1. Update the server with  ```# apt update && sudo apt upgrade -y```

2. Open the config file and disable Nextcloud Brute force protection and update the timezone by modifying the following file

   ```
   # nano /var/www/nextcloud/config/config.php
   ```
   ```editorconfig
   'auth.bruteforce.protection.enabled' => 'false',
   'logtimezone' => 'Europe/Zurich',
   ```

3. Install and check fail2ban status

   ```
   # apt install fail2ban -y
   $ systemctl status fail2ban 
   ```

4. To prevent your changes from being overwritten in the future copy `jail.conf` to `jail.local`

   ```
   # cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
   ```

5. Now implement your security measures in `jail.local`

   ```
   # nano /etc/fail2ban/jail.local
   ```
   ```editorconfig
   [DEFAULT]
   # Whitelist localhost, your vm and your hos tip
   ignoreip = 127.0.0.1/8  192.168.17.132/24 10.229.33.170/20
   # Find 720 minutes in logs, ban 1 day after 3 attempts
   maxretry = 3
   bantime = 86400
   findtime = 43200
   # Ban IP in all port
   port = 0:65535
   # set logpath
   logpath = /var/log/auth.log
   # Send mail after ban to 
   destemail = melodie.ohan@cpnv.ch
   sendername = Fail2BanAlerts
   # Send mail with whois and logs
   action = %(action_mwl)s 
   ```

#### SSH
6. Define your measures for ssh in the same file `jail.local`, by adding the following lines

   ```editorconfig 
   [sshd]
   # enable ssh jail
   enabled = true
   # Find 720 minutes in logs, ban 1 day after 3 attempts
   maxretry = 3
   bantime = 86400
   findtime = 43200
   # Bann in all port
   port = 0:65535
   ```

#### Apache2

7. Define your security measures for Apache by add the following lines to the file

   ```editorconfig
   [apache]
   # enable apache jail
   enabled = true
   # Set filter name
   filter = apache-auth
   # Find 720 minutes in logs, ban 1 day after 3 attempts
   maxretry = 3
   bantime = 86400
   findtime = 43200
   # Bann in all port
   port = 0:65535
   ```

#### Return Error 401

1. To detect unsuccessful connection attempts and return 401 error, you need to add the following lines to the file

   ```editorconfig
   [Definition]
   failregex =  - - \[.*\] ".*" 401
   ignoreregex =
   ```

#### Nextcloud

1. Set up a filter and jail for Nextcloud. Create `.conf` in `/etc/fail2ban/filter.d` named `nextcloud.conf`

   ```
    # nano /etc/fail2ban/filter.d/nextcloud.conf
   ```
2. Then add this following content

   ```editorconfig
   [Definition]
   # Defines how to handle the failed authentifications attempts found by nextcloud filter
   _groupsre = (?:(?:,?\s*"\w+":(?:"[^"]+"|\w+))*)
   failregex = ^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Login failed:
   ^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Trusted domain error. datepattern = ,?\s*"
   time"\s*:\s*"%%Y-%%m-%%d[T ]%%H:%%M:%%S(%%z)?"
   ```

3. Create `.local` file in `/etc/fail2ban/jail.d/` named `nextcloud.local` and add the following contents:

   ```
   # nano /etc/fail2ban/jail.d/nextcloud.local
   ```
   ```editorconfig
   [nextcloud]
   # enable jail
   enabled = true
   backend = auto
   # Bann in all port
   port = 0:65535
   protocol = tcp
   # Add filter name
   filter = nextcloud
   # Find 720 minutes in logs, ban 1 day after 3 attempts
   maxretry = 3
   bantime = 86400
   findtime = 43200
   # Set log path
   logpath = /var/www/nextcloud/data/nextcloud.log
    ```
   
4. Restart fail2ban and check his status 

   ```
   # service fail2ban restart
   # fail2ban-client status nextcloud
   ```
   
5. If everything is configured correctly, ativate fail2ban at startup with this command:

   ```
   # systemctl enable fail2ban.service
   ```

> To unban an ip you'll need this command `# fail2ban-client set sshd unbanip YOUR_IP`


### SSL OK

1. Create a directory that serves as the root directory for the CA.

   ```
   # mkdir /root/ca/
   ```

2. Enable permission restriction on these files and ony allow the owner to modify with

   ```
   su
   chmod 600 /root/ca
   cd /root/ca/
   ```

3. Run this OpenSSL command to create a RSA private key srw.cpnv.local.key of length of `4096 bits` for CA’s certificate.

   ```
    # openssl genrsa -aes128 -out private.key 4096
    # openssl rsa -in private.key -out srw.cpnv.local.key
   ```

4. Run this command to create a certificate that expires in 700 days.

   ```
   # openssl req -new -days 700 -key srw.cpnv.local.key -out srw.cpnv.local.csr
   ```

   > **This is important set values as follow (except for Common Name):**
   
   ```editorconfig
   Country Name (2 letter code) [AU]:CH
   State or Province Name (full name) [Some-State]:VAUD
   Locality Name (eg, city) []:STE-CROIX
   Organization Name (eg, company) [Internet Widgits Pty Ltd]:CPNV
   Organizational Unit Name (eg, section) []:ES
   Common Name (e.g. server FQDN or YOUR name) []:srw.cpnv.local
   Email Address []:

   Please enter the following 'extra' attributes
   to be sent with your certificate request
   A challenge password []:cpnv
   An optional company name []:cpnv
   ```

5. Send the request `srw.cpnv.local.csr` to the teacher to get srw.cpnv.local.cert

6. In your host pc, transfer the file to the server by executing this command:

   ``` 
   scp -P 332 srw.cpnv.local.cer cpnv@192.168.126.128 
   ```

7. Place the generated private key, Certificate Signing Request and the
valid HTTPS certificate in the appropriate locations:

   ```
   $ cd /root/ca/
   $ mv /home/cpnv/srw.cpnv.local.cer .
   $ cp srw.cpnv.local.cer /etc/ssl/certs/
   $ cp srw.cpnv.local.key /etc/ssl/private/
   $ cp srw.cpnv.local.csr /etc/ssl/private/
   ```

#### Install certificate

8. Recover ` CPNV-ME-ROOTCA.crt` and double click on it.
9. Do you want to open this file? `yes`
10. Click on `Install certificate`
11. Install for each users.
 
### Well-know present

> Your should get this error message: "Your web server is not properly set up to resolve “/.well-known/caldav”. Further information can be found in the documentation 67."

1. Update Apache configuration file to fix this error.  Update `default-ssl.conf` file

   ```
   # nano /etc/apache2/sites-available/default-ssl.conf
   ```   
   ```apacheconf
   <VirtualHost *:443>
      DocumentRoot /var/www/nextcloud

      <Directory /var/www/nextcloud/>
                  Options +FollowSymlinks AllowOverride All
                  <IfModule mod_dav.c>
                              Dav off
                  </IfModule>
                  SetEnv HOME /var/www/nextcloud SetEnv HTTP_HOME /var/www/nextcloud

                  RewriteEngine On
                  RewriteRule ^/\.well-known/carddav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
                  RewriteRule ^/\.well-known/caldav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
                  RewriteRule ^/\.well-known/host-meta https://%{SERVER_NAME}/public.php?service=host-meta [QSA,L]
                  RewriteRule ^/\.well-known/host-meta\.json https://%{SERVER_NAME}/public.php?service=host-meta-json [QSA,L]
                  RewriteRule ^/\.well-known/webfinger https://%{SERVER_NAME}/public.php?service=webfinger [QSA,L]
      </Directory>

      <IfModule mod_headers.c>
                  Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
      </IfModule>

      SSLCertificateFile /etc/ssl/certs/srw.cpnv.local.cer SSLCertificateKeyFile /etc/ssl/private/srw.cpnv.local.key
   </VirtualHost>
   ```

2. Execute these commands to enable configuration, ssl and rewrite. Then restart apache2 service:

   ```
   # a2ensite default-ssl.conf
   # a2enmod ssl
   # a2enmod rewrite
   # service apache2 restart
   ```

### Apache config

#### Set server name
 1. Set server name. Open and edit `apache2.conf`  and add this line at the end of the file

   ```
   # nano /etc/apache2/apache2.conf
   ```
   ```
   ServerName localhost
   ```
#### HTTPS Anytime (Force secure connections)

1. To force secure connection we will update the config file `000-default.conf` with this command:

   ```
   # nano /etc/apache2/sites-available/000-default.conf
   ```

2. Then update as follow
   ```apacheconf
   <VirtualHost *:80>
           RewriteEngine On
           RewriteCond %{HTTPS} off
           RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI}
   </VirtualHost>
   ```

3. Restart apache2 service with

   ```
   # service apache2 restart
   ```

#### XSS-Protection

In this section we will secure Apache from Cross-site scripting
1. Update `security.conf` and set

   ```
   # nano /etc/apache2/conf-enabled/security.conf
   ```
   ```editorconfig
   Header always set X-XSS-Protection "1; mode=block"
   ```

#### Clickjacking protection (X-Frame-Options)

> With the setting `sameorigin`, you can emed pages on same origin.

1. Update `security.conf` with this command and set

   ```
   # nano /etc/apache2/conf-enabled/security.conf
   ```
   ```
   Header set X-Frame-Options: "sameorigin"
   ```

2. Restart the service with

   ```
   # service apache2 restart
   ```

##### ETags off

1. Update `htaccess` with this command

   ```
   # nano /var/www/nextcloud/.htaccess
   ```

2. Add these directives

   ```
   <IfModule mod_headers.c> 
       Header unset ETag 
   </IfModule> 
   FileETag None
   ```

3. Restart the service with

   ```
   # service apache2 restart
   ```

##### HTTP trace

1. Update `htaccess` with this command and add these directives

   ```
   # nano /var/www/nextcloud/.htaccess
   ```
   ```
   RewriteEngine On
   RewriteCond %{REQUEST_METHOD} ^(TRACE|TRACK)
   RewriteRule .* - [F]
   ```

3. Restart the service with
   ```
   # service apache2 restart
   ```

### Signature off & Tokens Prod 

1. Update `security.conf` file
   ```
   # nano /etc/apache2/conf-enabled/security.conf.
   ```

2. Find `ServerSignature` and setit to `Off` to not display server version on error pages or other generated pages.

4. Right below, set `ServerTokens` to `Prod` to tell Apache to only return `Apache` in the server header on every
   returned page request.

### Directory Index off

1. Update `htaccess` with this command

   ```
   # nano /var/www/nextcloud/.htaccess
   ```

2. Add  `DirectoryIndex disabled`

3. Then add below `Options -Indexes`. It will display an 403 Forbidden error if someone tries to browse the content of
   any directory.

4. Update /etc/apache2/apache2.conf file

   ```
   # nano /etc/apache2/apache2.conf
   ```

   ```
   Options Indexes FollowSymLinks
   Options FollowSymLinks
   ```

--- 
   ## Sources

- [www.techrepublic.com](https://www.techrepublic.com/article/how-to-improve-apache-server-security-by-limiting-the-information-it-reveals/)
- [thesecmaster.com](https://www.thesecmaster.com/how-to-set-up-a-certificate-authority-on-ubuntu-using-openssl/)
- [smashingmagazine.com](https://www.smashingmagazine.com/2017/06/guide-switching-http-https/)
- [techuides.yt (for ssl certificate)](https://techguides.yt/guides/free-wildcard-ssl-certificate-for-nextcloud-and-wordpress/)
- [keycdn](https://www.keycdn.com/blog/http-security-headers)
- [tecadmin.net](https://tecadmin.net/configure-x-frame-options-apache/)
- [fedingo.com](https://fedingo.com/how-to-disable-http-trace-method-in-apache/)
- [simplified.guide](https://www.simplified.guide/apache/disable-directory-listing)
