
In this level, we will build a basic version with all the components deployed on one single server. The web application is developed with PHP, using Apache as the web serer, with MySQL as the database to store user upload information. You do not need to write any code, because I have a set of demo code prepared for you. You just need to launch an EC2 instance, carry out some basic configurations, then deploy the demo code.

Login to your AWS Console and navigate to the EC2 Console. Launch an EC2 instance with a Ubuntu 14.04 AMI. Make sure that you allocate a public IP to your EC2 instance. In your security group settings, open port 80 for HTTP and port 22 for SSH access. After the instance becomes “running” and passes health checks, SSH into your EC2 instance to setup software dependencies and download the demo code from Github to the default web server folder:

```shell
$ sudo apt-get update
$ sudo apt-get install apache2 php5 mysql-server php5-mysql php5-curl git
$ cd /var
$ sudo chown -R ubuntu:ubuntu www
$ cd /var/www/html
$ git clone https://github.com/qyjohn/web-demo
```

Then we create a MySQL database and a MySQL user for our demo. Here we use "web_demo" as the database name, and "username" as the MySQL user.

```sql
$ mysql -u root -p
mysql> CREATE DATABASE web_demo;
mysql> CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON web_demo.* TO 'username'@'localhost';
mysql> quit
```

In the code you clone from Github, we have prepopulated some demo data as examples. We use the following command to import the demo data in web_demo.sql to the web_demo database:

```shell
$ cd /var/www/html/web-demo
$ mysql -u username -p web_demo < web_demo.sql
```

The LEVEL 0 demo code is implemented in a two PHP files index.php and config.php. Before we can make it work, there are some minor modifications needed:

(1) Use a text editor to open config.php, then change the username and password for your MySQL installation.
(2) Change the ownership of folder “uploads” to “www-data” so that Apache can upload files to this folder.

```
$cd /var/www/html/web-demo
$ sudo chown -R www-data:www-data uploads
```

In your browser, browse to `http://<ip-address>/web-demo/index.php`. You should see that our application is now working. You can login with your name, then upload some photos for testing. (You might have noticed that this demo application does not ask you for a password. This is because we would like to make things as simple as possible. Handling user password is a very complicate issue, which is beyond the scope of this entry level tutorial.) Then I suggest that you spend 10 minutes reading through the demo code index.php. The demo code has reasonable documentation in the form of comments, so I am not going to explain the code here.