
## WEB STACK IMPLEMENTATION (LEMP STACK) IN AWS

### Introduction

The **LEMP stack** is a widely used open-source solution for building and hosting web applications. It combines four powerful technologies: **Linux**, **Nginx**, **MySQL**, and **PHP**. Together, these components provide everything needed to serve web pages, manage data, and process dynamic content efficiently. This guide documents the process of deploying and configuring a LEMP environment on an AWS EC2 instance.

---

## Step 0: Prerequisites

**1.** An **EC2 instance** running **Ubuntu 24.04 LTS (HVM)** was launched on AWS using the **t3.micro** instance type in the **eu-north-1b** region.

![Lunch Instance](<./images/Screenshot 2026-06-20 090647.png>)

**2.** An SSH key pair named **lemp-ec2-key** was generated to allow secure access to the server through prt 22.

**3.** The instance's security group was configured to allow the following inbound traffic:

* HTTP traffic on port 80 from any source.
* HTTPS traffic on port 443 from any source.
* SSH traffic on port 22 for remote administration.

![Security Rules](<./images/Screenshot (76).png>)

**4.** The default AWS VPC and subnet settings were used during instance configuration.

![Default Network](<./images/Screenshot 2026-06-20 090935.png>)

**5.** After locating the downloaded private key file, its permissions were adjusted before using it to establish an SSH connection to the server.

```
chmod 400 my-ec2-key.pem
```

```
ssh -i "my-ec2-key.pem" ubuntu@51.20.41.222
```

In this case, the login user is **ubuntu**, while **51.20.41.222** is the public IP address assigned to the EC2 instance.

![Connect to instance](<./images/Screenshot 2026-06-20 091816.png>)

---

## Step 1 - Install the Nginx Web Server

**1.** **Refresh the package repository and upgrade installed packages**

Before installing new software, the package index was updated and existing packages were upgraded to their latest available versions.

```
sudo apt update
sudo apt upgrade -y
```

![Update Packages](<./images/Screenshot 2026-06-20 092131.png>)
![Upgrade Packages](<./images/Screenshot 2026-06-15 122123.png>)

**2.** **Install Nginx**

The Nginx web server package was installed using the APT package manager.

```
sudo apt install nginx -y
```

![Instal Nginx](<./images/Screenshot 2026-06-20 092655.png>)

**3.** **Confirm that Nginx is running correctly**

After installation, the service status was checked to ensure that Nginx started successfully.

```
sudo systemctl status nginx
```

If the service status appears as **active (running)**, the installation was successful.

![Nginx Status](<./images/Screenshot 2026-06-20 092844.png>)

**4.** **Verify access locally from the server**

A local request was sent to Nginx from the Ubuntu terminal to confirm that it was responding properly.

```
curl http://127.0.0.1:80
```

![Local URL](<./images/Screenshot 2026-06-20 093239.png>)

**5.** **Verify access from a web browser using the public IP address**

The server was then tested externally by entering its public IP address into a browser.

```
http://18.209.18.61:80
```

![Nginx Default Page](<./images/Screenshot 2026-06-20 093436.png>)

The successful display of the Nginx welcome page confirms that the web server is operational and reachable through the configured firewall rules.

**6.** **Retrieve the instance public IP address using the metadata service**

Instead of checking the AWS Console, the public IP address can also be obtained directly from the instance metadata endpoint.

```
curl -s http://169.254.169.254/latest/meta-data/public-ipv4
```

Initially, the request returned a **401 Unauthorized** error.

![Unauthorized Error-401](./images/curl-401-unauthorized.png)

To resolve this issue, the following AWS configuration was updated:

* Actions → Instance Settings → Modify Instance Metadata Options
* Change **IMDSv2** from **Required** to **Optional**

![imds option](<./images/Screenshot 2026-06-20 094240.png>)

After applying the change, the same command was executed again and successfully returned the instance's public IP address.

```
curl -s http://169.254.169.254/latest/meta-data/public-ipv4
```

![Public IP with curl](<./images/Screenshot 2026-06-20 094325.png>)

---

## Step 2 - Install MySQL

**1.** **Install MySQL Server**

To manage application data, MySQL was selected as the relational database management system for this deployment.

