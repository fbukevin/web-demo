Now you have a scalable web application with two web servers, and you know that you can scale in and out the fleet when needed. Is it possible to scale your fleet automatically, according to the workload of your application?

In our demo code, we have a setting to simulate workloads. If you look at config.php, you will find this piece of code:

```
$latency = 0;
```

And, in index.php, there is a corresponding statement:

sleep($latency);
That is, when a user request index.php, PHP will sleep for $latency seconds. As we know, as the workload increases, the CPU utilization (as well as other factors) increases, resulting in an increase in latency. By manually manupulating the $latency setting, we can simulate heavy workload to your web application, which is reflected in the average latency. It should be noted that you can change the latency simulation settings on each web server. When you have two web servers and you set $latency = 0 on one server and $latency = 1 on the other server, the average latency will be 0.5 second.

With AWS, you can use AutoScaling to scale your server fleet in a dynamic fashion, according to the wordload of the fleet. In this tutorial, we use average latency as a trigger for scaling actions. You can achieve this following these steps:

(1) In your EC2 Console, create a launch configuration using the AMI and the IAM Role that we created in LEVEL 2.

(2) Create an AutoScaling group using the launch configuration we created in step (1), make sure that the AutoScaling group receives traffic from your ELB. Also, change the health check type from EC2 to ELB. (This way, when the ELB determines that an instance is unhealthy, the AutoScaling group will terminate it.) You don’t need to specify any scaling policy at this point.
(3) Click on your ELB and create a new CloudWatch Alarm (ELB -> Monitoring -> Create Alarm) when the average latency is greater than 1000 ms for at least 1 minutes.
(4) Click on your AutoScaling group, and create a new scaling policy (AutoScaling -> Scaling Policies), using the CloudWatch Alarm you just created. The auto scaling action can be “add 1 instance and then wait 300 seconds”. This way, if the average latency of your web application exceeds 1 second, AutoScaling will add one more instance to your fleet.

You can do the testing by adjusting the $latency value on your existing web servers. Please note that to acheve 1000 ms latency, the average value of the $latency settings on all your web servers needs to be greater than 1. So, you can keep $latency = 0 on one of your webserver, and change $latency = 3 on the other web server. This way the average latency will be 1500 ms, which will trigger the CloudWatch Alarm, and hency the auto scaling policy.

When you are done with this step, you can play with scaling down by creating another CloudWatch Alarm and a corresponding auto scaling policy. The CloudWatch alarm will be alarmed when the average latency is smaller than 500 ms for at least 1 minute, and the auto scaling action can be “remove 1 instance and then wait 300 seconds”.