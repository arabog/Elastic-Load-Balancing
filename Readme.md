# Elastic Load Balancing
## A. Start the Load Balancer service
On the EC2 dashboard, select the Load Balancers service from the 
navigation pane on left.  

Launch the Create Load Balancer wizard. AWS offers four types of 
load balancers:   
Application load balancer  
Network load balancer  
Gateway load balancer  
Classic Load BalancerInfo  

![lb](lb.png?raw=true "lb")

### Application Load Balancer (ALB)
A simple use case: Assume you are running a microservices-architecture based 
application. An Application Load Balancer allows you to host the different 
API endpoints of your application on different servers. The load balancer 
then redirects the incoming HTTP/HTTP traffic to the suitable server based 
on the rules you specify in the configuration.

### Network Load Balancer (NLB)
A Network Load Balancer helps to balance the load on each individual server. 
Having an NLB becomes essential when your application requires handling millions 
of requests per second securely while maintaining ultra-low latencies.

### Gateway load balancer
Choose a Gateway Load Balancer when you need to deploy and manage a fleet of 
third-party virtual appliances that support GENEVE. These appliances enable 
you to improve security, compliance, and policy controls.

### Classic Load BalancerInfo
Choose a Classic Load Balancer when you have an existing application running 
in the EC2-Classic network.

Let's create a network load balancer (NLB), and see the role of a NLB.

An NLB serves as the single point of contact for clients and automatically 
distributes the incoming traffic uniformly across multiple targets. The 
targets are the EC2 instances within the same or different AZs.

## Prerequisites for Network Load Balancer:
An AWS account  
A default VPC  
Subnets in each AZ in the default VPC. Also, notice that a common route 
table is attached to all subnets.  
A route table with a rule for internet facing communication. See that 
it requires an internet gateway  
Target groups  

![igw](igw.png?raw=true "igw")

Go to the EC2 dashboard. In order to use elastic load balancing, you will 
need to make sure that you've launched the EC2 instances that you plan to 
register with your load balancer. 

You must have more than one EC2 instance in the running state. In the 
example snapshot below, we have two instances, in two different 
availability zones (AZs).

![instances](instances.png?raw=true "instances")


## Step 1. Create the first EC2 instance
The steps below show how to create the first EC2 instance in a public subnet 
in any one Availability Zone, and install the Apache webserver on it. 
Auto-assign Public IP:	Enable  

Let's add advanced details.  
Under the Advanced Details → User data section, add the following configuration 
script to run automatically during launch.  

#!/bin/bash  
sudo yum update -y  
sudo amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2  
sudo yum install -y httpd mariadb-server  
sudo systemctl start httpd  
sudo systemctl enable httpd  
sudo chkconfig httpd on  

#Set file permissions for the Apache web server  
sudo groupadd www  
sudo usermod -a -G www ec2-user  
sudo chgrp -R www /var/www  
sudo chmod 2775 /var/www  
find /var/www -type d -exec sudo chmod 2775 {} +  
find /var/www -type f -exec sudo chmod 0664 {} +  

#Create a new PHP file at  /var/www/html/ path  
echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php  

The script above will install, configure, and launch the Apache 
webserver on the EC2 instance.  

Specify the user data to configure the EC2 instance  

Keep the storage as default, and use a tag as Name: Server1  

Go to VPC, click security group and Configure Security Group, you 
can create a new one, and ensure to have a firewall rule to allow 
incoming HTTP traffic on port 80.  

![sec-grp](sec-grp.png?raw=true "sec-grp")

## Step 2. Create the second EC2 instance, in a separate Availability Zone
Launch the second EC2 instance using the same steps above, except for 
the following changes at the Configure Instance Details step:

Select another public subnet in a different AZ, say us-east-2b  

#!/bin/bash  
sudo yum update -y  
sudo amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2  
sudo yum install -y httpd mariadb-server  
sudo systemctl start httpd  
sudo systemctl enable httpd  
sudo chkconfig httpd on  

#Set file permissions for the Apache web server  
sudo groupadd www  
sudo usermod -a -G www ec2-user  
sudo chgrp -R www /var/www  
sudo chmod 2775 /var/www  
find /var/www -type d -exec sudo chmod 2775 {} +  
find /var/www -type f -exec sudo chmod 0664 {} +  

#Create a new PHP file at  /var/www/html/ path  
echo "<?php echo "<h1>Welcome to server 2</h1>" ?>" > /var/www/html/phpinfo.php

Additionally, change the tag to Name: Server2.  

## Step 3. Verify the Apache webserver installation
Confirm that the newly created EC2 instances are in the running state 
and status check is showing2/2 checks passed.  
Verify that the Apache server is running successfully on both the EC2 
instances. Simply copy, and paste the public IPv4 address of each instance 
in a new browser window. If the Apache is configured successfully, you 
will see the Apache welcome page.

http://54.226.12.70  
http://52.202.155.208  

![test-page1](test-page1.png?raw=true "test-page1")

![test-page2](test-page2.png?raw=true "test-page2")

**Note:** We have opened the HTTP traffic on the default port, therefore the 
public IPv4 address should be prepended with http://, instead of https://.

View the content of the PHP page that you configured using the shell script.

http://54.226.12.70/phpinfo.php  
http://52.202.155.208/phpinfo.php  

![php1](php1.png?raw=true "php1")

## Step 4. Create an NLB
Select the Load Balancers service on the left-hand side menu of the EC2 dashboard, 
and click on the Create Load Balancer button.  

![lb-network-mapping](lb-network-mapping.png?raw=true "lb-network-mapping")


![target-grp](target-grp.png?raw=true "target-grp")  

At the Register Targets step, add the two EC2 instances created previously to the 
target group.

![register-targets](register-targets.png?raw=true "register-targets")

## Step 5. Test the NLB
You will be taken back to the Load Balancers dashboard. Copy the DNS name of the 
newly created NLB, and append the /phpinfo.php at the end of it. A sample DNS name 
appended with the file name looks like this:  
http://network-load-balancer-9ffad67548f8c722.elb.us-east-1.amazonaws.com/phpinfo.php  

Paste the copied DNS name to a new browser window and refresh the browser a few times, 
each after a few seconds. You will notice that sometimes the request is redirected to 
Server-A and other times, it is routed to Server-B.

## Step 6. Delete the resources
Delete the NLB by going to the Load Balancer dashboard.    
Stop and terminate the EC2 instances by going to the Instances dashboard.    
Delete the Target groups by going to the Target groups dashboard  