```
sudo apt install mysql-server
```

![Install MySQL](<./images/Screenshot 2026-06-20 094613.png>)

**2.** **Access the MySQL Shell**

Once installed, the MySQL console was opened using administrative privileges.

```
sudo mysql
```

Because the command was executed with **sudo**, access was granted as the MySQL **root** user.

![MySQL console](<./images/Screenshot 2026-06-20 094731.png>)

**3.** **Configure a Password for the Root User**

The root account was configured to use the **caching_sha2_password** authentication plugin.

```
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'Admin123$';
```

![Root password](<./images/Screenshot 2026-06-20 095038.png>)

Exit the MySQL shell:

```
exit
```

**4.** **Run MySQL Security Hardening Script**

MySQL provides a built-in security script that removes insecure defaults and strengthens database protection.

```
sudo mysql_secure_installation
```

![Change root password](<./images/Screenshot 2026-06-20 095823.png>)

**5.** **Verify Login Using the Newly Configured Password**

After setting the password, the MySQL shell was accessed again using password authentication.

```
sudo mysql -p
```

A password prompt appeared before access was granted.

![MySQL login with password](<./images/Screenshot 2026-06-20 100451.png>)

Exit MySQL:

```
exit
```

---

## Step 3 - Install PHP

**1.** **Install PHP and Required Extensions**

Since Nginx does not process PHP code directly, PHP-FPM was installed to handle PHP execution. Additionally, the MySQL PHP extension was installed to allow database connectivity.

Installed packages:

* php-fpm
* php-mysql

```
sudo apt install php-fpm php-mysql -y
```

![Install PHP](<./images/Screenshot 2026-06-20 100555.png>)

---

## Step 4 - Configure Nginx to Process PHP Requests

**1.** **Create a Document Root for the Website**

A dedicated directory was created to store website files.

```
sudo mkdir /var/www/projectLEMP
```


**2.** **Assign Ownership to the Current User**

Ownership of the directory was updated so that the current user could manage website files.

```
sudo chown -R $USER:$USER /var/www/projectLEMP
```

![Change owner](<./images/Screenshot 2026-06-20 114404.png>)

**3.** **Create a New Nginx Server Block Configuration**

A new site configuration file was created inside Nginx's available sites directory.

```
sudo nano /etc/nginx/sites-available/projectLEMP
```

Paste in the following bare-bones configuration:

```nginx
server {
  listen 80;
  server_name projectLEMP www.projectLEMP;
  root /var/www/projectLEMP;

  index index.html index.htm index.php;

  location / {
    try_files $uri $uri/ =404;
  }

  location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
  }

  location ~ /\.ht {
    deny all;
  }
}
```

![Nginx config](<./images/Screenshot 2026-06-20 120548.png>)

### Understanding the Configuration

