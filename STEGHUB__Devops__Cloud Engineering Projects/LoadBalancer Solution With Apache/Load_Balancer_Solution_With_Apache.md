# Apache Load Balancer Configuration for a Test Website

A load balancer receives client requests and distributes them across the available web servers so that traffic is shared instead of being handled by a single server.

The following diagram illustrates the architecture used for this solution.
![Architecture](<./images/architecture-diag.png>)

## Objective
Deploy Apache as a load balancer on a separate Ubuntu EC2 instance and configure it to forward requests to the Tooling Website web servers.

## Requirements Before Configuration

The following infrastructure should already be provisioned and configured before starting the load balancer setup.

- Two RHEL10 Web Servers
- One RHEL10 NFS Server

## Requirements Before Configuration Configurations

- Apache (httpd) is up and running on both Web Servers.
- ```/var/www``` directories of both Web Servers are mounted to ```/mnt/apps``` of the NFS Server.
- All neccessary TCP/UDP ports are opened on Web, DB and NFS Servers.
- Client browsers can access both Web Servers by their Public IP addresses or Public DNS names and can open the ```Tooling Website``` (e.g, ```http://<Public-IP-Address-or-Public-DNS-Name>/index.html```)


# Step 0 - Prepare the NFS and Web Servers
- The previously created NFS server was no longer available, so the storage server was provisioned again for this setup.

## Step A - Build and Configure the NFS Storage Server

__1.__ __Provision an EC2 instance with Red Hat Enterprise Linux__
![EC2](<./images/Screenshot 2026-09-10 145936.png>)

__2.__ __Set up Logical Volume Management on the storage server__

- Format the logical volumes with XFS
- Create two logical volumes: ```lv-apps```, and ```lv-logs```.
- Create the required mount points under ```/mnt```:
  - Mount lv-apps on /mnt/apps - Used by the web servers
  - Mount lv-logs on /mnt/logs - Used for web server logs

#### Create volumes in the same AZ as the NFS Server ec2 each of 5GB and attach all 2 volumes one by one to the NFS Server.

![Attached NFS volumes](<./images/Screenshot 2026-08-31 125912.png>)

#### Connect to the instance through SSH and begin the storage configuration.

```bash
ssh -i "ec2key.pem" ec2-user@13.49.67.185
```
![NFS server SSH connection](<./images/Screenshot 2026-09-10 145936.png>)

#### Run `lsblk` to identify the block devices attached to the instance. All devices in Linux reside in /dev/ directory. Inspect with ```ls /dev/``` and ensure all 2 newly created devices are there. Their name will likely be ```nvme1n1```, ```nvme2n1``` 

```bash
lsblk
```
![Attached block devices](<./images/Screenshot 2026-09-11 151252.png>)

#### Create a partition on each attached disk with `fdisk`

```bash
sudo fdisk /dev/nvme1n1
```

```bash
sudo fdisk /dev/nvme2n1
```
![Disk partition](<./images/Screenshot 2026-09-11 151246.png>)


#### Run `lsblk` again to verify that the partitions were created

```bash
lsblk
```
![Created partitions](<./images/Screenshot 2vvv.png>)

#### Install the LVM tools needed to manage the storage volumes

```bash
sudo yum install lvm2 -y
``` 

#### Initialize the three partitions as LVM physical volumes and verify them with `pvs`

```bash
sudo pvcreate /dev/nvme1n1p1 /dev/nvme2n1p1
```
![Physical volumes](<./images/Screenshot 2026-09-11 151544.png>)

#### Create the `webdata-vg` volume group from the three physical volumes and confirm it with `vgs`

```bash
sudo vgcreate vlgrp /dev/nvme1n1p1 /dev/nvme2n1p1
```

#### Create the two logical volumes and use `lvs` to verify the result

```bash
sudo lvcreate -n lv-apps -L 4.5G webdata-vg
sudo lvcreate -n lv-logs -L 4.5G webdata-vg

```
![Logical volumes](<./images/Screenshot 2026-09-11 151851.png>)


#### Format the logical volumes with XFS rather than ext4

```bash
sudo mkfs -t xfs /dev/webdata-vg/lv-apps
sudo mkfs -t xfs /dev/webdata-vg/lv-logs
sudo mkfs -t xfs /dev/webdata-vg/lv-opt
```

#### Create the required mount directories under `/mnt`

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
![Mounted directories](<./images/Screenshot 2026-09-11 152545.png>)

__3.__ __Install NFS, enable the service at startup, and confirm that it is running__.

