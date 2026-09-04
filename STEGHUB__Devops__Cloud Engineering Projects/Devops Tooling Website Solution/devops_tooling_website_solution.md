# DevOps Tooling Website Deployment

## Project Overview

This project brings together AWS infrastructure, Linux servers, shared NFS storage, MySQL, PHP, Apache, and GitHub to deploy a three-tier tooling application.

- Infrastructure: AWS
- Web Server Linux: Red Hat Enterprise Linux 9
- Database Server: Ubuntu Linux + MySQL
- Sotrage Server: Red Hat Enterprise Linux 9 + NFS Server
- Programming Language: PHP
- Code Repository: GitHub

The architecture below illustrates the relationship between the storage, database, and web layers.

![Three-tier architecture](<./images/3-Tier.png>)

## Step 1 - Configure the NFS Storage Server

__1.__ __Launch an EC2 instance running Red Hat Enterprise Linux__

![NFS server instance](<./images/Screenshot 2026-08-31 125858.png>)

__2.__ __Configure Logical volume management on the server__

- Format the lvm as xfs
- Create 3 Logical volumes: lv-opt, lv-appa, lv-logs.
- Create mount points on /mnt directory for the logical volumes as follows:
  - Mount lv-apps on /mnt/apps - To be used by web servers
  - Mount lv-logs on /mnt/logs - To be used by web serveer logs
  - Mount lv-opt on /mnt/opt - To be used by Jenkins server in next project.

#### Create 3 volumes in the same AZ as the NFS Server ec2 each of 10GB and attache all 3 volumes one by one to the NFS Server.

![Attached NFS volumes](<./images/Screenshot 2026-08-31 125912.png>)

#### Connect to the instance through the terminal and start the configuration.

```bash
ssh -i "my-devec2key.pem" ec2-user@58.84.60.223
```
![NFS server SSH connection](<./images/Screenshot 2026-08-31 130018.png>)

#### Run `lsblk` to identify the block devices attached to the instance. All devices in Linux reside in /dev/ directory. Inspect with ```ls /dev/``` and ensure all 3 newly created devices are there. Their name will likely be ```xvdf```, ```xvdg``` and ```xvdh```

```bash
lsblk
```
![Attached block devices](<./images/Screenshot 2026-08-31 130210.png>)

#### Use `fdisk` to create one partition on each of the three attached disks

```bash
sudo fdisk /dev/nvme1n1
```
![Disk partition](<./images/Screenshot 2026-08-31 130227.png>)

```bash
sudo gdisk /dev/nvme2n1
```
![Disk partition](<./images/Screenshot 2026-08-31 130316.png>)

```bash
sudo gdisk /dev/nvme3n1
```
![Disk partition](<./images/Screenshot 2026-08-31 130401.png>)

#### Run `lsblk` again to confirm that the new partitions are visible

```bash
lsblk
```
![Created partitions](<./images/lsblk2.png>)

#### Install the LVM utilities required for storage management

```bash
sudo yum install lvm2 -y
```
![LVM installation](<./images/Screenshot 2026-08-31 130534.png>)

#### Convert the three partitions into LVM physical volumes and verify them with `pvs`

```bash
sudo pvcreate /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
sudo pvs
```
![Physical volumes](<./images/Screenshot 2026-08-31 130803.png>)

#### Combine the three physical volumes into a volume group named `webdata-vg`, then verify it with `vgs`

```bash
sudo vgcreate webdata-vg /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
sudo vgs
```
![Volume group](<./images/Screenshot 2026-08-31 130915.png>)

#### Create the three logical volumes `lv-apps`, `lv-logs`, and `lv-opt`, then confirm them with `lvs`

```bash
sudo lvcreate -n lv-apps -L 9G webdata-vg
sudo lvcreate -n lv-logs -L 9G webdata-vg
sudo lvcreate -n lv-opt -L 9G webdata-vg

sudo lvs
```
![Logical volumes](<./images/Screenshot 2026-08-31 131114.png>)