* **listen** specifies the port Nginx should monitor for incoming requests.
* **root** points to the directory containing website files.
* **index** defines the order in which default pages should be loaded.
* **server_name** determines which domain names or IP addresses this server block responds to.
* **location /** handles standard requests and checks whether the requested file exists.
* **location ~ .php$** forwards PHP requests to PHP-FPM for processing.
* **location ~ /.ht** prevents access to hidden `.htaccess` files.

**4.** **Enable the Site Configuration**

A symbolic link was created so Nginx could use the new configuration.

```
sudo ln -s /etc/nginx/sites-available/projectLEMP /etc/nginx/sites-enabled/
```


**5.** **Validate the Configuration**

Before applying changes, the configuration syntax was tested.

```
sudo nginx -t
```

![Test syntax](<./images/Screenshot 2026-06-20 135659.png>)

**6.** **Disable the Default Nginx Site**

The default site was removed to prevent conflicts with the new configuration.

```
sudo unlink /etc/nginx/sites-enabled/default
```

![Disable nginx default](<./images/Screenshot 2026-06-20 135832.png>)

**7.** **Reload Nginx**

Reloading the service applies the updated configuration without restarting the server.

```
sudo systemctl reload nginx
```

![Reload nginx](<./images/Screenshot 2026-06-20 135929.png>)

**8.** **Create a Test Landing Page**

Since the document root was empty, an index page was created for testing.


![Site content](<./images/Screenshot 2026-06-20 135328.png>)

#### Access Using Public IP

```
http://18.209.18.61:80
```

![Site with ip-address](<./images/Screenshot 2026-06-20 140006.png>)

#### Access Using Public DNS

```
http://<public-DNS-name>:80
```

![Site with dns name](<./images/Screenshot 2026-06-20 140039.png>)

At this stage, the Nginx server was successfully serving content from the custom web root. The temporary HTML page can remain in place until a PHP application is deployed.

The LEMP environment is now fully configured and operational.

---

## Step 5 - Validate PHP Processing

The next step is to verify that Nginx can successfully pass PHP requests to PHP-FPM.

**1.** **Create a PHP Test File**

```
sudo nano /var/www/projectLEMP/info.php
```

Paste in:

```php
<?php
phpinfo();
```

**2.** **Open the PHP Page in a Browser**

```
http://18.209.18.61/info.php
```

![PHP page](<./images/Screenshot 2026-06-20 144203.png>)

The displayed page provides detailed information about the installed PHP environment and server configuration.

For security reasons, the file should be deleted after testing because it exposes sensitive server information.

```
sudo rm /var/www/projectLEMP/info.php
```

---

## Step 6 - Access MySQL Data Through PHP

To allow PHP applications to communicate with MySQL, a dedicated database and user account were created.

**1.** **Log in to MySQL as Root**

```
sudo mysql -p
```


**2.** **Create a Database**

```
CREATE DATABASE todo_dbs;
```

![Create database](<./images/Screenshot 2026-06-20 215911.png>)

**3.** **Create a User and Grant Permissions**

```
CREATE USER 'todo_ownr'@'%' IDENTIFIED WITH mysql_native_password BY '!Aexite';

GRANT ALL ON todo_database.* TO 'todo_user'@'%';
```

![Create user](<./images/Screenshot 2026-06-20 215919.png>)

![Create user](<./images/Screenshot 2026-06-20 220609.png>)
```
exit
```

**4.** **Verify Database Access**

```
mysql -u todo_user -p

SHOW DATABASES;
```

![Show database](<./images/Screenshot 2026-06-25 153303.png>)

**5.** **Create a Table**

```
CREATE TABLE todo_database.todo_list (
  item_id INT AUTO_INCREMENT,
  content VARCHAR(255),
  PRIMARY KEY(item_id)
);
```
![Create table](<./images/Screenshot 2026-06-25 153314.png>)

**6.** **Insert Sample Records**


![Insert rows](<./images/Screenshot 2026-06-25 160520.png>)

**7.** **Verify the Records**

```
SELECT * FROM todo_database.todo_list;
```

![Query table](<./images/Screenshot 2026-06-25 160801.png>)

```
exit
```

### Create a PHP Application to Read Database Records

**1.** **Create a New PHP File**

```
sudo nano /var/www/projectLEMP/todo_list.php
```

The script below connects to MySQL, retrieves entries from the table, and displays them in an ordered list.


![PHP Script](<./images/Screenshot 2026-06-25 165329.png>)

**2.** **Test the Script in a Browser**

```
http://18.209.18.61/todo_list.php
```


A **502 Bad Gateway** error could appear when attempting to access the page.

The issue is caused by the Nginx configuration referenced **PHP 8.1**, while the installed PHP version was **8.3**. Since Nginx could not locate the correct PHP-FPM socket, requests failed.

To fix the issue, the Nginx server block was updated to point to the correct PHP version.


After saving the changes and reloading Nginx, the page loaded successfully.

```
http://18.209.18.61/todo_list.php
```

![Updated site with dns](<./images/Screenshot 2026-06-25 165410.png>)

The same page was also accessible through the server's DNS name.

![Updated site with dns](<./images/Screenshot 2026-06-25 165410.png>)

---

## Conclusion

The LEMP stack offers a dependable and efficient environment for hosting modern web applications. By combining Linux, Nginx, MySQL, and PHP, it becomes possible to deploy applications capable of handling dynamic content, database interactions, and web traffic reliably. This implementation demonstrates how these technologies can be integrated on AWS to create a functional and scalable web hosting platform.