```bash
sudo yum update -y
sudo yum install nfs-utils -y
```
![NFS installation](<./images/Screenshot 2026-09-11 152732.png>)

```bash
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```


__4.__ __Make the NFS exports available to the Web Server subnet using its IPv4 CIDR range. For simplicity, all 2 Web Servers are installed in the same subnet but in production set up, each tier should be separated inside its own subnet or higher level of security__

#### Set ownership and permissions on the shared NFS directories so the web servers can access the files.

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
![NFS export configuration](<./images/Screenshot 2026-09-11 153336.png>)
![Exported NFS filesystems](<./images/Screenshot 2026-09-11 153429.png>)


__5.__ __Identify the NFS ports and permit the required traffic through the security group__

```bash
rpcinfo -p | grep nfs
```
![NFS port information](<./images/Screenshot 2026-09-11 153700.png>)

__Note__: For NFS Server to be accessible from the client, the following ports must be opened: TCP 111, UDP 111, UDP 2049, NFS 2049.
Set the Web Server subnet cidr as the source


## Step B - Prepare the Web Tier

The web tier must serve the same application content through shared NFS storage while using a centralized MySQL database. The database can be accessed for both ```read``` and ```write``` operations by multiple clients.
NFS provides the shared application storage. The previously created ```lv-apps``` logical volume is exported and mounted at ```/var/www```, where Apache serves the application files.

This design keeps the web servers ```stateless```. A web server can therefore be replaced or recreated without losing application data because persistent data remains on the database and NFS storage.

The web-tier preparation consists of the following tasks:
- Configured NFS (This step was done on the servers)
- Deployed a test application to the Web Servers into a shared NFS folder

#### Web Server 1 Configuration

__1.__ __Provision a new RHEL EC2 instance for Web Server 1__
__2.__ __Install the NFS client packages__

```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```
![NFS installation](<./images/Screenshot 2026-09-11 153855.png>)

__3.__ __Mount the NFS ```apps``` export at ```/var/www```__.
NFS Server private IP address = 172.31.25.211

```bash
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.25.211:/mnt/apps /var/www
```

__4.__ __Confirm the NFS mount with `df -h`, then add it to `/etc/fstab` so it is restored after a reboot.__

![NFS application mount](<./images/Screenshot 2026-09-11 154128.png>)


```bash
sudo vi /etc/fstab
```

Add the following line
```bash
172.31.25.211:/mnt/apps /var/www nfs defaults 0 0
```
![Persistent mount configuration](<./images/Screenshot 2026-09-11 154357.png>)


__5.__ __Install Apache, the Remi repository, and the required PHP components__

```bash
sudo yum install httpd -y
```
![Apache installation](<./images/Screenshot 2026-09-11 154457.png>)

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
```
```bash
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-10.rpm
```
![EPEL installation](<./images/Screenshot 2026-09-11 154650.png>)

```bash
sudo dnf module reset php
```
![PHP module reset](<./images/Screenshot 2026-09-11 154903.png>)

```bash
sudo dnf module enable php:remi-8.5
```
![PHP 8.5 module](<./images/Screenshot 2026-09-11 154903.png>)

```bash
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
```
![PHP installation](<Screenshot 2026-09-11 155043.png>)

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl status php-fpm

sudo setsebool -P httpd_execmem 1  # Allows the Apache HTTP server (httpd) to execute memory that it can also write to. This is often needed for certain types of dynamic content and applications that may need to generate and execute code at runtime.
sudo setsebool -P httpd_can_network_connect=1   # Allows the Apache HTTP server to make network connections to other servers.
sudo setsebool -P httpd_can_network_connect_db=1  # allows the Apache HTTP server to connect to remote database servers.
```
![PHP-FPM service](<./images/Screenshot 2026-09-11 155251.png>)


### Web Server 2 Configuration

__1.__ __Launch another new EC2 instance with RHEL Operating System__

__2.__ __Install the NFS client packages__

```bash
sudo yum install nfs-utils nfs4-acl-tools -y
```
![NFS installation](<./images/Screenshot 2026-09-11 153911.png>)

__3.__ __Mount the NFS ```apps``` export at ```/var/www```__.
NFS Server private IP address = 172.31.25.211

```bash
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.25.211:/mnt/apps /var/www
```

__4.__ __Confirm the NFS mount with `df -h`, then add it to `/etc/fstab` so it is restored after a reboot.__

![NFS application mount](<./images/ssss.png>)


```bash
sudo vi /etc/fstab
```

