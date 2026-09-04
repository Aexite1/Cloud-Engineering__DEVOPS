# Task - Implement a Client Server Architecture using MySQL Database Management System (DBMS)

## The following steps were used to complete the task:

### Step 1. Create and configure two Linux-based virtual servers (EC2 instances in AWS)

Two EC2 instances were created, one to be used as the MySQL server and
the other as the MySQL client.

-   `mysql server`
-   `mysql client`

**1.** Two EC2 instances with t2.micro type and Ubuntu 24.04 LTS (HVM)
were launched in the us-east-1 region through the AWS console.

![Lunch Instance](<./images/create-ec2.png>)

**mysql server**

![Lunch Instance](<./images/ec2-server-detail.png>)

**mysql client**

![Lunch Instance](<./images/ec2-client-detail.png>)

The inbound security group rule for both instances was set to allow SSH
on port 22, with the source set to anywhere.

![Lunch Instance](<./images/ec2-server-security-rule.png>) ![Lunch
Instance](<./images/ec2-client-security-rule.png>)

**2.** The SSH key named **my-ec2-key** was attached to the instances so
that they could be accessed through SSH.

## Step 2 - On `mysql server` Linux Server, install MySQL Server software

**1.** The permission of the private SSH key was changed first, after
which it was used to connect to the server.

``` bash
chmod 400 my-ec2-key.pem
```

``` bash
ssh -i "my-ec2-key.pem" ubuntu@3.87.82.40
```

The username is **ubuntu** and the public IP address is **3.87.82.40**.

![Connect to instance](<./images/ssh-server.png>)

**2.** **Update and upgrade Ubuntu**

The Ubuntu packages were updated and upgraded using:

``` bash
sudo apt update && sudo apt upgrade -y
```

![Update ubuntu](<./images/server-update-upgrade.png>)
![Update ubuntu](<./images/server-update-upgrade.png>)

**3.** **Install MySQL Server software**

MySQL Server was installed with the following command:

``` bash
sudo apt install mysql-server -y
```

![Install mysql](<./images/install-mysql-server.png>)

**4.** **Enable mysql server**

The MySQL service was enabled so that it can start automatically:

``` bash
sudo systemctl enable mysql
```

![Enable mysql](<./images/enable-mysql.png>)

## Step 3 - On `mysql client` Linux Server install MySQL Client software.

**1.** **Connect to the instance**

The client EC2 instance was accessed using SSH:

``` bash
ssh -i "my-ec2-key.pem" ubuntu@54.242.30.171
```

The username is **ubuntu** and the public IP address is
**54.242.30.171**.

![Connect to instance](<./images/ssh-client.png>)

**2.** **Update and upgrade Ubuntu**

The Ubuntu system on the client was updated and upgraded:

``` bash
sudo apt update && sudo apt upgrade -y
```

![Update ubuntu](<./images/client-update-upgrade.png>)

**3.** **Install MySQL Client software**

The MySQL client package was installed by running:

``` bash
sudo apt install mysql-client -y
```

![Install mysql](<./images/install-mysql-client.png>)

## Step 4 - Use `mysql server's` local IP address to connect from `mysql client`.

Since both EC2 instances are in the same local virtual network, they can
communicate using their local IP addresses. The local IP address of the
MySQL server was used from the client machine.

MySQL normally uses TCP port 3306, so port 3306 was added to the inbound
rules of the MySQL server's security group.

For security, access was restricted instead of allowing every IP
address. Only the local IP address of the MySQL client was allowed.

![server security rule](<./images/server-security-rule2.png>)

## Step 5 - Configure MySQL server to allow connections from remote hosts.

**Befor the configuration stated above, the following were
implemented:**

**1.** The MySQL security script was run on the **mysql server** using:

``` bash
sudo mysql_secure_installation
```

![secure mysql](<./images/secure-mysql.png>)

**2.** **Access MySQL shell**

The MySQL shell was opened with:

``` bash
sudo mysql
```

![mysql shell](<./images/mysql-shell.png>)

**3.** **On mysql server, create a user named `client` and a database
named `test_db`**.

The MySQL user, database and required privileges were created using the
following commands:

``` sql
CREATE USER 'client'@'%' IDENTIFIED WITH cachin_sha2_password BY 'User123$';

CREATE DATABASE test_db;

GRANT ALL ON test_db.* TO 'client'@'%' WITH GRANT OPTION;

FLUSH PRIVILEGES;
```

![DB](<./images/create-user-n-db.png>)

**4.** **Now, configure MySQL server to allow connections from remote
hosts**.

The MySQL configuration file was opened using:

``` bash
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

The following setting was located:

``` text
bind-address = 127.0.0.1
```

The address was changed from `127.0.0.1` to `0.0.0.0` so that the server
could accept connections from remote hosts.

![command](<./images/mysql-bind-command.png>)
![bind-address](<./images/mysql-bind-address.png>)

## Step 6 - From `mysql client` Linux Server, connect remotely to `mysql server` Database Engine without using SSH. The mysql utility must be used to perform this action.

The connection to the database server was made directly from the MySQL
client using the MySQL utility:

``` bash
sudo mysql -u client -h 172.31.18.141 -p
```

![Client to DB](<./images/access-db-server-from-client.png>)

## Step 7 - Check that the connection to the remote MySQL server was successful and can perform SQL queries.

The connection was tested by displaying the databases available on the
MySQL server:

``` sql
show databases;
```

![show db](<./images/show-db.png>)

**Create table, insert rows into table and select from the table**

A table was then created in the test database, two records were
inserted, and the records were displayed using a SELECT query.

``` sql
CREATE TABLE test_db.test_table (
  item_id INT AUTO_INCREMENT,
  content VARCHAR(255),
  PRIMARY KEY(item_id)
);

INSERT INTO test_db.test_table (content) VALUES ("My first choice football club is Chelsea");

INSERT INTO test_db.test_table (content) VALUES ("My second choice football club is R.Madrid");

SELECT * FROM test_db.test_table;
```

![Query](<./images/sql-query.png>)

At this point, the project was successfully completed. The result is a
working MySQL Client-Server setup.
