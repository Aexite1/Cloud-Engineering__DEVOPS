# Web Solution With WordPress

This project sets up a WordPress application using separate EC2 instances for the web and database layers. LVM is used to organize the storage attached to both servers, while Apache, PHP, WordPress, and MySQL provide the application stack.

## Step 1 - Configure the Web Server

### 1. Create the Web Server and Attach Storage

Launch a RedHat EC2 instance to act as the **Web Server**. In the same Availability Zone, create three 10 GB EBS volumes and attach them to the instance.

![Instance detail](<./images/Screenshot 2026-08-16 133439.png>)
![Security rules](<./images/Screenshot 2026-08-16 133454.png>)
![Web server volumes](<./images/Screenshot 2026-08-16 133717.png>)

### 2. Connect to the Server

Open a terminal and connect to the EC2 instance through SSH.

```bash
ssh -i "ec2key.pem" ec2-user@3.80.101.79
```

![Web server SSH connection](<./images/Screenshot 2026-08-16 134008.png>)

### 3. Check the Attached Block Devices

Use `lsblk` to confirm that the storage devices are visible to the operating system. Linux exposes devices under `/dev/`, so `ls /dev/` can also be used to inspect them. The attached disks may appear as `nvme0n0f`, `nvme0n0g`, and `nvme0n0h`.

```bash
lsblk
```

![Attached block devices](<./images/Screenshot 2026-08-16 134113.png>)

### 4. Check Existing Disk Usage

Run `df -h` to review mounted filesystems and available disk space before making the storage changes.

```bash
df -h
```

![Mounted filesystems and free space](<./images/Screenshot 2026-08-16 134147.png>)

### 5. Create Partitions

Use `gdisk` or `fdisk` to create one partition on each of the three attached disks.

For the first disk:

```bash
sudo fdisk /dev/nvme1n1
```

![Partition for nvme1n1f](<./images/Screenshot 2026-08-16 184626.png>)

For the second disk:

```bash
sudo fdisk /dev/nvme2n1
```

![Partition for nvme2n1g](<./images/Screenshot 2026-08-16 184800.png>)

For the third disk:

```bash
sudo gdisk /dev/nvme3n1
```

![Partition for nvme0n0h](<./images/Screenshot 2026-08-16 184815.png>)

### 6. Confirm the New Partitions

Run `lsblk` again and verify that the partitions created on the three disks are present.

```bash
lsblk
```

![New disk partitions](<./images/Screenshot 2026-08-16 185018.png>)

### 7. Install LVM

Install the LVM package so the attached disks can be managed as physical volumes, a volume group, and logical volumes.

```bash
sudo yum install lvm2 -y
```

![LVM installation](<./images/Screenshot 2026-08-16 185228.png>)

### 8. Create the Physical Volumes

Convert the three disk partitions into LVM physical volumes. Then use `pvs` to verify the result.

```bash
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

sudo pvs
```
![Physical volumes](<./images/Screenshot 2026-08-16 185512.png>)
![Physical volumes](<./images/Screenshot 2026-08-16 185753x.png>)

### 9. Create the Volume Group

Combine the three physical volumes into one volume group called `webdata-vg`. Confirm the volume group with `vgs`.

```bash
sudo vgcreate webdata-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

sudo vgs
```

![Web volume group](<./images/Screenshot 2026-08-16 185512.png>)
![Web volume group](<./images/Screenshot 2026-08-16 185753.png>)
### 10. Create Logical Volumes

Create two logical volumes inside `webdata-vg`. `apps-lv` will hold the website files, while `logs-lv` will be used for system logs.

```bash
sudo lvcreate -n apps-lv -L 14G webdata-vg

sudo lvcreate -n logs-lv -L 14G webdata-vg

sudo lvs
```

![Logical volumes](<./images/Screenshot 2026-08-16 190035.png>)
![Logical volumes](<./images/Screenshot 2026-08-16 190101.png>)
### 11. Review the Complete LVM Configuration

Use `vgdisplay` to inspect the volume group, physical volumes, and logical volumes.

```bash
sudo vgdisplay -v   # view complete setup, VG, PV and LV
```

![Detailed volume group configuration](<./images/Screenshot 2026-08-16 190319.png>)
![Detailed volume group configuration](<./images/Screenshot 2026-08-16 190407.png>)
You can also use `lsblk` to see the complete block-device layout.

```bash
lsblk
```

![Complete block-device layout](<./images/list-block3.png>)

### 12. Format the Logical Volumes

Format both logical volumes with the `ext4` filesystem.

