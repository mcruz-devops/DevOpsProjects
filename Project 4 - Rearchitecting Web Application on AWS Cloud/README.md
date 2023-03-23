# Project 4 - Rearchitecting Web Application on AWS Cloud 

## Prerequisites:
• AWS Account <br>
• Cloudfront distribution <br>
• Maven <br>
• JDK8 <br>
• Route 53 domain name <br>
• ACM certificate

## Architecture <br>
![image](https://user-images.githubusercontent.com/31238382/227367889-57829851-255d-473b-90ad-4c0085d2d2d3.png)

# Tasks
## Create a key pair and security group for the backend services
* On the EC2 console, search for key pairs and create one
![image](https://user-images.githubusercontent.com/31238382/227368275-bd5d3a75-d1b6-4bc9-8330-267456f0d8e1.png)

* Create a security group
![image](https://user-images.githubusercontent.com/31238382/227368344-80044c24-e839-47aa-813e-6603148b3f87.png)

## Create an RDS instance
* Firstly, create a subnet group. Select all availability zones and all subnets 
![image](https://user-images.githubusercontent.com/31238382/227368576-7a795b9d-68d5-4fef-a6ac-3b6ddd2bf280.png)

* Create a parameter group
```
Parameter group family: mysql5.7
Type: DB Parameter Group
Group Name: vprofile-rds-para-group
```
![image](https://user-images.githubusercontent.com/31238382/227368925-441e65fb-72ba-4eab-98fb-226829ac5ac7.png)

* Create the RDS instance with these properties
```
Method: Standard Create
Engine Options: MySQL
Engine version: 5.7.33
Templates: Free-Tier
DB Instance Identifier: vprofile-rds-mysql
Master username: admin
Password: Auto generate psw
Instance Type: db.t2.micro
Subnet grp: vprofile-rds-sub-grp
SecGrp:  vprofile-backend-SG
No public access
DB Authentication: Password authentication
Additional Configuration
Initial DB Name: accounts
Keep the rest default or you may add as your own preference
```
![image](https://user-images.githubusercontent.com/31238382/227369968-dd902764-0415-4b2d-af0d-26f7ab0a3eef.png)

## Create Elasticache
* Create a parameter group with the details below
```
Name: vprofile-memcached-para-group
Description: vprofile-memcached-para-group
Family: memcached1.4
```
![image](https://user-images.githubusercontent.com/31238382/227370343-6b4f75a8-8f8b-4917-8410-19c9be3371f7.png)

* Create a subnet group
```
Name: vprofile-memcached-sub-grp
AZ: Select All
Subnet: Select All
```
![image](https://user-images.githubusercontent.com/31238382/227370463-17ba2bc9-1208-4c25-b947-06717bbd8e50.png)

* Create the Memcached cluster
```
Name: vprofile-elasticache-svc
Engine version: 1.4.5
Parameter Grp: vprofile-memcached-para-group
NodeType: cache.t2.micro
# of Nodes: 1
SecGrp: vprofile-backend-SG
```
![image](https://user-images.githubusercontent.com/31238382/227370593-d5137ff3-650e-4c40-ad46-a6729ca469b5.png)

## Create RabbitMQ service
* Search for Amazon MQ, select Rabbit MQ and create it with these properties
```
Engine type: RabbitMQ
Single-instance-broker
Broker name: vprofile-rmq
Instance type: mq.t3.micro
username: rabbit
password: Blue7890bunny
Additional Settings:
private Access
VPC: use default
SEcGrp: vprofile-backend-SG
```
![image](https://user-images.githubusercontent.com/31238382/227370903-0a13af6d-98e6-4a13-9185-db59c044141c.png)

## Database Initialization
* Go to the RDS instance, copy the endpoint. In my case, it was: <br>
``` vprofile-rds-mysql.clacmsizwwve.us-east-1.rds.amazonaws.com ```

* Create an EC2 instance to initialize the DB, this instance will be terminated after initialization.
```
Name: mysql-client
OS: ubuntu 18.04
t2.micro
SecGrp: Allow SSH on port 22
Keypair: vprofile-beanstalk
Userdata:
#! /bin/bash
apt update -y
apt upgrade -y
apt install mysql-client -y
```

* SSH into mysl-client instance. We can check mysql version <br>
``` mysql -V ```

* Update the ```vprofile-backend-SG``` security group to allow inbound access on port 3306 then run the command below <br>
``` 
mysql -h vprofile-rds-mysql.clacmsizwwve.us-east-1.rds.amazonaws.com -u admin -p<db_password>
mysql> show databases;
exit
```
![image](https://user-images.githubusercontent.com/31238382/227372316-b2f375a0-8446-4c80-8209-f8e16e903ce3.png)

* Clone the source code into the EC2 instance to initialize the database
```
git clone https://github.com/rumeysakdogan/vprofileproject-all.git
cd vprofileproject-all
git checkout aws-Refactor
cd src/main/resources
mysql -h vprofile-rds-mysql.chrgxmhxkprk.us-east-1.rds.amazonaws.com -u admin -padvPtIYOfqGe4T41MUXk accounts < db_backup.sql
mysql -h vprofile-rds-mysql.chrgxmhxkprk.us-east-1.rds.amazonaws.com -u admin -padvPtIYOfqGe4T41MUXk accounts
show tables;
```

## Create a Beanstalk Environment
* Copy the endpoints for the backend services
```
RDS - vprofile-rds-mysql.clacmsizwwve.us-east-1.rds.amazonaws.com

RabbitMQ - b-a9a1367d-6326-4712-84e5-9aff3999dc1b.mq.us-east-1.amazonaws.com:5671

Memcached - vprofile-elasticache-svc.ohsezc.cfg.use1.cache.amazonaws.com:11211
```

* On the Beanstalk console, create a new environment with these details. NB: this process will take a couple of minutes.
```
Name: vprofilejavaapp-prod-rd
Platform: Tomcat
Keep the rest default
Configure more options:
- Custom configuration
****Instances****
EC2 SecGrp: vprofile-backend-SG
****Capacity****
LoadBalanced
Min:2
Max:4
InstanceType: t2.micro
****Rolling updates and deployments****
Deployment policy: Rolling
Percentage :50 %
****Security****
EC2 key pair: vprofile-beanstalk
```

## Update Security Group & Elastic Beanstalk
* On the ```vprofile-backend-SG```, add these inbound rules
```
Custom TCP 3306 from Beanstalk SecGrp(you can find id from EC2 insatnces)
Custom TCP 11211 from Beanstalk SecGrp
Custom TCP 5671 from Beanstalk SecGrp
```

* On the Elastic Beanstalk console, on the application environment, we need to clink Configuration apply these chnages <br>
```
Add Listener HTTPS port 443 with SSL cert
Processes: Health check path : /login
```

## Build and Deploy Artifact
* On the terminal, cd into the directory containing the cloned repository and change to the aws-Refactor branch
```
git checkout aws-Refactor
```

* Open the application properties file in the scr/main/resources folder and make changes to the below using endpoints and passwords you copied down in the previous sections
```
vim src/main/resources/application.properties
*****Chnages to be made*****
jdbc.url
jdbc.password
memcached.active.host
rabbitmq.address
rabbitmq.username
rabbitmq.password
```

* Switch to the root directory containing the pom.xml file and do a maven install <br>
```
mvn install
``` 

## Upload Artifact to Elastic Beanstalk
* Select the correct application and upload the artifact
* Select the uploaded application and click deploy
![image](https://user-images.githubusercontent.com/31238382/227375162-5e4c6fa2-4ccb-4eef-a907-dce82f79b365.png)

* Check if the deployment was successful
![image](https://user-images.githubusercontent.com/31238382/227375296-313a2561-d678-4279-a075-0ddd7f197cb3.png)
![image](https://user-images.githubusercontent.com/31238382/227375328-75a4cd5c-c40e-4e60-8724-0eea8c39f06c.png)

### Create a DNS record in Route53
![image](https://user-images.githubusercontent.com/31238382/227375427-3579c1a6-5624-42e0-8360-789f5cb3d27d.png)

## Create CloudFront distribution
```
Origin Domain: DNS record name we created for our app in previous step
Viewer protocol: Redirect HTTP to HTTPS
Alternate domain name: DNS record name we created for our app in previous step
SSL Certificate: 
Security policy: TLSv1
```
![image](https://user-images.githubusercontent.com/31238382/227375681-8c12721b-5e46-41cd-aa44-f24d54c4798a.png)

## Clean up
* Delete all AWS resources