Add the following line
```bash
172.31.25.211:/mnt/apps /var/www nfs defaults 0 0
```
![Persistent mount configuration](<./images/Screenshot 2026-09-11 154354.png>)

__5.__ __Install Apache, the Remi repository, and the required PHP components__

```bash
sudo yum install httpd -y
```
![Apache installation](<./images/Screenshot 2026-09-11 154525.png>)

```bash
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
```
```bash
sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-10.rpm
```
![Remi repository installation](<./images/Screenshot 2026-09-11 154710.png>)

```bash
sudo dnf module reset php
```
![PHP module reset](<./images/Screenshot 2026-09-11 154919.png>)

```bash
sudo dnf module enable php:remi-8.5
```
![PHP 8.5 module](<./images/PHP 8.5 module.png>)

```bash
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd
```
![PHP installation](<./images/Screenshot 2026-09-11 155051.png>)

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl status php-fpm
sudo setsebool -P httpd_execmem 1
```
![PHP-FPM service](<./images/Screenshot 2026-09-11 155348.png>)


__6.__ __Confirm that the application files are visible under ```/var/www``` on the web servers and under ```/mnt/apps``` on the NFS server. Matching files confirm that the NFS mount is working correctly.__
A ```index.hmtl``` file was created on Web Server 1 and then verified from Web Servers 2.

![create file](<./images/Screenshot 2026-09-11 161659.png>)
![create file](<./images/Screenshot 2026-09-11 161616.png>)
![check test file](<./images/Screenshot 2026-09-11 161810.png>)
![check test file-wb1](<./images/Screenshot 2026-09-11 163557.png>) ```WEB1```
![check test file-wb-2](<./images/Screenshot 2026-09-11 163634.png>)```WEB2```
__7.__ __Locate the log folder for Apache on the Web Servers and mount it to NFS server's export for logs. Repeat ```step 4``` to ensure the mount point persists after reboot__.

```bash
sudo vi /etc/fstab
```

Add the following line
```bash
172.31.25.211:/mnt/logs /var/log/httpd nfs defaults 0 0
```
![mount and persist logs](<./images/Screenshot 2026-09-11 162101.png>)


![Application deployment](<./images/Screenshot 2026-09-11 163449.png>)
![Application deployment](<./images/Screenshot 2026-09-11 164222.png>)
__Note__:
Acces the website on a browser

- Ensure TCP port 80 is open on the Web Server.




# Step 1 - Configure Apache as the Load Balancer

## 1. Provision the Ubuntu 24.04 Load Balancer Instance

![ec2 lb](<./images/Screenshot 2026-09-11 164812.png>)

## 2. Allow HTTP Traffic to the Load Balancer

![Port 80](<./images/Screenshot 2026-09-11 164654.png>)

## 3. Install Apache and Configure Traffic Forwarding

### A. Install Apache2

- Access the instance

```bash
ssh -i "ec2key.pem" ubuntu@100.56.229.233
```

- Update and upgrade Ubuntu

```bash
sudo apt update && sudo apt upgrade
```
![update ubuntu](<./images/Screenshot 2026-09-11 165004.png>)

- Install Apache

```bash
sudo apt install apache2 -y
```
![Apache](<./images/Screenshot 2026-09-11 165917.png>)

```bash
sudo apt-get install libxml2-dev
```
![Apache dependencies](<./images/Screenshot 2026-09-11 170015.png>)

### B. Enable the Required Apache Modules

```bash
sudo a2enmod rewrite

sudo a2enmod  proxy

sudo a2enmod  proxy_balancer

sudo a2enmod  proxy_http

sudo a2enmod  headers

sudo a2enmod  lbmethod_bytraffic
```
![modules](<./images/Screenshot 2026-09-11 170051.png>)

### C. Restart and Verify Apache2

```bash
sudo systemctl restart apache2
sudo systemctl status apache2
```
![Restart apache](<./images/Screenshot 2026-09-11 170118.png>)

## Configure the Backend Load-Balancing Pool

### A. Open the Default Virtual Host Configuration

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```
### B. Add the Load-Balancer Configuration Inside the Virtual Host

```apache
<Proxy balancer://mycluster>
            BalancerMember http://172.31.29.123:80 loadfactor=5 timeout=1
           BalancerMember http://172.31.23.7:80 loadfactor=5 timeout=1
           ProxySet lbmethod=bytraffic
           # ProxySet lbmethod=byrequests
</Proxy>


ProxyPreserveHost on
ProxyPass / balancer://mycluster/
ProxyPassReverse / balancer://mycluster/
```
![Server config](<./images/Screenshot 2026-09-11 173630.png>)

