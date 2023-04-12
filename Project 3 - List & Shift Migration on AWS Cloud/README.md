# Project 3 - Lift & Shift Migration on AWS Cloud

## Prerequisites:
• AWS Account <br>
• DNS Name <br>
• Maven <br>
• JDK8 <br>
• AWS CLI

## AWS Architecture
![image](https://user-images.githubusercontent.com/31238382/226939970-b31b76c2-508f-4d83-8f92-394471d76e0b.png)

## Create Security Groups on the EC2 console and open the necessary ports


## Create a key pair to connect to the EC2 instances via SSH
![image](https://user-images.githubusercontent.com/31238382/226941634-7a8ca291-0988-43c7-9ede-ed1d9d7e1fe1.png)

## Provision Backend instances with user-data script
* Create the DB instance using the user-data
```
#!/bin/bash
DATABASE_PASS='admin123'
sudo yum update -y
sudo yum install epel-release -y
sudo yum install git zip unzip -y
sudo yum install mariadb-server -y


# starting & enabling mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
cd /tmp/
git clone -b vp-rem https://github.com/devopshydclub/vprofile-repo.git
#restore the dump file for the application
sudo mysqladmin -u root password "$DATABASE_PASS"
sudo mysql -u root -p"$DATABASE_PASS" -e "UPDATE mysql.user SET Password=PASSWORD('$DATABASE_PASS') WHERE User='root'"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-repo/src/main/resources/db_backup.sql
sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"

# Restart mariadb-server
sudo systemctl restart mariadb


#starting the firewall and allowing the mariadb to access from port no. 3306
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
sudo systemctl restart mariadb

```
![image](https://user-images.githubusercontent.com/31238382/226942292-dc71388f-79d0-4bdc-a727-4b99313bcae1.png)

SSH into the instance and validate the status of MariaDB
```
chmod 400 vprofile-prod-key.cer
ssh -i vprofile-prod-key.cer centos@<public_ip_of_instance>
sudo -i
curl http://169.254.169.254/latest/user-data
systemctl status mariadb
```

![image](https://user-images.githubusercontent.com/31238382/226943306-04fbc9f9-3801-48d2-aee8-88174401a43f.png)

* Create the Memcached instance <br>
User data
```
#!/bin/bash
sudo yum install epel-release -y
sudo yum install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached
sudo memcached -p 11211 -U 11111 -u memcached -d
```

![image](https://user-images.githubusercontent.com/31238382/226943713-eea45a2d-ec5f-4960-b414-4469b060f85e.png)

SSH into the memcached instance
```
ssh -i vprofile-prod-key.cer centos@<public_ip_of_instance>
sudo -i
curl http://169.254.169.254/latest/user-data
systemctl status memcached.service
ss -tunpl | grep 11211
```
![image](https://user-images.githubusercontent.com/31238382/226944558-b6dce475-3e4f-47b0-8734-6e4c0a0b298b.png)
![image](https://user-images.githubusercontent.com/31238382/226944611-696a9b23-e060-4a87-9cd8-d1ab5898f730.png)

* Create the RabbitMQ instance <br>
User data
```
#!/bin/bash
sudo yum install epel-release -y
sudo yum update -y
sudo yum install wget -y
cd /tmp/
wget http://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm
sudo rpm -Uvh erlang-solutions-2.0-1.noarch.rpm
sudo yum -y install erlang socat
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash
sudo yum install rabbitmq-server -y
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server
sudo systemctl status rabbitmq-server
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
sudo systemctl restart rabbitmq-server
```

SSH into the RabbitMQ instance
```
ssh -i vprofile-prod-key.cer centos@<public_ip_of_instance>
sudo -i
curl http://169.254.169.254/latest/user-data
systemctl status rabbitmq-server
```

![image](https://user-images.githubusercontent.com/31238382/226945531-778ad2a6-676c-402b-b38c-df3db7bb7403.png)

## Create a Privated hosted zone on Route53
![image](https://user-images.githubusercontent.com/31238382/226945981-1bc24dd9-3293-49ee-ab48-535da2820b8c.png)

Create records on the hosted zone for each EC2 instance using the private IP address of each one
![Screenshot 2023-03-22 at 12 17 27](https://user-images.githubusercontent.com/31238382/226946538-23a2cc89-c7e5-4adb-929a-0641f3fe12f4.png)

## Create EC2 instance for Tomcat <br>
User data
```
#!/bin/bash
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-8-jdk -y
sudo apt install tomcat8 tomcat8-admin tomcat8-docs tomcat8-common git -y
```

## Create Artifact with Maven <br>
* Edit the application properties file in the ``` src/main/resources ``` folder
```
jdbc.url=jdbc:mysql://db01.vprofile.in:3306/accounts?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
memcached.active.host=mc01.vprofile.in
rabbitmq.address=rmq01.vprofile.in
```

* Switch to the root directory where the pom.xml file exists and build the artifact <br>
``` mvn install ```
![image](https://user-images.githubusercontent.com/31238382/226951466-3ee3af8e-251c-4f06-b610-7ff104f51b5d.png)

## Create an S3 bucket using AWS CLI <br>
```
aws s3 mb s3://vprofile-artifact-storage-rolake
aws s3 cp vprofile-v2.war s3://vprofile-artifact-storage-rolake
aws s3 ls vprofile-artifact-storage-rolake
```
![image](https://user-images.githubusercontent.com/31238382/226951660-20daca40-cff3-4ca3-bbe5-0b21f73f3473.png)

## Create a role for the Tomcat instance with S3FullAccess permission and attach it
![image](https://user-images.githubusercontent.com/31238382/226952546-3562ebb6-7641-475c-94f5-19a0fd8a6075.png)

## Download Maven Artifact to Tomcat server from S3
* Open port 22 on the Tomact instance security group and SSH into the (aap01) instance
```
ssh -i "vprofile-prod-key.cer" ubuntu@<public_ip_of_server>
sudo -i
systemctl status tomcat8
```

* Delete ROOT (where default tomcat app files stored) directory under /var/lib/tomcat8/webapps/. Before deleting it we need to stop Tomcat server.
```
cd /var/lib/tomcat8/webapps/
systemctl stop tomcat8
rm -rf ROOT
```

* Next we will download our artifact from s3 using aws cli commands. First we need to install aws cli. We will initially download our artifact to /tmp directory, then we will copy it under /var/lib/tomcat8/webapps/ directory as ROOT.war. Since this is the default app directory, Tomcat will extract the compressed file.
```
apt install awscli -y
aws s3 ls s3://vprofile-artifact-storage-rd
aws s3 cp s3://vprofile-artifact-storage-rd/vprofile-v2.war /tmp/vprofile-v2.war
cd /tmp
cp vprofile-v2.war /var/lib/tomcat8/webapps/ROOT.war
systemctl start tomcat8
```

* Check that the ```application.properties``` file has the updated changes
``` cat /var/lib/tomcat8/webapps/ROOT/WEB-INF/classes/application.properties ```

* Validate network connectivity
```
apt install telnet
telnet db01.vprofile.in 3306
```

## Create a Load Balancer
* Before the loab balancer, a target group must be created
![image](https://user-images.githubusercontent.com/31238382/226955079-d330ac4f-3185-4e45-b985-7ea7cd9dda58.png)

* Create an internet-facing application load balancer
![image](https://user-images.githubusercontent.com/31238382/226955290-083fe001-7a98-41e4-9bad-d9304c641772.png)

## Create Route53 record for the ELB endpoint
![image](https://user-images.githubusercontent.com/31238382/226956184-fc5c1416-54ed-47ec-81e6-3871888fc1dc.png)

## Validate your application using the record name
![image](https://user-images.githubusercontent.com/31238382/226956393-6c78d2d8-542f-4376-8897-ee1896f6ffdb.png)
![image](https://user-images.githubusercontent.com/31238382/226956481-8f93be8c-2d8e-4a97-97a0-1c5b5beed788.png)

## Configuare an auto-scaling group for the application instance
* Create an AMI for the app01 instance
* Create a launch configuration
![image](https://user-images.githubusercontent.com/31238382/226957280-f5e5e393-48f4-4d37-a723-5ed7f1c2a505.png)

* Create the auto-scaling group
![image](https://user-images.githubusercontent.com/31238382/226957411-457601a2-54d7-4942-bf66-85a4fb1082b5.png)

## Clean up
* Delete all AWS resources to avoid incurring charges

