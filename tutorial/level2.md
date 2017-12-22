Using a shared file system is probably OK for web applications with reasonably limited traffic, but will be be problematic when the traffic to your site increases. At that point you can scale out your front end to have as many web servers as you want, but your web application is limited by the capability of the shared file system running on a single server. In this level, we will resovle this issue by moving the shared storage from one web server to S3. This way, the web servers only handle critical business, while the images are being served by S3.

We first terminate the web server running NFS client. Then we edit /etc/exports on the web server running NFS server to disable (comment out) the NFS exports and stop nfs-kernel-server service. We no longer need a share file system in the level.

```
$ sudo service nfs-kernel-server stop
```

Next, you will need to edit config.php and make some minor changes. In this configuration file, $s3_bucket is the name of the S3 bucket for share storage, and $s3_baseurl is the URL pointing to the S3 endpoint in the region hosting your S3 bucket. You can find the S3 endpoint from `http://docs.aws.amazon.com/general/latest/gr/rande.html`. you can also identify this end point in the S3 Console by viewing the properties of an S3 object in the S3 bucket.

```
$storage_option = "s3";
$s3_bucket  = "my_uploads_bucket";
$s3_baseurl = "https://s3-ap-southeast-2.amazonaws.com/";
```

Next we create an AMI from the running instance. Then we create an EC2 Role in the IAM Console, which (IAM Console -> Role -> Create New Role -> Amazon EC2 -> Amazon S3 Full Access). Then we launch a new web server using the AMI and the IAM Role we just created. After the instance becomes “running” and passes health checks, remove the original web server from the ELB (you can terminate it now) and add this new web server to the ELB. In your browser, again browse to `http://<dns-name-of-elb>/web-demo/index.php`. Currently the only web server behind the ELB is this newly launched instance, but it still has your login information. Newly uploaded images will go to S3 instead of local disk. As the workload of your application increases, you can launch more EC2 instance using the AMI and IAM Role we created just now, and add these new instance to the ELB. When the workload of your application decreases, you can remove some instance from the ELB and terminate them to save cost.

The reason we use IAM Role in this tutorial is that with IAM Role you do not need to supply your AWS Access Key and Secret Key in your code. Rather, Your code will assume the role assigned to the EC2 instance, and access the AWS resources that your EC2 instance is allowed to access. Today many people and organizations host their source code on github.com or some other public repositories. By using IAM roles you no longer hard code your AWS credentials in your application, thus eliminating the possibility of leaking your AWS credentials to the public.