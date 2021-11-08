# SharedHosting

> Project CLD1
>
> Creation of machine allowing the hosting of several websites

## Authors

- Ohan Mélodie
- Dubuis Hélène

## Introduction

### Tutorial prerequisites

> [Debian 11.1.0 amd64-netinst.iso](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-11.1.0-amd64-netinst.iso)

We chose the Debian operating system at its most recent stable release. Debian is one of the most stable Linux distributions and is initially designed to run servers.

## VM configuration

1) **New Virtual Machine:** _set_ Custom (advanced) configuration > next
2) **Virtual machine compatibility**: _set_ Workstation 16 > next
3) **Guest operating System installation**: _set_ Install disc image (iso) > *set your Debian 11 iso* > next
4) **Name the virtual machine**: _Be careful to put it in a subdirectory._
5) **Processor configuration**:
   - Number of processor: 1
   - Number of cores per processor: 1
6) **Memory for the Virtual Machine**: 2048 MB
7) **Network type**: Use Network address translation (NAT)
8) **Select I/O controller type**: LSI Logic (Recommended)
9) **Select disk type**: SCSI (Recommended)
10) **Select a disk**: Create a new virtual disk
11) **Specify Disk capacity** Maximum disk size(GB): 20.0 _and_ Split virtual disk into multiples files
12) **Specify disk File**: default

## Debian Installation

Our settings during this installation:

1) At the installer menu, choose `Install`
2) **Language**: `English`
3) **Country, area**: `Europe, Switzerland`
4) **Local settings**: `United Kingdom`
5) **Keymap**: `Swiss french`
6) **Hostname**: `sharedhosting`
7) **Domain**:  -
8) **user**: `username`
9) **Partitionning method**: `Guided - User entire disk` then `All files in one partition`
10) **Debian archive mirror**: `Switzerland` `deb.debian.org`
11) **HTTP proxy information**: -
12) **Software to install**: `standards system utilities` (_**only**_)
13) **Install the GRUB boot loader** : `Yes` in `/dev/sda`

## Environnment installation

We will now run the next commands from our user created  during the installation. To do this, we need to add it to the suddoers group

```shell
su
apt update
apt install sudo
# Add user to sudo group
/sbin/usermod -aG sudo username
# switch to this user from root
su username
```

### SSH

To connect remotely to our server we will install the SSH server: `openssh-server`

```shell
sudo apt install openssh-server
# get your IP address with this command
ip addr 
```

Check updates with `sudo systemctl status sshd`

We will keep default server configuration and log in with login and password.

Test to connect to your server with this command:

```shell
ssh username@xxx.xxx.xxx.xxx 
```

### Nginx

Let's now install and configure Nginx

```shell
sudo apt install nginx
# Remove default website
sudo rm /etc/nginx/sites-enabled/default  
sudo rm /etc/nginx/sites-available/default  
```

Check updates with `sudo systemctl status nginx`

### PHP-FPM

Same idea for the installation and configuration of php-fpm.

```shell
sudo apt install php-fpm
# Remove default config file
sudo rm /etc/php/7.4/fpm/pool.d/www.conf 
```

### Maria DB

```shell
sudo apt install mariadb-server
```

## Client website creation

> We want normal users to have execute permissions on /home because we want them to be able to go through home to access their folders.
> However, we don't want them to have read access on /home because we don't want them to be able to execute ls command and see all other client's folder.

We'll edit home permissions:
```shell
sudo chmod 751 /home
```

Then we'll create our group and user

```shell
sudo groupadd client1
sudo useradd client1 -p password -g client1 -m -d /home/client1
```

### Creation of the client file tree
> We make sure to give the user and their group full permissions to their own directory
> However, the rest of users cannot enter this folder.
> 
```shell
sudo mkdir /home/client1/www
sudo touch /home/client1/www/index.php
# Permission edition, change group ownership 
sudo chown client1:www-data -R /home/client1
sudo chmod 770 -R /home/client1 
```

### FPM file configuration

Let's edit `sudo nano /etc/php/7.4/fpm/pool.d/client1.conf`

```shell
[client1]
user = client1
group = client1
listen = /var/run/php/php-fpm-client1.sock
listen.owner = www-data
listen.group = www-data
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
chdir = /
```

Restart php fpm 
```shell
sudo service php7.4-fpm restart
``` 

### Nginx configuration for a client

Edit client config file with `nano /etc/nginx/sites-available/client1`

```shell
server {
    listen 80;
    
    root /home/client1/www;
    index index.php index.html index.htm;

    server_name client1.ch www.client1.ch;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php-fpm-client1.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

Then create the symbolic link with to activate the website

```shell
sudo ln -s /etc/nginx/sites-available/client1 /etc/nginx/sites-enabled/client1
```

Restart nginx service 
```shell 
sudo service nginx restart
```

### Configure Maria DB

```shell
# Open MariaDB
sudo mariadb
# Create database
CREATE DATABASE client1;
# Provide privileges on database to the client 
GRANT ALL PRIVILEGES ON client1.* to 'client1'@localhost IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
exit
```


## Sources
```
https://www.digitalocean.com/community/tutorials/how-to-host-multiple-websites-securely-with-nginx-and-php-fpm-on-ubuntu-14-04
https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-on-ubuntu-14-04
https://www.vultr.com/docs/use-php-fpm-pools-to-secure-multiple-web-sites
```