#### Review the complete LVM configuration

```bash
sudo vgdisplay -v   #view complete setup, VG, PV and LV
```
![LVM configuration](<./images/Screenshot 2026-08-31 131216.png>)

```bash
lsblk
```
![Attached block devices](<./images/Screenshot 2026-08-31 131437.png>)

#### Use ```mkfs -t xfs``` to format the logical volumes instead of ext4 filesystem

```bash
sudo mkfs -t xfs /dev/webdata-vg/lv-apps
sudo mkfs -t xfs /dev/webdata-vg/lv-logs
sudo mkfs -t xfs /dev/webdata-vg/lv-opt
```
![XFS filesystems](<./images/Screenshot 2026-08-31 131443.png>)

#### Create the mount directories under `/mnt`

```bash
sudo mkdir /mnt/apps
sudo mkdir /mnt/logs
sudo mkdir /mnt/opt
```
```bash
sudo mount /dev/webdata-vg/lv-apps /mnt/apps
sudo mount /dev/webdata-vg/lv-logs /mnt/logs
sudo mount /dev/webdata-vg/lv-opt /mnt/opt
```
![Mounted directories](<./images/Screenshot 2026-08-31 131705.png>)

__3.__ __Install the NFS server, enable it at boot, and verify that the service is running__.

```bash
sudo yum update -y
sudo yum install nfs-utils -y
```
![NFS installation](<./images/Screenshot 2026-08-31 132358.png>)

```bash
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```
![NFS service status](<./images/Screenshot 2026-08-31 132630.png>)


__4.__ __Make the NFS exports available to the Web Server subnet using its IPv4 CIDR range. For simplicity, all 3 Web Servers are installed in the same subnet but in production set up, each tier should be separated inside its own subnet or higher level of security__

#### Set ownership and permissions so the Web Servers can read, write, and execute files on the shared NFS directories.

```bash
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

#### Define the subnet-based access rules for NFS clients (example Subnet Cidr - 172.31.32.0/20)

```bash
sudo vi /etc/exports

/mnt/apps 172.31.0.0/16(rw,sync,no_all_squash,no_root_squash)
/mnt/logs 172.31.0.0/16(rw,sync,no_all_squash,no_root_squash)
/mnt/opt 172.31.0.0/16(rw,sync,no_all_squash,no_root_squash)

`We use 16 because, it offers a wider range for the instances in our subnet.`

sudo exportfs -arv
```
![NFS export configuration](<./images/Screenshot 2026-09-01 141712.png>)
![Exported NFS filesystems](<./images/Screenshot 2026-09-01 142306.png>)


__5.__ __Identify the NFS ports and allow the required traffic through the security group__

```bash
rpcinfo -p | grep nfs
```
![NFS port information](<./images/Screenshot 2026-08-31 134239.png>)

__Note__: For NFS Server to be accessible from the client, the following ports must be opened: TCP 111, UDP 111, UDP 2049, NFS 2049.
Set the Web Server subnet cidr as the source

![NFS security group rule](<./images/Screenshot 2026-08-31 135304.png>)


## Step 2 - Configure the Database Layer

#### Create a separate Ubuntu EC2 instance to act as the Database Server

![Database server instance](<./images/Screenshot 2026-08-31 135728.png>)

#### Connect to the database instance and begin the initial configuration.

```bash
ssh -i "my-devec2key.pem" ubuntu@3.93.169.218
```
![Database server SSH connection](<./images/Screenshot 2026-08-31 135930.png>)

#### Update the Ubuntu package index and upgrade installed packages

```bash
sudo apt update && sudo apt upgrade -y
```
![Ubuntu update](<./images/Screenshot 2026-08-31 140406.png>)


__1.__ __Install the MySQL database service__

#### Install mysql server

```bash
sudo apt install mysql-server
```
![MySQL installation](<./images/Screenshot 2026-08-31 140644.png>)

#### Run mysql secure script

```bash
sudo mysql_secure_installation
```
![MySQL security configuration](<./images/Screenshot 2026-08-31 140810.png>)

__2.__ __Create a database named `tooling`__

__3.__ __Create a MySQL user named `webaccess`__

__4.__ __Grant `webaccess` full privileges on the `tooling` database while restricting access to the Web Server subnet__

```sql
sudo mysql

