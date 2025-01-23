# LAN PROJECT Part 1: DNS, DHCP and FTP Servers on Ubuntu 22.04 LTS
## VM Structures
We're using VMware Workstation 16.2.x with the following machines:
![](https://hackmd.io/_uploads/ry2rwQXv3.png)

#### Running on Ubuntu 22.04 LTS server
Download:
https://releases.ubuntu.com/jammy/ubuntu-22.04.2-live-server-amd64.iso

#### Netplan configuration for ns1 and ftp
```shell
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens32:    ~#Internal Network
      dhcp4: true
    ens34:     #WAN Network
      dhcp4: true
  version: 2
```
#### Netplan configuration for dhcp1
```shell
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens32:
      dhcp4: false
      addresses:
         - 192.168.174.10/24
      nameservers:
        addresses: [192.168.174.20]
        search: [ data.eibanez.cf ]
    ens34:
      dhcp4: true
  version: 2
```
#### Netplan configuration for hosts
```shell
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens32:    ~#Internal Network
      dhcp4: true
  version: 2
```

#### MAC Address
You can check your MAC Address from the machine's Network Adapter -> Advanced Configuration
![](https://hackmd.io/_uploads/rJyoamXPh.png)

Alternatively log in into your machine and run the following command
```shell
ip a
```
![](https://hackmd.io/_uploads/ByE0pXmD2.png)

#### Network planification
dhcp1  ens32: 192.168.174.10 - 00:0C:29:57:59:10
ns1    ens32: 192.168.174.20 - 00:50:56:3C:2F:B8
ftp1   ens32: 192.168.174.30 - 00:0c:29:fb:8e:67
host1  ens32: 192.168.174.10 - 00:50:56:39:A8:2D
host2  ens32: 192.168.174.10 - 00:50:56:3A:C0:CF

## DHCP Server
First let's change our hostname
```shell
sudo hostnamectl set-hostname dhcp1
```

Then we'll update our repositories
```shell
sudo apt update
```

We install the isc-dhcp-server
```shell
sudo apt install isc-dhcp-server
```

Now we can change the configuration file
```shell
sudo nano /etc/dhcp/dhcpd.conf
```

This is what our dhcpd.conf file would look like
```shell
# dhcpd.conf
#
# Sample configuration file for ISC dhcpd
#
# Attention: If /etc/ltsp/dhcpd.conf exists, that will be used as
# configuration file instead of this file.
#

# option definitions common to all supported networks...
# option domain-name "eibanez.cf";
# option domain-name-servers ns1.data.eibanez.cf;

default-lease-time 600;
max-lease-time 7200;

# The ddns-updates-style parameter controls whether or not the server will
# attempt to do a DNS update when a lease is confirmed. We default to the
# behavior of the version 2 packages ('none', since DHCP v2 didn't
# have support for DDNS.)
ddns-update-style none;

# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
authoritative;

# A slightly different configuration for an internal subnet.

subnet 192.168.174.0 netmask 255.255.255.0 {
  range 192.168.174.100 192.168.174.199;
  option domain-name-servers ns1.data.eibanez.cf;
  option domain-name "data.eibanez.cf";
  option subnet-mask 255.255.255.0;
  option routers 192.168.174.2;
  option broadcast-address 192.168.174.255;
  default-lease-time 600;
  max-lease-time 7200;
}

# Reserving addresses through the MAC address for our machines
host ns1 {
  hardware ethernet 00:50:56:3C:2F:B8;
  fixed-address 192.168.174.20;
}

host dhcp1 {
  hardware ethernet 00:0C:29:57:59:10;
  fixed-address 192.168.174.10;
}

host ftp1 {
  hardware ethernet 00:0c:29:fb:8e:67;
  fixed-address 192.168.174.30;
}

host host1 {
  hardware ethernet 00:50:56:39:A8:2D;
  fixed-address 192.168.174.100;
}

host host2 {
  hardware ethernet 00:50:56:3A:C0:CF;
  fixed-address 192.168.174.101;
}
```

We can check our configuration file using this command
```shell
dhcpd -t -cf /etc/dhcp/dhcpd.conf
```
![](https://hackmd.io/_uploads/HJ3Zb4Xwn.png)

Now we can restart the service and check if it's working
```shell
sudo systemctl enable isc-dhcp-server
sudo systemctl restart isc-dhcp-server.service
sudo systemctl status isc-dhcp-server
```
![](https://hackmd.io/_uploads/Hy4cbEXv3.png)

We can also check our logs using journalctl
```shell
journalctl -u isc-dhcp-server
```

## DNS Server
### Setup of Bind9
First let's change our hostname
```shell
sudo hostnamectl set-hostname ns1
```

Then we'll update our repositories
```shell
sudo apt update
```

To begin the installation process, open the terminal and execute the following command:
```shell

sudo apt install bind9 bind9utils bind9-doc
```
### Configuration files
#### named.conf.options
Let's modify our options file
```shell
sudo nano /etc/bind/named.conf.options
```

We'll uncomment and add the following lines
```shell
options {
        directory "/var/cache/bind";
        recursion yes;                 # enables recursive queries
        allow-recursion { localnets; };# Allow recursion to local network
        listen-on { 192.168.174.20; }; # ns1 private IP address
        allow-transfer { none; };      # disable zone transfers by default
        forwarders {                   # Cloudflare forwarders
            1.1.1.1;
            1.0.0.1;
        };
        dnssec-validation auto;
        listen-on-v6 { any; };
};
```
![](https://hackmd.io/_uploads/S1F5o47w3.png)

#### named.conf.local
Let's modify our local file
```shell
sudo nano /etc/bind/named.conf.local
```

Let's add our primary and reverse zones
```shell
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
# Primary Zone
zone "data.eibanez.cf" {
    type primary;
    file "/etc/bind/zones/db.data.eibanez.cf";          # Zone file path
};

zone "174.168.192.in-addr.arpa" {
    type primary;
    file "/etc/bind/zones/db.174.168.192";              # Reverse zone 192.168.174.0/24 subnet
};
```
![](https://hackmd.io/_uploads/SJcto47D3.png)

### Zone Files
#### /etc/bind/zones Directory
Create the directory:
```shell
sudo mkdir /etc/bind/zones
```

Copy the local file to the zone directory:
```shell
sudo cp /etc/bind/db.local /etc/bind/zones/db.data.eibanez.cf
```

Edit the zone file:
```shell
sudo nano /etc/bind/zones/db.data.eibanez.cf
```

Define the Start of Authority (SOA) using the FQDN of your primary DNS server (ns1). Increment the serial value +1:
```shell
;
$TTL    604800
@       IN      SOA     ns1.data.eibanez.cf. root.data.eibanez.cf. (
                              3         ; Serial
```

We'll delete the last 3 entries as they're the localhost ones.
Specify the NS records that point to your DNS server:
```shell
; name servers - NS records
        IN      NS      ns1.data.eibanez.cf.
```

Add the A records for your DNS server and clients:
```shell
; name servers - A records
ns1.data.eibanez.cf.          IN      A       192.168.174.20

; 192.168.174.0/24 - A records
dhcp1.data.eibanez.cf.        IN      A      192.168.174.10
ftp1.data.eibanez.cf.         IN      A      192.168.174.30
host1.data.eibanez.cf.        IN      A      192.168.174.100
host2.data.eibanez.cf.        IN      A      192.168.174.101
```
![](https://hackmd.io/_uploads/r1WuiN7Dn.png)

#### Reverse Zone File
Copy the example file to create the reverse zone:

```shell
sudo cp /etc/bind/db.127 /etc/bind/zones/db.174.168.192
```

Edit the reverse zone file:
```shell
sudo nano /etc/bind/zones/db.174.168.192
```

Increment the serial value in the SOA record and add FQDN:
```shell
@       IN      SOA     ns1.data.eibanez.cf. root.data.eibanez.cf. (
                              2         ; Serial
```

Add the NS records for your DNS servers:
```shell
; name servers - NS records
        IN      NS      ns1.data.eibanez.cf.
```

Add the PTR records for the IP addresses in your zone (192.168.174.0/24):
```shell
; PTR Records
10      IN      PTR     dhcp1.data.eibanez.cf.          ; 192.168.174.10
20      IN      PTR     ns1.data.eibanez.cf.            ; 192.168.174.20
30      IN      PTR     ftp1.data.eibanez.cf.           ; 192.168.174.30
100     IN      PTR     host1.data.eibanez.cf.          ; 192.168.174.100
101     IN      PTR     host2.data.eibanez.cf.          ; 192.168.174.101
```
![](https://hackmd.io/_uploads/rywJpEmw2.png)

### Final Configuration
To ensure the configuration files are correct, run the following commands:

```shell
sudo named-checkconf
sudo named-checkzone data.eibanez.cf /etc/bind/zones/db.data.eibanez.cf
sudo named-checkzone 174.168.192.in-addr.arpa /etc/bind/zones/db.174.168.192
```
Restart the BIND service:

```shell
sudo systemctl restart bind9
```

Allow Bind9 through UFW Firewall:
```shell
sudo ufw allow Bind9
```
And you should be done!
```shell
sudo systemctl status bind9
```
![](https://hackmd.io/_uploads/Bkb4CE7P3.png)

We can always check our logs with Journalctl
```shell
journalctl -u named
```

## FTP Server
### Initial setup of PROFTPD
Setting our hostname
```shell
sudo hostnamectl set-hostname ftp1
```

Updating our repositories
```shell
sudo apt update
```

Now we can begin the installation
```shell
sudo apt -y install proftpd
```

If we want to implement tls/sftp we'll also install the crypto mod
```shell
sudo apt -y install proftpd-mod-crypto
```

Now we can check if the service is running through systemctl
```shell
sudo systemctl status proftpd
```
![](https://hackmd.io/_uploads/HJuZNumPn.png)

Logs with journalctl
```shell
journalctl -u proftpd
```

Correct configuration syntax with proftpd -t
```shell
proftpd -t
```

### User management
PROFTPD will use the regular UNIX users as the method of authentication.
We can create a new user with this command:
```shell
sudo adduser ftp1
```
![](https://hackmd.io/_uploads/B1jUQO7Dh.png)

We'll log with the user
```shell
su ftp1
```

Place a few things to play with in it's home directory
![](https://hackmd.io/_uploads/r15Em_Xvh.png)

You can quit this user's session with the command:
```shell
exit
```
![](https://hackmd.io/_uploads/Hyx0X_QP3.png)

We can also modify their home directory with this command where **/var/www/** is the path to the new home.
```shell
sudo usermod -m -d /var/www/ username
```

### Use of FTP
We can log from our network using the command:
```shell
ftp ftp1@192.168.174.30
```
You'll be prompted for the password
![](https://hackmd.io/_uploads/SktI4_Xw2.png)

Once inside you can type **help** to get the commands
![](https://hackmd.io/_uploads/Sy_0EuQPn.png)

**dir** or **ls** is used to see what's inside our directory
![](https://hackmd.io/_uploads/BJM3VFXv2.png)

**delete** erases a file
![](https://hackmd.io/_uploads/SyceSY7v2.png)

**cd** to change directories
![](https://hackmd.io/_uploads/r174HK7v2.png)

**put** uploads a file from the host directory
![](https://hackmd.io/_uploads/SJaFHY7w3.png)

**get** downloads a file
![](https://hackmd.io/_uploads/SyPhBK7D3.png)

**exit** to leave ftp

### Auto-signed certificates
To enable FTPS and SFTP we need encryption, we can get an auto-signed certificates through one of these commands:
1. Install SSL in case we don't have it
```shell
sudo apt-get install openssl -y
```
2. **Gencert**
```shell
sudo proftpd-gencert
```
![](https://hackmd.io/_uploads/rydYLK7Ph.png)

or

2. **OpenSSL**
```shell
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/proftpd.key -out /etc/ssl/certs/proftpd.crt
```

Change the permissions on these files
```shell
sudo chmod 0600 /etc/ssl/private/proftpd.key
sudo chmod 0600 /etc/ssl/certs/proftpd.crt
```

### Configuration files
#### proftpd.conf
This is the general configuration file, we'll modify it using the following command:
```shell
sudo nano /etc/proftpd/proftpd.conf
```

##### Enable TLS
Uncomment this line
```shell
Include /etc/proftpd/tls.conf
```
![](https://hackmd.io/_uploads/H1L0vFQv2.png)

##### Enable SFTP
Uncomment this to enable SFTP:
```shell
Include /etc/proftpd/sftp.conf
```
![](https://hackmd.io/_uploads/By-yuFXD3.png)

##### Anonymous users
We could activate anonymous users uncommenting the following lines:
```shell
 <Anonymous ~ftp>
   User ftp
   Group nogroup
   # We want clients to be able to login with "anonymous" as well as "ftp"
   UserAlias anonymous ftp
   # Cosmetic changes, all files belongs to ftp user
   DirFakeUser on ftp
   DirFakeGroup on ftp

   RequireValidShell off

   # Limit the maximum number of anonymous logins
   MaxClients 10

   # We want 'welcome.msg' displayed at login, and '.message' displayed
   # in each newly chdired directory.
#   DisplayLogin welcome.msg
#   DisplayChdir .message
#
#   # Limit WRITE everywhere in the anonymous chroot
#   <Directory *>
#     <Limit WRITE>
#       DenyAll
#     </Limit>
#   </Directory>
#
#   # Uncomment this if you're brave.
#   # <Directory incoming>
#   #   # Umask 022 is a good standard umask to prevent new files and dirs
#   #   # (second parm) from being group and world writable.
#   #   Umask022  022
#   #   <Limit READ WRITE>
#   #     DenyAll
#   #     </Limit>
#   #       <Limit STOR>
#   #         AllowAll
#   #     </Limit>
#   # </Directory>
#
 </Anonymous>
```
#### sftp.conf
Modify the file
```shell
sudo nano /etc/proftpd/sftp.conf
```

Uncomment these lines
```shell
<IfModule mod_sftp.c>
SFTPEngine     on
Port           2222
SFTPLog        /var/log/proftpd/sftp.log
SFTPHostKey /etc/ssh/ssh_host_rsa_key
</IfModule>
```
#### tls.conf
Modify the file
```shell
sudo nano /etc/proftpd/sftp.conf
```

To get TLS working we have to uncomment these lines
```shell
<IfModule mod_tls.c>
TLSEngine                               on
TLSLog                                  /var/log/proftpd/tls.log
TLSProtocol                             SSLv23
TLSRSACertificateFile                   /etc/ssl/certs/proftpd.crt
TLSRSACertificateKeyFile                /etc/ssl/private/proftpd.key
TLSOptions                              AllowClientRenegotiations
TLSRequired                             on
</IfModule>
```
#### modules.conf
##### TLS Activation
Uncomment these lines
```shell
# Install proftpd-mod-crypto to use this module for TLS/SSL support.
LoadModule mod_tls.c
# Even these modules depend on the previous one
LoadModule mod_tls_fscache.c
LoadModule mod_tls_shmcache.c
```
![](https://hackmd.io/_uploads/B14dcKmPh.png)
##### SFTP Activation
Uncomment these lines
```shell
# Install proftpd-mod-crypto to use this module for SFTP support.
LoadModule mod_sftp.c
LoadModule mod_sftp_pam.c
```

### TLS testing
Once the required configuration files of proftpd.conf, tls.conf and modules.conf have been modified we can restart and check our service
```shell
sudo systemctl restart proftpd.service
sudo systemctl status proftpd.service
```
![](https://hackmd.io/_uploads/rJym3YQP2.png)

Now since we enabled TLSRequired On we won't be able to connect through normal ftp
![](https://hackmd.io/_uploads/BkFeatXw2.png)

We can install ftp-ssl to connect securely 
```shell
sudo apt install ftp-ssl
```

Now we can connect using the command ftp-ssl **server name**
```shell
ftp-ssl ftp1
```

We'll be prompted by the user name and password
![](https://hackmd.io/_uploads/BJrH15Qvh.png)

### SFTP Testing
We'll test our connection with the sftp command:
```shell
sftp ftp1@192.168.174.30
```

We'll accept the key by typing **yes** and enter the password
![](https://hackmd.io/_uploads/rkrPOqXDh.png)

## Hosts
Since we're in a private network we only have to enable dhcp4 in our netplan
```shell
sudo nano /etc/netplan/00-installer-config.yaml
```

And we set dhcp to **true**
```shell
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens32:    ~#Internal Network
      dhcp4: true
  version: 2
```

Now we can check our connection through this command
```shell
ip a
```
![](https://hackmd.io/_uploads/SkCJqqmw2.png)

We can confirm that our DNS is working properly through the following commands
```shell
resolvectl status
```
![](https://hackmd.io/_uploads/BJZLq9Qv3.png)

```shell
nslookup ftp1
```
![](https://hackmd.io/_uploads/B1O4qcQv3.png)

```shell
dig dhc1.data.eibanez.cf
```
![](https://hackmd.io/_uploads/Bk1oq97v2.png)

```shell
host 192.168.174.30
```
![](https://hackmd.io/_uploads/r11Cq9Xwn.png)
