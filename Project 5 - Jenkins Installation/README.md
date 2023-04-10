# Project 5 - Jenkins Pipeline as a Code 

## Prerequisites:
• AWS Account <br>
• Jenkins <br>
• Java SDK 8 <br>
• Maven

## Create key pair on AWS
* On the EC2 console, create a new key pair to use for the project
![image](https://user-images.githubusercontent.com/31238382/228039445-f4ef3ffa-38e4-4b59-9fce-30d9502a4442.png)

## Create an EC2 instance
User data for the instance <br>
```
#!/bin/bash
sudo apt update
sudo apt install openjdk-11-jdk -y
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```

## Create a security group for the instance
```
Name: jenkins-SG
Allow: SSH from MyIP
Allow: 8080 from MyIP
```

## Open the Address of the created EC2 instance 
Copy the public IPv4 address into your browser tab and add the 8080 port to it <br>
```<public_IPv4>:8080```
![image](https://user-images.githubusercontent.com/31238382/228040919-3e4d3b15-ac5e-4df7-9c64-2a59a1f07be5.png)

To get the administrator password, ssh into the EC2 instance <br>
```
 ssh -i "jenkins-key.cer" ubuntu@<public_IPv4_DNS>
 sudo -i
 cd /var/lib/jenkins/secrets
 cat initialAdminPassword
```
![image](https://user-images.githubusercontent.com/31238382/228041758-c6d87864-52fb-47a7-8379-66c1812cdf9b.png)

On the next page, click on ```Select plugins to install``` and click ```Install```, you'll get the page below <br>
![image](https://user-images.githubusercontent.com/31238382/228042203-218fa583-8100-4879-b369-9488cbbf5882.png)

## Create first admin user on Jenkins
Create a user with whatever details you wish
![image](https://user-images.githubusercontent.com/31238382/228042314-b6341085-d16d-42e1-8866-8309f8440c20.png)

Copy the Jenkins URL on your screen

## Ensure you have JDK8
On the Jenkins page, click ```Manage Jenkins```, then ```Global Tool Configuration``` <br>
Search for and click ```Add JDK``` <br>
SSH into the EC2 instance, install jdk 8 and copy the path <br>
```
sudo apt update
sudo apt install openjdk-8-jdk -y
sudo -i
ls /usr/lib/jvm
```
Paste the path in JAVA_HOME
```
/usr/lib/jvm/java-1.8.0-openjdk-amd64
```

Search and click for ```Add Maven```, check the install automatically box and hit save

## Install Plugins
* Click on ```Manage Jenkins``` <br>
Search for ```pipeline maven integration plugin``` and ```pipeline utility steps``` on the available plugin section and install <br>

## Create a pipeline script
On the Jenkins home page, click ```New Item```<br>
Enter item name ```sample-paac```, and select pipeline add the script below and click build now <br>

```
pipeline {
    agent any
    stages {
        stage('Fetch code') {
            steps {
                git branch: 'paac', url: 'https://github.com/devopshydclub/vprofile-project.git'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn install'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }
}
```
![image](https://user-images.githubusercontent.com/31238382/228047797-7821f8ac-c29d-44c1-94a7-7c21e858ebd4.png)