CREATE DATABASE tooling;
CREATE USER 'webaccess'@'172.31.0.0/20' IDENTIFIED WITH mysql_native_password BY '$Admin1';
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'172.31.0.0/16' WITH GRANT OPTION;
FLUSH PRIVILEGES;
show databases;

use tooling;
select host, user from mysql.user;
exit
```
![Tooling database](<./images/Screenshot 2026-08-31 145646.png>)
![Tooling database](<./images/Screenshot 2026-08-31 162115.png>)
![Tooling database](<./images/Screenshot 2026-09-03 020608.png>)
![Tooling database](<./images/Screenshot 2026-09-01 143909.png>)

#### Configure MySQL's bind address and restart the service

```bash
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf

sudo systemctl restart mysql
sudo systemctl status mysql
```

![MySQL bind address](<./images/Screenshot 2026-08-31 162455.png>)
![MySQL service status](<./images/Screenshot 2026-08-31 162659.png>)


#### Allow MySQL traffic on TCP port 3306 through the Database Server security group

Access to the DB Server is allowed only from the ```Subnet Cidr``` configured as source.

![Database security group rule](<./images/Screenshot 2026-08-31 163756.png>)


## Step 3 - Configure the Web Tier

There is need to ensure that the Web Servers can serve the same content from a shared storage solution, in this case - NFS and MySQL database. One DB can be accessed for ```read``` and ```write``` by multiple clients.
For storing shared files that the Web Servers will use, NFS is utilized and previousely created Logical Volume ```lv-apps``` is mounted to the folder where Apache stores files to be served to the users (/var/www).

This approach makes the Web server ```stateless``` which means they can be replaced when needed and data (in the database and on NFS) integrtity is preserved

In further steps, the following was done:
- Configured NFS (This step was done on all 3 servers)
- Deployed a tooling application to the Web Servers into a shared NFS folder
- Configured the Web Server to work with a single MySQL database

#### Web Server 1 Configuration

__1.__ __Launch a new EC2 instance with RHEL Operating System__

![Web Server 1 instance](<./images/Screenshot 2026-08-31 170154.png>)

__2.__ __Install the NFS client utilities__

```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```
![NFS installation](<./images/Screenshot 2026-08-31 170450.png>)

__3.__ __Mount ```/var/www/``` and target the NFS server's export for ```apps```__.
NFS Server private IP address = 172.31.22.164

```bash
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.22.164:/mnt/apps /var/www
```

__4.__ __Verify the NFS mount with `df -h` and configure it in `/etc/fstab` so it remains available after reboot.__

![NFS application mount](<./images/Screenshot 2026-09-01 143113.png>)


```bash
sudo vi /etc/fstab
```

Add the following line
```bash
172.31.22.164:/mnt/apps /var/www nfs defaults 0 0
```
![Persistent mount configuration](<./images/Screenshot 2026-09-01 143455.png>)


__5.__ __Install Apache, the Remi repository, and the required PHP packages__

```bash
sudo yum install httpd -y
```
![Apache installation](<./images/Screenshot 2026-09-01 143550.png>)

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
```
![EPEL installation](<./images/install-epel-web1.png>)

```bash
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-10.rpm
```
![Remi repository installation](<./images/Screenshot 2026-09-01 145915.png>)

```bash
sudo dnf module reset php
```
![PHP module reset](<./images/reset-php.png>)

```bash
sudo dnf module enable php:remi-8.5
```
![PHP 8.5 module](<./images/Screenshot 2026-09-01 150243.png>)

