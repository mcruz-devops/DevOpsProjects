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
``` 
mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'app01' identified by 'admin123' "
cd ../../..
mysql -u root -p"$DATABASE_PASS" accounts < src/main/resources/db_backup.sql
mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES" 
```

* Log into the database and do a quick verification <br>
``` 
mysql -u root -p"$DATABASE_PASS"
MariaDB [(none)]> show databases;
MariaDB [(none)]> use accounts;
MariaDB [(none)]> show tables;
exit 
```

* Restart the MariaDb server and logout <br>
``` 
systemctl restart mariadb
logout
```

## Provisioning Memcached
* Log into the memcached VM <br>
``` vagrant ssh mc01 ```

* Swicth to root user and update <br>
``` 
sudo -i
yum install update -y 
```

* Install the epel release package <br>
``` yum install epel-release -y ```

* Install the memcached packaged <br>
``` yum install memcached -y ```

* Start/enable the memcached service and check the status of service <br>
``` 
systemctl start memcached
systemctl enable memcached
systemctl status memcache
```

* Run one more command to ensure memcached can listen on TCP port 11211 and UDP port 11111 <br>
``` memcached -p 11211 -U 11111 -u memcached -d ```

* Validate if it is running on the right port <br>
``` ss -tunlp | grep 11211 ```
![image](https://user-images.githubusercontent.com/31238382/226073583-b6761f91-1908-43f0-a4d3-c32645f5a481.png)

## Provisioning RabbitMQ
* Log into the RabbitMQ VM <br>
``` vagrant ssh rmq01 ```

* Swicth to root user and update <br>
``` 
sudo -i
yum install update -y 
```

* Install the epel release package <br>
``` yum install epel-release -y ```

* Install these dependencies before installing RabbitMQ <br>
``` 
yum install wget -y
cd /tmp/
wget http://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm
sudo rpm -Uvh erlang-solutions-2.0-1.noarch.rpm
```

* Install RabbitMQ with the command below <br>
``` 
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash
sudo yum install rabbitmq-server -y 
```

* Start/Enable the rabbitmq service and check the status of service <br>
``` 
systemctl start rabbitmq-server
systemctl enable rabbitmq-server
systemctl status rabbitmq-server 
```

* Create a test user with the password test <br>
``` 
cd ~
echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
systemctl restart rabbitmq-server 
```

* Check the rabbitmq status and exit <br>
``` 
systemctl status rabbitmq-server
exit 
```

## Provisioning Tomcat
* Log into the Tomcat VM <br>
``` vagrant ssh app01 ```

* Swicth to root user and update <br>
``` 
sudo -i
yum install update -y 
```

* Install the epel release package <br>
``` yum install epel-release -y ```

* Install dependencies for the Tomcat server
```
yum install java-1.8.0-openjdk -y
yum install git maven wget -y
```

* Now we can download Tomcat. First switch to /tmp/ directory
``` 
cd /tmp/
wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.37/bin/apache-tomcat-8.5.37.tar.gz
tar xzvf apache-tomcat-8.5.37.tar.gz
```

* Add a tomcat user and copy data to tomcat home directory
``` useradd --home-dir /usr/local/tomcat8 --shell /sbin/nologin tomcat ```

* Copy data to /usr/local/tomcat8 directory which is the home-directory for tomcat user
```
cp -r /tmp/apache-tomcat-8.5.37/* /usr/local/tomcat8/
ls /usr/local/tomcat8
```
* Change ownership of all files from the root user to the tomcat user
```
ls -l /usr/local/tomcat8/
chown -R tomcat.tomcat /usr/local/tomcat8
ls -l /usr/local/tomcat8/
```

* Setup systemd for tomcat, create a file with the code below. After creating this file, we will be able to start tomcat service with systemctl start tomcat and stop tomcat with systemctl stop tomcat commands.
``` vi /etc/systemd/system/tomcat.service ```
Content to add tomcat.service file:
```
[Unit]
Description=Tomcat
After=network.target

[Service]
User=tomcat
WorkingDirectory=/usr/local/tomcat8
Environment=JRE_HOME=/usr/lib/jvm/jre
Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_HOME=/usr/local/tomcat8
Environment=CATALINE_BASE=/usr/local/tomcat8
ExecStart=/usr/local/tomcat8/bin/catalina.sh run
ExecStop=/usr/local/tomcat8/bin/shutdown.sh
SyslogIdentifier=tomcat-%i

[Install]
WantedBy=multi-user.target
```

* Run the command below to effect the changes
``` systemctl daemon-reload ```

Now we should be able to enable tomcat service. The service name tomcat has to be same as given /etc/systemd/system/tomcat.service directory.
```
systemctl enable tomcat
systemctl start tomcat
systemctl status tomcat
```

* Our Tomcat server is active running, now we will build our source code and deploy it to Tomcat server.
Code Build & Deploy to Tomcat(app01) Server
Clone the source code on the /tmp directory
```
git clone https://github.com/devopshydclub/vprofile-project.git
ls
cd vprofile-project/
```

Before we build our artifact, we need to update our configuration file that will be connect to our backend services db, memcached and rabbitmq service.
```
vi src/main/resources/application.properties
application.properties file: Here we need to make sure the settings are correct. First check DB configuration. Our db server is db01 , and we have admin user with password admin123 as we setup. For memcached service, hostname is mc01 and we validated it is listening on tcp port 11211. Fort rabbitMQ, hostname is rmq01 and we have created user test with pwd test.
#JDBC Configutation for Database Connection
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://db01:3306/accounts?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
jdbc.username=admin
jdbc.password=admin123

#Memcached Configuration For Active and StandBy Host
#For Active Host
memcached.active.host=mc01
memcached.active.port=11211
#For StandBy Host
memcached.standBy.host=127.0.0.2
memcached.standBy.port=11211

#RabbitMq Configuration
rabbitmq.address=rmq01
rabbitmq.port=5672
rabbitmq.username=test
rabbitmq.password=test
```

* Run the "mvn install" command which will create our artifact. The artifact will be created /tmp/vprofile-project/target/vprofile-v2.war
```
cd target/
ls
```

* Deploy the artifact vprofile-v2.war to the Tomcat server. But before that, we will remove default app from our server. For that reason first we will shutdown server. The default app will be in /usr/local/tomcat8/webapps/ROOT directory.
```
systemctl stop tomcat
systemctl status tomcat
rm -rf /usr/local/tomcat8/webapps/ROOT
```

* Our artifact is under vprofile-project/target directory. Now we will copy our artifact to /usr/local/tomcat8/webapps/ directory as ROOT.war and start tomcat server. Once we start the server, it will extract our artifact ROOT.war under ROOT directory.
```
cd ..
cp target/vprofile-v2.war /usr/local/tomcat8/webapps/ROOT.war
systemctl start tomcat
ls /usr/local/tomcat8/webapps/
```
By the time, our application is coming up we can provision our Nginx server.

# Provisioning Nginx
The Nginx server is Ubuntu, while other servers are RedHat. Install updates with the command below
```sudo apt update && sudo apt upgrade```

* Install nginx
```
sudo -i
apt install nginx -y
```
* Create an Nginx configuration file under directory /etc/nginx/sites-available/ with below content:
```vi /etc/nginx/sites-available/vproapp```
Content to add:
```
upstream vproapp {
server app01:8080;
}
server {
listen 80;
location / {
proxy_pass http://vproapp;
}
}
```

* Remove the default nginx configuration file
```rm -rf /etc/nginx/sites-enabled/default```

* After that, create a symbolic link for the configuration file using default config location as below to enable our site. Then restart nginx server.
```
ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
systemctl restart nginx
```

# Validate Application from Browser
* We are in web01 server, run ifconfig to get its IP address. So the IP address of our web01 is : 192.168.56.11
![image](https://user-images.githubusercontent.com/31238382/226170246-e3cdf9b6-5185-4416-b7cb-2fb27871668c.png)

* Validate the db connection by logging in with admin_vp as username and password
![image](https://user-images.githubusercontent.com/31238382/226170352-151d4493-a8fb-4225-9947-ce2393898ab5.png)

* Validate the RabbitMQ connection by clicking RabbitMQ on the page
![image](https://user-images.githubusercontent.com/31238382/226170384-4269886c-73f7-4011-b16f-3056c184a536.png)

* Validate the Memcached connection
![image](https://user-images.githubusercontent.com/31238382/226170415-7231a685-6877-46f9-8f79-0b594c2202ca.png)

![image](https://user-images.githubusercontent.com/31238382/226170422-2779a1c1-9c03-45f3-bb96-e55edcb2682c.png)

* After checking that everything works correctly, destroy the VMs using the vagrant destroy command and check your virtualbox manager to confirm the machines are indeed destroyed.
``` vagrant destroy ```
![image](https://user-images.githubusercontent.com/31238382/226170467-3cbbb3b1-47ff-46d3-9cc4-fcc40bf6d77d.png)


```