### C. Apply the Configuration

```bash
sudo systemctl restart apache2
```
![Restart apache](<./images/Screenshot 2026-09-11 172138.png>)

The bytraffic balancing method distributes incoming load between web servers according to the volume of network traffic they process. The proportion in which traffic is distributed is controlled by the loadfactor parameter.

Other methods such as ```bybusyness```, ```byrequests```, ```heartbeat``` can also be adopted.


## 4. Test the Load Balancer

### A. Open the Application Through the Load Balancer

![lb public ip](<./images/Screenshot 2026-09-11 172325.png>)
![lb-website](<./images/lb-wesite.png>)

__Note__: If in the previous project, ```/var/log/httpd``` was mounted from the Web Server to the NFS Server, unmount them and ensure that each Web Servers has its own log directory.

### B. Remove the Shared Apache Log Mount

- Check if the Web Server's log directory is mounted to NSF

```bash
df -h
sudo umount -f /var/log/httpd
```
If the directory is busy, the services using it needs to be stopped first.
```bash
sudo systemctl stop httpd
```

- Check that the directory is unmounted
```bash
df -h
```

### C. Monitor Both Web Server Access Logs

```bash
sudo tail -f /var/log/httpd/access_log
```
### D. Confirm That Requests Reach Both Web Servers

Refresh the application several times while monitoring the access logs on both web servers. New HTTP GET entries should appear on both servers. Since both backend members use the same ```loadfactor```, the traffic should be distributed at approximately the same rate.

Web Server 1 ```access_log```
![logs](<./images/Screenshot 2026-09-11 173105.png>)

Web Server 2 ```access_log```
![logs](<./images/Screenshot 2026-09-11 173116.png>)


# Optional Step - Configure Local Name Resolution

Working with several private IP addresses can become inconvenient as the number of servers grows. For this lab, local name resolution can be configured with the ```/etc/hosts``` file. This approach is simple for a small environment and demonstrates the concept clearly, although it is not very scalable.

## Create Local Name Mappings for the Load Balancer

### A. Edit the Hosts File

```bash
sudo vi /etc/hosts
```

### B. Map the Web Server IP Addresses to Local Names

![dns host](<./images/Screenshot 2026-09-11 173410.png>)

### C. Use the Local Names in Apache

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```
```bash
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
```
![dns name](<./images/Screenshot 2026-09-11 173630.png>)


### D. Test the Backend Names Locally

```bash
curl http://Web1
```
![curl web1](<./images/Screenshot 2026-09-11 173658.png>)

```bash
curl http://Web2
```
![curl web2](<./images/curl-web2.png>)


These names are defined only on the load balancer. Other servers and Internet clients will not resolve them unless a separate DNS service is configured.


### Summary

Apache's ```mod_proxy_balancer``` provides several mechanisms for distributing application traffic, including different balancing algorithms and options such as sticky sessions and health checks. With the backend servers correctly configured, the load balancer provides a central entry point while allowing requests to be handled across multiple web server.

 __SECURITY NOTE__

* The Core Risk: Combining chmod 777, broad NFS network exposure, and no_root_squash breaks defense-in-depth boundaries. Together, they create a high-severity vulnerability that allows a remote attacker to gain full host root control.
* Vulnerability Breakdown:
* chmod 777: Grants read/write/execute rights to all local users. Any compromised low-privilege service can alter files or run malicious code.
   * Broad NFS Access (e.g., *): Exports shares to wide subnets. Anyone on the network can mount the share without authentication.
   * no_root_squash: Trusts the client's root identity (UID 0) instead of downgrading it to nobody. Anyone who is root on a client machine becomes root on the shared files.
* The Attack Chain (Host Takeover):
1. Mount: Attacker mounts the broadly exposed NFS share from their own machine.
   2. Inject: Being root on their own machine (no_root_squash), they write a malicious binary to the share and set the SUID bit.
   3. Escalate: Because permissions are open (777), any local low-privilege account on the target server can execute that binary, instantly dropping the attacker into a host root shell.
* RIGHT APLLICATIONS:
* Restrict Files: Replace 777 with 755 for directories and 644 for files. Use chown for explicit ownership.
   * Lock Network: Limit /etc/exports strictly to explicit, trusted target IPs.
   * Enforce Squash: Keep root_squash enabled (system default) to strip remote administrative rights.