```bash
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
```
![PHP installation](<./images/Screenshot 2026-09-01 150322.png>)

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl status php-fpm

sudo setsebool -P httpd_execmem 1  # Allows the Apache HTTP server (httpd) to execute memory that it can also write to. This is often needed for certain types of dynamic content and applications that may need to generate and execute code at runtime.
sudo setsebool -P httpd_can_network_connect=1   # Allows the Apache HTTP server to make network connections to other servers.
sudo setsebool -P httpd_can_network_connect_db=1  # allows the Apache HTTP server to connect to remote database servers.
```
![PHP-FPM service](<./images/Screenshot 2026-09-01 151101.png>)


### Web Server 2 Configuration

__1.__ __Launch another new EC2 instance with RHEL Operating System__
![Web Server 2 instance](<./images/Screenshot 2026-09-01 151725.png>)

__2.__ __Install the NFS client utilities__

```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```
![NFS installation](<./images/Screenshot 2026-09-01 152331.png>)

__3.__ __Mount ```/var/www/``` and target the NFS server's export for ```apps```__.
NFS Server private IP address = 172.31.22.164

```bash
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.22.164:/mnt/apps /var/www
```

__4.__ __Verify the NFS mount with `df -h` and configure it in `/etc/fstab` so it remains available after reboot.__

![NFS application mount](<./images/Screenshot 2026-09-01 152229.png>)


```bash
sudo vi /etc/fstab
```

Add the following line
```bash
172.31.22.164:/mnt/apps /var/www nfs defaults 0 0
```
![Persistent mount configuration](<./images/Screenshot 2026-09-01 152837.png>)

__5.__ __Install Apache, the Remi repository, and the required PHP packages__

```bash
sudo yum install httpd -y
```
![Apache installation](<./images/Screenshot 2026-09-01 152914.png>)

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```
![EPEL installation](<./images/Screenshot 2026-09-01 153052.png>)

```bash
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm
```
![Remi repository installation](<./images/Screenshot 2026-09-01 153125.png>)

```bash
sudo dnf module reset php
```
![PHP module reset](<./images/Screenshot 2026-09-01 153207.png>)

```bash
sudo dnf module enable php:remi-8.5
```
![PHP 8.5 module](<./images/PHP 8.5 module.png>)

```bash
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
```
![PHP installation](<./images/Screenshot 2026-09-01 153318.png>)

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl status php-fpm
sudo setsebool -P httpd_execmem 1
```
![PHP-FPM service](<./images/Screenshot 2026-09-01 153356.png>)


### Web Server 3 Configuration

__1.__ __Launch another new EC2 instance with RHEL Operating System__
![Web Server 3 instance](<./images/Screenshot 2026-09-03 005154.png>)

__2.__ __Install the NFS client utilities__

```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```
![NFS installation](<./images/Screenshot 2026-09-03 005329.png>)

__3.__ __Mount ```/var/www/``` and target the NFS server's export for ```apps```__.
NFS Server private IP address = 172.31.22.164

```bash
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.22.164:/mnt/apps /var/www
```

__4.__ __Verify the NFS mount with `df -h` and configure it in `/etc/fstab` so it remains available after reboot.__

![NFS application mount](<./images/Screenshot 2026-09-03 005533.png>)


```bash
sudo vi /etc/fstab
```

Add the following line
```bash
172.31.22.164:/mnt/apps /var/www nfs defaults 0 0
```
![Persistent mount configuration](<./images/Screenshot 2026-09-03 005638.png>)

__5.__ __Install Apache, the Remi repository, and the required PHP packages__

```bash
sudo yum install httpd -y
```
![Apache installation](<./images/Screenshot 2026-09-03 005803.png>)

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
```
![EPEL installation](<./images/Screenshot 2026-09-03 005858.png>)

```bash
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-10.rpm
```
![Remi repository installation](<./images/Screenshot 2026-09-03 005943.png>)

```bash
sudo dnf module reset php
```
![PHP module reset](<./images/Screenshot 2026-09-02 120203.png>)

