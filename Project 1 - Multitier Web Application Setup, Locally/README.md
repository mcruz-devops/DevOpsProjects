# Project 1 - Multitier Web Application Setup, Locally

![image](https://user-images.githubusercontent.com/31238382/226070269-5d5477f7-2f2b-4061-83b7-6cc709ba3486.png)

# Prerequisites
* Vagrant
* Oracle VM Virtualbox
* JDK 1.8 or later
* Maven 3 or later
* MySQL 5.6 or later

# VM Setup
* Clone the project repository to your local machine <br>
``` git clone https://github.com/devopshydclub/vprofile-project.git ```

* Switch to branch local-setup <br>
``` git checkout local-setup ```

* Cd into the Manual_provisioning directory <br>
``` cd vagrant/Manual_provisioning ```

* Install vagrant plugin <br>
``` vagrant plugin install vagrant-hostmanager ```

* Bring up the VMs <br>
``` vagrant up ```

![image](https://user-images.githubusercontent.com/31238382/226071171-a75191ae-c762-4257-924a-ec8ec592d43a.png)

* Validate the VMs with the vagrant ssh command <br>
``` vagrant ssh web01 ```

* Check the hosts file <br>
``` cat /etc/hosts/ ``` <br>
![image](https://user-images.githubusercontent.com/31238382/226071377-a23a6f0f-2ba9-4c38-8fe7-a66f9e9722d0.png)

# Provisioning
* Services to Provision
  * Nginx
  * Tomcat
  * RabbitMQ
  * Memcached
  * ElasticSearch
  * MySQL
  
 Order to Provision Services: MySQL -> Memcached -> RabbitMQ -> Tomcat -> Nginx
 
 ## Provisioning MySQL
 * Log in to db01 <br>
 ``` vagrant ssh db01 ```
 
 * Switch to root user and update packages <br>
 ``` sudo -i ``` <br>
 ``` yum update -y ```

 * Set the db password using the environmental variable DATABASE_PASS <br>
 ``` DATABASE_PASS = admin123 ```
 
 * To make the variable permanent, add it /etc/profilr file and update file <br>
``` vi /etc/profile ``` <br>
``` source /etc/profile ```

* Install the epel-release repository <br>
``` yum install epel-release -y ```

* Install the MariaDB package <br>
``` yum install git mariadb-server -y ```

* After installing MariaDb, start the service, enable it and check its status <br>
``` systemctl start mariadb ```
``` systemctl enable mariadb ```
``` systemctl status mariadb ```

* RUN mysql secure installation script <br>
``` mysql_secure_installation ```
Use admin123 as root password
``` [root@db01 ~]# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] y
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB! 
```

* Clone the source code <br>
``` git clone -b local-setup https://github.com/devopshydclub/vprofile-project.git ```

* Change directory to the resources directory <br>
``` cd vprofile-project/src/main/resources/ ```

* Initialize the database <br>
``` mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'app01' identified by 'admin123' "
cd ../../..
mysql -u root -p"$DATABASE_PASS" accounts < src/main/resources/db_backup.sql
mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES" ```

* Log into the database and do a quick verification <br>
``` mysql -u root -p"$DATABASE_PASS"
MariaDB [(none)]> show databases;
MariaDB [(none)]> use accounts;
MariaDB [(none)]> show tables;
exit ```

* Restart the MariaDb server and logout <br>
``` 
systemctl restart mariadb
logout ``

## Provisioning Memcached
* Log into the memcached VM <br>
``` vagrant ssh mc01 ```

* Swicth to root user and update <br>
``` sudo -i ``` <br>
``` yum install update -y ```

* Install the epel release package <br>
``` yum install epel-release -y ```

* Install the memcached packaged <br>
``` yum install memcached -y

* Start/enable the memcached service and check the status of service <br>
``` systemctl start memcached
systemctl enable memcached
systemctl status memcache ```

* Run one more command to ensure memcached can listen on TCP port 11211 and UDP port 11111 <br>
``` memcached -p 11211 -U 11111 -u memcached -d ```

* Validate if it is running on the right port <br>
``` ss -tunlp | grep 11211 ```
![image](https://user-images.githubusercontent.com/31238382/226073583-b6761f91-1908-43f0-a4d3-c32645f5a481.png)

## Provisioning RabbitMQ
* Log into the RabbitMQ VM <br>
``` vagrant ssh rmq01 ```

* Swicth to root user and update <br>
``` sudo -i
yum install update -y ```

* Install the epel release package <br>
``` yum install epel-release -y ```

* Install these dependencies before installing RabbitMQ <br>
``` yum install wget -y
cd /tmp/
wget http://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm
sudo rpm -Uvh erlang-solutions-2.0-1.noarch.rpm ```

* Install RabbitMQ with the command below <br>
``` curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash
sudo yum install rabbitmq-server -y ```

* Start/Enable the rabbitmq service and check the status of service <br>
``` systemctl start rabbitmq-server
systemctl enable rabbitmq-server
systemctl status rabbitmq-server ```

* Create a test user with the password test <br>
``` cd ~
echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
systemctl restart rabbitmq-server ```

* Check the rabbitmq status and exit <br>
``` systemctl status rabbitmq-server
exit ```