```bash
sudo mkfs.ext4 /dev/webdata-vg/apps-lv

sudo mkfs.ext4 /dev/webdata-vg/logs-lv
```

![Formatting the filesystems](<./images/Screenshot 2026-08-16 190811s.png>)

### 13. Create the Required Mount Points

Create the directory for the WordPress website and another directory that will temporarily hold a backup of the existing log files.

```bash
sudo mkdir -p /var/www/html

sudo mkdir -p /home/recovery/logs
```

### 14. Mount the Website Logical Volume

Mount `apps-lv` at `/var/www/html`, which will become the location for the WordPress application files.

```bash
sudo mount /dev/webdata-vg/apps-lv /var/www/html
```

### 15. Back Up the Existing Log Files

Before mounting the new logical volume over `/var/log`, copy the existing log data to the recovery directory.

```bash
sudo rsync -av /var/log /home/recovery/logs
```

### 16. Mount the Log Logical Volume

Mount `logs-lv` on `/var/log`. Because the mount hides the previous contents of that directory, the log files were backed up in the previous step.

```bash
sudo mount /dev/webdata-vg/logs-lv /var/log
```

![14, 15 & 16](<./images/Screenshot 2026-08-16 192001.png>)

### 17. Restore the Previous Log Data

Copy the backed-up log files into the newly mounted `/var/log` filesystem.

```bash
sudo rsync -av /home/recovery/logs/log/ /var/log
```

![Restoring log files](<./images/Screenshot 2026-08-16 192247.png>)

### 18. Make the Mounts Persistent

Edit `/etc/fstab` so that the logical volumes are mounted automatically whenever the server starts.

First obtain the UUID values:

```bash
sudo blkid   # To fetch the UUID

sudo vi /etc/fstab
```

Use the appropriate UUID values in `/etc/fstab`, following the format already shown in the file. Remove the leading and trailing quotation marks from the UUID values.

![Updating fstab](<./images/Screenshot 2026-08-16 192526.png>)

### 19. Test the Configuration

Test the entries in `/etc/fstab`, reload the system daemon, and check the available filesystems.

```bash
sudo mount -a   # Test the configuration

sudo systemctl daemon-reload

df -h   # Verify the setup
```

![Verified web server storage setup](<./images/Screenshot 2026-08-16 193950.png>)

## Step 2 - Configure the Database Server

Create a second RedHat EC2 instance for the **Database Server**. The storage configuration follows the same general LVM process used for the web server, but this server uses a `db-lv` logical volume mounted at `/db`.

### 1. Create and Attach Database Storage

Create three 10 GB volumes in the same Availability Zone as the database EC2 instance and attach all three volumes to the server.

![Database instance details](<./images/Screenshot 2026-08-16 143210.png>)
![Database security rules](<./images/Screenshot 2026-08-16 143221.png>)
![Database volumes](<./images/db-volume.png>)

### 2. Connect to the Database Server

Open a terminal and connect to the database EC2 instance.

```bash
ssh -i "ec2key.pem" ec2-user@98.81.191.98
```

![Database server SSH connection](<./images/Screenshot 2026-08-16 144551.png>)

### 3. Inspect the Attached Disks

Use `lsblk` to identify the three attached disks. They may appear as `nvme1n1`, `nvme2n1`, and `nvme3n1`.

```bash
lsblk
```

![Database block devices](<./images/Screenshot 2026-08-16 223347.png>)

### 4. Partition the Disks

Create one partition on each of the three disks with `fdisk` or `gdisk`.

```bash
sudo fdisk /dev/nvme0n0f
```

```bash
sudo fdisk /dev/nvme0n0g
```

```bash
sudo fdisk /dev/nvme0n0h
```

![Database disk partition](<./images/Screenshot 2026-08-16 223440.png>)

### 5. Confirm the Partitions

Run `lsblk` to confirm that the new partitions have been created.

```bash
lsblk
```