```bash
sudo dnf module enable php:remi-8.5
```
![PHP 8.5 module](<./images/Screenshot 2026-09-03 010007.png>)

```bash
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
```
![PHP installation](<./images/Screenshot 2026-09-03 010127.png>)

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl status php-fpm
sudo setsebool -P httpd_execmem 1
```
![PHP-FPM service](<./images/Screenshot 2026-09-03 010210.png>)


__6.__ __Verify that Apache files and directories are availabel on the Web Servers in ```/var/www``` and also on the NFS Server in ```/mnt/apps```. If the same files are present in both, it means NFS was mounted correctly.__
test.txt file was created from Web Server 1, and it was accessible from Web Server 2 and 3.

![create file](<./images/Screenshot 2026-09-03 010607.png>)
![check test file](<./images/Screenshot 2026-09-03 011049.png>)


__7.__ __Locate the log folder for Apache on the Web Servers and mount it to NFS server's export for logs. Repeat ```step 4``` to ensure the mount point persists after reboot__.

```bash
sudo vi /etc/fstab
```

Add the following line
```bash
172.31.22.164:/mnt/logs /var/log/httpd nfs defaults 0 0
```

![mount and persist logs](<./images/Screenshot 2026-09-03 011627.png>)

__8.__ __Create a fork of the tooling source repository from the `StegHub GitHub Account`__

![Forked repository](<./images/Screenshot 2026-09-03 012211.png>)


__9.__ __Deploy the tooling application to the shared Web Server directory, placing the repository's `html` content in `/var/www/html`__

#### Install Git

![Git installation](<./images/Screenshot 2026-09-03 012448.png>)

#### Initialize the directory and clone the tooling repository

Ensure to clone the forked repository

![Git repository setup](<./images/Screenshot 2026-09-03 012748.png>)

![Application deployment](<./images/Screenshot 2026-09-03 013249.png>)

__Note__:
Acces the website on a browser

- Ensure TCP port 80 is open on the Web Server.
- If ```403 Error``` occur, check permissions to the ```/var/www/html``` folder and also disable ```SELinux```
```bash
sudo setenforce 0
```
To make the change permanent, open selinux file and set selinux to disable.

```bash
sudo vi /etc/sysconfig/selinux

SELINUX=disabled

sudo systemctl restart httpd
```
![SELinux configuration](<./images/Screenshot 2026-09-03 013713.png>)


__10.__ __Update the application's database settings in `/var/www/html/functions.php` and import the `tooling-db.sql` schema into MySQL__
```sudo mysql -h <db-private-IP> -u <db-username> -p <db-password < tooling-db.sql```

```bash
sudo vi /var/www/html/functions.php
```
![Application database configuration](<./images/Screenshot 2026-09-03 014214.png>)

```sql
sudo mysql -h 172.31.22.193 -u webaccess -p tooling < tooling-db.sql
```
![](<./images/Screenshot 2026-09-03 020831  .png>)

#### Access the database server from Web Server

```sql
sudo mysql -h 172.31.22.139 -u webaccess -p
```
![Web Server database connection](<./images/Screenshot 2026-09-03 021037.png>)

__11.__ __Add a new administrator account to MySQL using username `myuser` and password `password`__

```sql
INSERT INTO users(id, username, password, email, user_type, status) VALUES (2, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
```
![Administrator database record](<./images/Screenshot 2026-09-03 021554.png>)


__12.__ __Open the application using the Web Server public IP at `http://<Web-Server-public-IP-address>/index.php` and verify that the `myuser` account can log in.__

### For Web Server 2 and 3

__Temporarily disable SELinux enforcement__

```bash
sudo setenforce 0

SELINUX=disabled
```

![SELinux configuration](<./images/Screenshot 2026-09-03 013713.png>)

__Open the deployed application and verify that it is reachable__

![Tooling application](<./images/Screenshot 2026-09-03 023112.png>)


