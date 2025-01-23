# LAN PROJECT Part 2: HTTPS server
## Main servers configuration
We're adding a Ubuntu Desktop machine called host3, we're going to set the static IP for our DHCP server  
![](https://hackmd.io/_uploads/HJP09aHDn.png)

Adding the DNS records  
![](https://hackmd.io/_uploads/HJDvsTHwn.png)

host3 - 00:0C:29:08:1E:4A

All the web configuration is going to be done in our **ftp1** server

## Installation
First let's update our repositories
```shell
sudo apt update
```

Let's install our LAMP
```shell
sudo apt install apache2 mysql-server php libapache2-mod-php php-mysql
```
## PHP Configuration
You don't really have to configure anything, you can check the version you have installed with this command:
```shell
php -v
```
![](https://hackmd.io/_uploads/Hyg4YDEPh.png)

## MYSQL Configuration
Start the configuration with this command
```shell
sudo mysql_secure_installation
```

If you happen to encounter this error:
```shell
Failed! Error: SET PASSWORD has no significance for user 'root'@'localhost' as the authentication method used doesn't store authentication data in the MySQL server. Please consider using ALTER USER instead if you want to change authentication parameters
```

You can enter mysql as root with the following command
```shell
sudo mysql
```

And alter the root password with this command where 'mynewpassword' is your new password.
```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password by 'mynewpassword';
```
![](https://hackmd.io/_uploads/SJcqdDEPh.png)

You can exit MYSQL by typing **exit**

## Apache Configuration
Allow Apache ports 80 for regular HTTP, 443 for HTTPS (TLS/SSL)

#### Ports for Apache
We'll have to allow traffic through our firewall for our Apache to work
We can allow **Apache** just for port **80**
```shell
sudo ufw allow in "Apache"
```

**Apache Full** for both port **80** and **443**
```shell
sudo ufw allow in "Apache Full"
```

**Apache Secure** just for port **443**
```shell
sudo ufw allow in "Apache Secure"
```

We can check our firewall configuration using ufw status
```shell
sudo ufw status
```

#### Access your webpage
You can now type your local IP or localhost if you installed it in your local machine, in my case:
http://192.168.222.155
![](https://hackmd.io/_uploads/r10jvvVwn.png)

If you see this default page, then it means it was successful!

#### Allow a virtual host
We'll start by making a working directory, in my case the directory will be **dataproject** and the user will be **ftp1**
```shell
sudo mkdir /var/www/dataproject
```

We'll make **ftp1** it's owner
```shell
sudo chown -R ftp1:ftp1 /var/www/dataproject
```

We can create an html for testing
```shell
nano /var/www/dataproject/index.html
```

And type something like this
```shell
<h1>Testing Page</h1>
<p>If you're seeing this, it's working properly!</p>
```
![](https://hackmd.io/_uploads/SkEZAvVv3.png)

We'll start a new conf file in sites-available
```shell
sudo nano /etc/apache2/sites-available/dataproject.conf
```

We'll add the following:
```shell
<VirtualHost *:80>
    ServerName 192.168.222.155
#    ServerAlias www.your_domain        # I'm not using an alias for now
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/dataproject
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Now we can enable our site
```shell
sudo a2ensite dataproject
```

We should disable the default site
```shell
sudo a2dissite 000-default
```

We can test our syntax by doing this command
```shell
sudo apache2ctl configtest
```

Now we can reload our configuration 
```shell
sudo systemctl reload apache2
```

Don't forget to check if the service is working properly:
```shell
sudo systemctl status apache2
```
![](https://hackmd.io/_uploads/BJzFCDNPh.png)

Now we can access our website
![](https://hackmd.io/_uploads/HkEhAwVPn.png)

## HTTPS
### Self-Signed
We're going to enable the ssl mod
```shell
sudo a2enmod ssl
sudo systemctl restart apache2
```
We'll issue our certificate and key with this command:
```shell
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```
![](https://hackmd.io/_uploads/BJouwaHvn.png)

Now we'll have to modify our virtual host configuration file
```shell
sudo nano /etc/apache2/sites-available/dataproject.conf
```

We're changing the port from 80 to 443 and adding the SSL lines, we're also going to add another virtualhost configuration for the redirection from port 80 to https
```shell
<VirtualHost *:443>
    ServerName 192.168.174.30
#    ServerAlias www.your_domain        # I'm not using an alias for now
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/dataproject
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
</VirtualHost>

<VirtualHost *:80>
    ServerName 192.168.174.30
    Redirect / https://192.168.174.30/
</VirtualHost>
```
:warning: I changed the IP to our LAN to work in local :warning: 

Let's check syntax, restart and see the status of Apache
```shell
sudo apache2ctl configtest
sudo systemctl reload apache2
sudo systemctl status apache2
```

### Let's Encrypt
We're going to secure our Apache service with Let's Encrypt certificate
First we're going to update our repositories
```shell
sudo apt update
```

Then we're going to install let's encrypt
```shell
sudo apt install certbot python3-certbot-apache
```

Now we can obtain our certificate with this command:
```shell
sudo certbot --apache
```

## Testing
Now we can access our https server from inside the LAN with the IP or the FQDM:
https://192.168.174.30
![](https://hackmd.io/_uploads/HJxd66rP2.png)

We can see we're using https instead of http, however the browser will still see it as not to be trusted, simply because the certificate is self-signed.