![Database partitions](<./images/```.png>)

### 6. Install LVM

Install `lvm2` on the database server.

```bash
sudo yum install lvm2 -y
```

![LVM installation on database server](<./images/Screenshot 2026-08-16 224520.png>)

### 7. Create the Database Physical Volumes and Volume Group

Convert the three partitions into physical volumes, then combine them into a volume group named `database-vg`.

```bash
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

sudo pvs
```

```bash
sudo vgcreate database-vg /dev/nvme1n1p1 /dev/nvme2n1p1 /dev/nvme3n1p1

sudo vgs
```
![Database PV and VG](<./images/Screenshot 2026-08-17 082146.png>)
![Database PV and VG](<./images/Screenshot 2026-08-17 084334.png>)

### 8. Create the Database Logical Volume

Create a single 20 GB logical volume called `db-lv`.

```bash
sudo lvcreate -n db-lv -L 14G database-vg
sudo lvcreate -n logs-lv -L 14G database-vg
sudo lvs
```

![Database logical volume](<./images/Screenshot 2026-08-17 084912.png>)

### 9. Format and Mount the Database Volume

Format `db-lv` with `ext4` and mount it at `/db`.

```bash
sudo mkfs.ext4 /dev/database-vg/db-lv
```

```bash
sudo mount /dev/database-vg/db-lv /db
```

![Database filesystem](<./images/Screenshot 2026-08-17 085341.png>)
![mnt to /db](<./images/mnt to /db.png>)

### 10. Configure Persistent Mounting

Find the UUID of the database logical volume.

```bash
sudo blkid
```

![Database volume UUID](<./images/Screenshot 2026-08-17 092813.png>)

Edit `/etc/fstab` and add the mount configuration using the UUID. Remove the quotation marks surrounding the UUID.

```bash
sudo vi /etc/fstab
```

![Database fstab configuration](<./images/Screenshot 2026-08-17 092801.png>)

Test the configuration and confirm the mounted filesystem.

```bash
sudo mount -a   # Test the configuration

sudo systemctl daemon-reload

df -h   # Verify the setup
```

![Verified database storage setup](<./images/Screenshot 2026-08-17 142629.png>)

## Step 3 - Install WordPress on the Web Server

### 1. Update the Web Server

Start by updating the system packages.

```bash
sudo yum -y update
```

### 2. Install Apache and Required Packages

Install Apache (`httpd`) together with the packages required by the web application.

```bash
sudo yum install wget httpd php-fpm php-json php-mysqlnd
```

![Apache installation](<./images/Screenshot 2026-08-25 084448.png>)

### 3. Install a Current PHP Version

The project uses the EPEL and Remi repositories to obtain the required PHP packages.

#### Install EPEL

The system uses RHEL version 10. The `dnf` package manager is used for repository and module operations.

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
```

![EPEL repository installation](<./images/Screenshot 2026-08-25 085002.png>)

#### Install the Required Repository Utilities

Install `dnf-utils` and the Remi repository.

```bash
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm
```

![Repository utilities and Remi](<./images/Screenshot 2026-08-25 085111.png>)

#### Check Available PHP Modules

List the available PHP module streams.

```bash
sudo dnf module list php
```

![Available PHP versions](<./images/Screenshot 2026-08-25 085311.png>)

If the current module stream needs to be replaced with PHP 8.5, reset the existing PHP module configuration first.

```bash
sudo dnf module reset php
```

![Resetting the PHP module](<./images/Screenshot 2026-08-25 085354.png>)

Enable the Remi PHP 8.5 module.

```bash
sudo dnf module enable php:remi-8.5
```

Install PHP, PHP-FPM, and the required PHP extensions.

```bash
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
```

![PHP installation](<./images/Screenshot 2026-08-25 085504.png>)

Confirm the installed PHP version.

```bash
php -v
```

![PHP version](<./images/Screenshot 2026-08-25 085552.png>)

Start PHP-FPM, configure it to start automatically, and check its status.

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl status php-fpm
```

![PHP-FPM status](<./images/Screenshot 2026-08-25 085624.png>)

### 4. Configure SELinux for Apache and PHP

Set the ownership and SELinux context required for Apache to work with the WordPress files and PHP-FPM.

```bash
sudo chown -R apache:apache /var/www/html
sudo chcon -t httpd_sys_rw_content_t /var/www/html -R
sudo setsebool -P httpd_execmem 1
sudo setsebool -P httpd_can_network_connect=1
sudo setsebool -P httpd_can_network_connect_db=1
```

![Web directory permissions](<./images/Screenshot 2026-08-25 085754.png>)

Restart Apache after making the configuration changes.

```bash
sudo systemctl restart httpd
```

![Apache restart](<./images/Screenshot 2026-08-25 085821.png>)

Open the server's public IP address in a browser and confirm that the Apache test page is accessible.

![Apache test page](<./images/Screenshot 2026-08-25 091342e.png>)

### 5. Download and Prepare WordPress

Create a temporary directory, download the WordPress archive, and extract it.

```bash
sudo mkdir wordpress && cd wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz   # Extract WordPress
```

![Downloading WordPress](<./images/Screenshot 2026-08-25 091554.png>)

Enter the extracted WordPress directory and create `wp-config.php` from the sample configuration file.

```bash
cd wordpress/
sudo cp -R wp-config-sample.php wp-config.php
```

![Creating wp-config.php](<./images/Screenshot 2026-08-26 002909.png>)

Move back to the parent directory and copy the WordPress files into the Apache document root.

```bash
cd ..
sudo cp -R wordpress/. /var/www/html/
```

![Copying WordPress into the web root](<./images/Screenshot 2026-08-27 012650.png>)

## Step 4 - Configure MySQL on the Database Server

### 1. Update the Database Server

Update the database server packages before installing MySQL.

```bash
sudo yum update -y
```

![Database server update](<./images/Screenshot 2026-08-25 181448.png>)

### 2. Install MySQL Server

Install MySQL Server on the database EC2 instance.

```bash
sudo yum install mysql-server -y
```

![MySQL Server installation](<./images/Screenshot 2026-08-25 233453.png>)

Start MySQL, enable it to start after reboot, and verify its status.

```bash
sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo systemctl status mysqld
```

![MySQL service status](<./images/Screenshot 2026-08-25 233541.png>)

### 3. Secure MySQL and Create the WordPress Database

Run the MySQL security script.

```bash
sudo mysql_secure_installation
```

![MySQL secure installation](<./images/Screenshot 2026-08-25 233724.png>)

Log into MySQL as the root user and create the WordPress database and application user. The WordPress user is restricted to the Web Server's private IP address.

```bash
sudo mysql -u root -p

CREATE DATABASE wordpress_db;
CREATE USER 'wordpress'@'172.31.31.27' IDENTIFIED WITH mysql_native_password BY 'Admin123$';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'aexite'@'172.31.19.19' WITH GRANT OPTION;
FLUSH PRIVILEGES;
show databases;
exit
```

![WordPress database creation](<./images/Screenshot 2026-08-18 200303.png>)

### 4. Configure the MySQL Bind Address

Edit the MySQL configuration and set the bind address to the private address of the Database Server rather than exposing MySQL on all interfaces.

```bash
sudo vi /etc/my.cnf
sudo systemctl restart mysqld
```

![MySQL bind address](<./images/Screenshot 2026-08-25 234216.png>)

## Step 5 - Connect WordPress to the Remote Database

### 1. Allow MySQL Traffic from the Web Server

Open TCP port `3306` in the Database Server's security group. Restrict the inbound rule to the Web Server's IP address using `/32` so that the database is not broadly exposed.

![MySQL security group rule](<./images/Screenshot 2026-08-19 085216.png>)

### 2. Install MySQL Client Support on the Web Server

Install MySQL on the Web Server so that it has the client tools required to test connectivity with the remote database.

```bash
sudo yum install mysql-server
```

![MySQL package on web server](<./images/Screenshot 2026-08-26 002554.png>)

Start and enable the service as shown in the project setup.

```bash
sudo systemctl start mysqld
sudo systemctl enable mysqld
sudo systemctl status mysqld
```

![MySQL service on web server](<./images/Screenshot 2026-08-26 002631.png>)

### 3. Update the WordPress Database Configuration

Open the WordPress configuration file.

```bash
cd /var/www/html
sudo vi wp-config.php
sudo systemctl restart httpd
```

![Opening wp-config.php](<./images/Screenshot 2026-08-26 064511.png>)

Set the database connection details so that `DB_HOST` points to the **private IP address of the Database Server**. Because both servers communicate over their private network, WordPress can reach MySQL without using the database server's public address.

![WordPress database configuration](<./images/Screenshot 2026-08-26 072427.png>)

### 4. Disable the Apache Welcome Page

Rename the default Apache welcome configuration so that it no longer takes precedence over the WordPress site.

```bash
sudo mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf_backup
```

![Disabling the Apache default page](<./images/Screenshot 2026-08-26 064609.png>)

### 5. Test the Web-to-Database Connection

From the Web Server, connect to MySQL using the Database Server's private IP address and verify that the WordPress database is visible.

```bash
sudo mysql -h 172.31.19.19 -u wordpress -p

show databases;
exit;
```

![Testing Web Server to Database Server connectivity](<./images/Screenshot 2026-08-26 065601.png>)

### 6. Complete the WordPress Installation

Open the Web Server's public IP address in a browser. The WordPress installation page should now appear.

Follow the WordPress setup screens to complete the installation and create the site.

![WordPress installation page](<./images/wp-installed.png>)
![WordPress login page](<./images/login-to-wp.png>)
![WordPress website](<./images/wp-website.png>)

## Project Completion

The WordPress environment is now configured with a dedicated Web Server and Database Server. The Web Server hosts Apache, PHP, and the WordPress application, while the Database Server provides MySQL storage. The LVM configuration also separates application, log, and database storage into dedicated logical volumes.
