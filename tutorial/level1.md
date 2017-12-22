In this level, we will expand the basic version we have in LEVEL 0 and deploy it to multiple servers. Currently we already have a working web server, so we launched another EC2 instance and follow the steps in LEVEL 0 to get a second web server (you can skip mysql-server and the part to set up the database, because we don’t need multiple MySQL servers). Also, we know that these two web servers must write upload data to the same database server, so we launch an RDS instance and use the RDS instance as a shared database server. In front of the two web servers, we create an ELB to distribute the workload to two web servers.

(1) Launch a second web server as describe in LEVEL 0.
(2) Launch an RDS instance running MySQL. When launching the RDS instance, create a default database named “web_demo”. When the RDS instance becomes available, use the following command to import the demo data in web_demo.sql to the web_demo database on the RDS database:

```shell
$ mysql -h [endpoint-of-rds-instance] -u username -p web_demo < web_demo.sql
```

(3) On both web servers, modify config.php with the new database server hostname, username, password, and database name.
(4) Create an ELB and add the two web servers to the ELB. Since we do have Apache running on both web servers, you might want to use HTTP as the ping protocol with 80 as the ping port and “/” as the ping path for the health check parameter for your ELB.
(5) In your browser, browser to `http://elb-endpoint/web-demo/index.php`. As you can see, our demo seems to be working on multiple servers. This is so easy!

But there are issues. Sometimes your browser asks you to login, but sometime not. Also, some newly uploaded images seem to be missing, but they are back from time to time and some other images seem to be missing!

You must have noticed that the IP address in the top left position in your browser changes from time to time. We use this trick to let you know which web server is processing your request. Although the two web servers are using the same set of code and the same RDS database, they do not share your session information. When you are being served by server A and login to server A, you can upload an image. But server B is not aware of the fact that you have already login on server B. So when you are being served by server B, it will ask you to login again. Also, when you upload an image to server A, it is only available on server A. If you are being served by server B, although the database has the information about that particular upload (because both web servers write to, and read from, the same database server) but server B can’t give you that image because it is on server A.

With ELB, you can configure session stickiness so that the ELB always routes traffic from the same session to the same web server. However, for a large scale applicaion with dynamic workload, web servers are being added to, or removed from, the fleet according to workload requirements. In a worse case, a particular web server might be removed from the fleet due to a fault in the web server itself. In such cases, existing sessions might be routed to other web servers, on which there exists no login information. Therefore, it is desirable that such session information can be shared among all web servers.

We user ElastiCache to resolve the issue about session sharing between multiple web servers. In the ElastiCache console, launch an ElasticCache with Memcache and obtain the endpoint information. On both web servers, install php5-memcached and configure php.ini to use memcached for session sharing.

```
$ sudo apt-get install php5-memcached
```

Then edit `/etc/php5/apache2/php.ini`, make the following modifications:

session.save_handler = memcached
session.save_path = "[endpoint-to-the-elasticache-instance]:11211"
Then you need to restart Apache on both web servers to make the new configuration effective.

```
$ sudo service apache2 restart
```

Now go back to your browser to do some testing. As you can see, now your session is being shared across the two web servers. You only need to login once, and your login status remains the same regardless of the back end web server.

To solve the issue of image upload, you need to have a shared storage between your two web servers. One simple solution would be using one of the web servers as NFS server, which exports the `/var/www/html/web-demo` folder to the subnet. The second web server will act as the NFS client, mounting the remote NFS share as /var/www/html/web-demo. As long as the permissions are properly setup, the two web servers should be able to write to, and read from the same folder.

On one of your web servers, use the following command to install NFS server:

```
$ sudo apt-get install nfs-kernel-server
```

Then edit `/etc/exports` to export the `/var/www/html/web-demo` folder (assume that 172.31.0.0/16 is the CIDR range of your subnet):

```
/var/www/html/web-demo       172.31.0.0/16(rw,fsid=0,insecure,no_subtree_check,async)
```

Then you need to restart the NFS server for the new export to take effect:

```
$ sudo service nfs-kernel-server restart
```

On the other of your web servers, use the following command to install NFS client (assume that 172.31.0.11 is the private IP address of your NFS server):

```shell
$ sudo apt-get install nfs-common
$ cd /var/www/html
$ sudo rm -Rf web-demo
$ sudo mkdir web-demo
$ sudo chown -R ubuntu:ubuntu web-demo
$ sudo mount 172.31.0.11:/var/www/html/web-demo web-demo
```

In your browser, again browse to `http://<dns-name-of-elb>/web-demo/index.php.` You should see that our application is now working on multiple web servers with a load balancer as the front end, without any code changes.