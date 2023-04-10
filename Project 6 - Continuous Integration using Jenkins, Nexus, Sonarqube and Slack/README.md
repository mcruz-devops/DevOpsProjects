# Project 6 - Continuous Integration using Jenkins, Nexus, Sonarqube and Slack
 

## Prerequisites:
• AWS Account <br>
• Jenkins <br>
• Java SDK 8 <br>
• Nexus <br>
• Sonarqube <br>
• Maven <br>

## Create key pair on AWS
* On the EC2 console, create a new key pair to use for the project
![image](https://user-images.githubusercontent.com/31238382/228039445-f4ef3ffa-38e4-4b59-9fce-30d9502a4442.png)

## Create security groups for the instances
Security Group for Jenkins <br>
```
Name: jenkins-SG
Allow: SSH from MyIP
Allow: 8080 from MyIP
```

Security Group for Nexus <br>
```
Name: nexus-SG
Allow: SSH from MyIP
Allow: 8081 from MyIP and Jenkins-SG
```

Security Group for SonarQube <br>
```
Name: sonar-SG
Allow: SSH from MyIP
Allow: 80 from MyIP and Jenkins-SG
```

## Create three EC2 instances
Properties for Jenkins instance <br>
```
Name: jenkins-server
AMI: Ubuntu 20.04
SecGrp: jenkins-SG
InstanceType: t2.small
KeyPair: jenkins-key
Additional Details: Add the user data
```

User Data for the Jenkins instance <br>
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
Properties for Nexus Instance <br>
```
Name: nexus-server
AMI: Amazon Linux-2
InstanceType: t2.medium
SecGrp: nexus-SG
KeyPair: jenkins-key
Additional Details: Add the nexus userdata below
```

User data for the Nexus instance <br>
```
#!/bin/bash
yum install java-1.8.0-openjdk.x86_64 wget -y   
mkdir -p /opt/nexus/   
mkdir -p /tmp/nexus/                           
cd /tmp/nexus/
NEXUSURL="https://download.sonatype.com/nexus/3/latest-unix.tar.gz"
wget $NEXUSURL -O nexus.tar.gz
EXTOUT=`tar xzvf nexus.tar.gz`
NEXUSDIR=`echo $EXTOUT | cut -d '/' -f1`
rm -rf /tmp/nexus/nexus.tar.gz
rsync -avzh /tmp/nexus/ /opt/nexus/
useradd nexus
chown -R nexus.nexus /opt/nexus 
cat <<EOT>> /etc/systemd/system/nexus.service
[Unit]                                                                          
Description=nexus service                                                       
After=network.target                                                            
                                                                  
[Service]                                                                       
Type=forking                                                                    
LimitNOFILE=65536                                                               
ExecStart=/opt/nexus/$NEXUSDIR/bin/nexus start                                  
ExecStop=/opt/nexus/$NEXUSDIR/bin/nexus stop                                    
User=nexus                                                                      
Restart=on-abort                                                                
                                                                  
[Install]                                                                       
WantedBy=multi-user.target                                                      

EOT

echo 'run_as_user="nexus"' > /opt/nexus/$NEXUSDIR/bin/nexus.rc
systemctl daemon-reload
systemctl start nexus
systemctl enable nexus
```

Properties for SonarQube Instance <br>
```
Name: sonar-server
AMI: Ubuntu 20.04
InstanceType: t2.medium
SecGrp: sonar-SG
KeyPair: vprofile-ci-key
Additional Details: Add userdata for sonarqube
```

User Data for SonarQube Instance <br>
```
#!/bin/bash
cp /etc/sysctl.conf /root/sysctl.conf_backup
cat <<EOT> /etc/sysctl.conf
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
EOT
cp /etc/security/limits.conf /root/sec_limit.conf_backup
cat <<EOT> /etc/security/limits.conf
sonarqube   -   nofile   65536
sonarqube   -   nproc    409
EOT

sudo apt-get update -y
sudo apt-get install openjdk-11-jdk -y
sudo update-alternatives --config java

java -version

sudo apt update
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -

sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
sudo apt install postgresql postgresql-contrib -y
#sudo -u postgres psql -c "SELECT version();"
sudo systemctl enable postgresql.service
sudo systemctl start  postgresql.service
sudo echo "postgres:admin123" | chpasswd
runuser -l postgres -c "createuser sonar"
sudo -i -u postgres psql -c "ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin123';"
sudo -i -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
sudo -i -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;"
systemctl restart  postgresql
#systemctl status -l   postgresql
netstat -tulpena | grep postgres
sudo mkdir -p /sonarqube/
cd /sonarqube/
sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
sudo apt-get install zip -y
sudo unzip -o sonarqube-8.3.0.34182.zip -d /opt/
sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube
sudo groupadd sonar
sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
sudo chown sonar:sonar /opt/sonarqube/ -R
cp /opt/sonarqube/conf/sonar.properties /root/sonar.properties_backup
cat <<EOT> /opt/sonarqube/conf/sonar.properties
sonar.jdbc.username=sonar
sonar.jdbc.password=admin123
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
sonar.web.host=0.0.0.0
sonar.web.port=9000
sonar.web.javaAdditionalOpts=-server
sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
sonar.log.level=INFO
sonar.path.logs=logs
EOT

cat <<EOT> /etc/systemd/system/sonarqube.service
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonar
Group=sonar
Restart=always

LimitNOFILE=65536
LimitNPROC=4096


[Install]
WantedBy=multi-user.target
EOT

systemctl daemon-reload
systemctl enable sonarqube.service
#systemctl start sonarqube.service
#systemctl status -l sonarqube.service
apt-get install nginx -y
rm -rf /etc/nginx/sites-enabled/default
rm -rf /etc/nginx/sites-available/default
cat <<EOT> /etc/nginx/sites-available/sonarqube
server{
    listen      80;
    server_name sonarqube.groophy.in;

    access_log  /var/log/nginx/sonar.access.log;
    error_log   /var/log/nginx/sonar.error.log;

    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    location / {
        proxy_pass  http://127.0.0.1:9000;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;
              
        proxy_set_header    Host            \$host;
        proxy_set_header    X-Real-IP       \$remote_addr;
        proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto http;
    }
}
EOT
ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
systemctl enable nginx.service
#systemctl restart nginx.service
sudo ufw allow 80,9000,9001/tcp

echo "System reboot in 30 sec"
sleep 30
reboot
```

## Post Installation Steps
### For the Jenkins server 
• SSH into the instance and get the ```initialAdminPassword``` <br>
```
sudo -i
system status jenkins
cat /var/lib/jenkins/secrets/initialAdminPassword
```

• Get the public IP of the Jenkins EC2 instance and open it in the browser with port 8080 appended to it
```http://<public_ip_of_jenkins_server>:8080```

• Install selected plugins
```
Maven Integration
Github Integration
Nexus Artifact Uploader
SonarQube Scanner
Slack Notification
Build Timestamp
```

### For Nexus
• SSH into the instance and check the service status <br>
```systemctl status nexus```

• Get the public IP of the Nexus EC2 instance and open it in the browser with port 8081 appended to it. Select sign in <br>
```http://<public_ip_of_nexuss_server>:8081```

• Get the password <br>
```cat /opt/nexus/sonatype-work/nexus3/admin.password```

• Login with username ```admin```, and the password you copied

• Create repositories
```
maven2 hosted
Name: vprofile-release
Version policy: Release
```

```
maven2 proxy
Name: vpro-maven-central
remote storage: https://repo1.maven.org/maven2/
```

```
maven2 hosted
Name: vprofile-snapshot
Version policy: Snapshot
```

```
maven2 group
Name: vpro-maven-group
Member repositories: 
 - vprofile-maven-central
 - vprofile-release
 - vprofile-snapshot
```
![image](https://user-images.githubusercontent.com/31238382/230965947-f56d99db-cd0d-4b92-ba33-c5b358aa6266.png)

### For SonarQube
• Get the public IP of the SonarQube EC2 instance and paste it in the browser <br>
• Login with admin as both username and password

## Git Code Migration
• Create a new repository on Github <br>
• Clone into the repo from https://github.com/devopshydclub/vprofile-project.git
``` git clone -b ci-jenkins https://github.com/devopshydclub/vprofile-project.git ```

## Build job with Nexus Repo
• SSH into the Jenkins server
```
sudo apt update -y
sudo apt install openjdk-8-jdk -y
sudo -i
ls /usr/lib/jvm
```

• Go back to your browser tab for Jenkins, first step is building the artefact with Maven <br>
• Go to ```Manage Jenkins``` -> ```Global Tool Configuration``` <br>
• Search for JDK and add the following
```
Under JDK -> Add JDK
Name: OracleJDK8
untick Install Automatically
JAVA_HOME: /usr/lib/jvm/java-1.8.0-openjdk-amd64
```

• On the same ```Global Tool Congiguration``` page, search for Maven <br>
```
Name: MAVEN3
```

• Add Nexus login credentials to Jenkins. Go to Manage Jenkins -> Manage Credentials -> Global -> Add Credentials <br>
```
username: admin
password: <password_setup_for_nexus>
ID: nexuslogin
description: nexuslogin
```

• Create Jenkinsfile with the following data <br>
```
pipeline {
    agent any
    tools {
        maven "MAVEN3"
        jdk "OracleJDK8"
    }

    environment {
        SNAP_REPO = 'vprofile-snapshot'
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'admin'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vprofile-maven-central'
        NEXUSIP = '172.31.19.41'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN = 'nexuslogin'
        SONARSERVER = 'sonarserver'
        SONARSCANNER = 'sonarscanner'
    }

    stages {
        stage('Build'){
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo "Now Archiving"
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }
    }
```

• Create a new Jenkins job with these details <br>
```
Pipeline from SCM 
Git
URL: <url_from_project> I will use SSH link
Crdentials: we will create github login credentials
#### add Jenkins credentials for github ####
Kind: SSH Username with private key
ID: githublogin
Description: githublogin
Username: git
Private key file: paste your private key here
#####
Branch: */ci-jenkins
path: Jenkinsfile
```

* An error is returned, to fix this, ssh into the jenkins server and run the commands below <br>
```
sudo -i
sudo su - jenkins
git ls-remote -h -- git@github.com:RolakeAnifowose/vprofile-project.git HEAD
```

* Build your Jenkins pipeline <br>
![image](https://user-images.githubusercontent.com/31238382/230970879-10bfab7c-f1f6-4d4c-8796-1c2602e427e7.png)


## Github Webhook
• Create a Github webhook
![image](https://user-images.githubusercontent.com/31238382/230969104-c1a7839f-c257-4457-9415-634cce6bbf95.png)

• Go back to Jenkins and add the following to the vprofile-ci-pipeline job <br>
```Build Trigger: GitHub hook trigger for GITScm polling```

• Update your Jenkins file and then commit and push
```
        stage('Test'){
            steps {
                sh 'mvn -s settings.xml test'
            }
        }

        stage('Checkstyle Analysis'){
            steps {
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
        }
```

## Code Analysis with SonarQube
• Start with the SonarScanner tool configuration. Go to Manage Jenkins -> Global Tool Configuration
```
Add sonar scanner
name: sonarscanner
Select install automatically
```

• Next we need to go to Configure System, and find  SonarQube servers section
```
Check environment variables
Add sonarqube
Name: sonarserver
Server URL: http://<private_ip_of_sonar_server>
Server authentication token: Create a token from sonar website
```
![image](https://user-images.githubusercontent.com/31238382/230970055-971cb8ad-23bc-48d1-86a9-064ace3d70e2.png)

• Add the sonar token to the credentials
```
Kind: secret text
Secret: <generated token>
name: sonartoken
description: sonartoken
```

• Add SonarQube to your Jenkins pipeline. On the Jenkins file, add the code below <br>
```
 stage('CodeAnalysis') {
          
          environment {
             scannerHome = tool "${SONARSCANNER}"
          }

          steps {
            withSonarQubeEnv("${SONARSERVER}") {
               sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
            }

          }
     }
```
![image](https://user-images.githubusercontent.com/31238382/230970715-07ebf12c-1de2-4cf3-869e-2fd934d95ccf.png)

• Go to SonarQube 
![image](https://user-images.githubusercontent.com/31238382/230970941-6435447b-e458-4001-aa5d-9cf32c7e1239.png)
![image](https://user-images.githubusercontent.com/31238382/230971022-c1f4d62f-eb9f-4449-9ce3-f31b79e95149.png)

* Create your own Quality gate
Click Quality gate -> Create. Add Condition. You can give Bug is greater than 90 then Save it. Click on projectname, it will have a dropdown, click Quality Gate and choose the new Quality gate you have created.

• Add this to your Jenkinsfile and check the pipeline after its built <br>
```
stage('QUALITY GATE') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
            }
            }
        }
```
![image](https://user-images.githubusercontent.com/31238382/230971416-0c8b86fa-be90-4cac-999a-42cf198eb4f9.png)

## Publish Artifact to Nexus Repo
• On Jenkins, go to Manage Jenkins -> Configure System. Under Build Timestamp, update the pattern to: <br>
```yy-MM-dd_HHmm```

• Add the code below to your Jenkins file, and commit & push
```
stage('UploadArtifact') {
                steps {
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                        groupId: 'QA',
                        version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                        repository: "${RELEASE_REPO}",
                        credentialsId: ${NEXUS_LOGIN},
                        artifacts: [
                            [artifactId: 'vproapp' ,
                            classifier: '',
                            file: 'target/vprofile-v2.war',
                            type: 'war']
                        ]
                    )
                }
        }
 ```
 
 • Check your pipeline
 ![image](https://user-images.githubusercontent.com/31238382/230972025-f226cd3d-ef0d-46da-8b48-2927986db954.png)

• Open your Nexus browser tab, select ```Browse``` and you can see the artefact
![image](https://user-images.githubusercontent.com/31238382/230972250-3fd5014b-01b2-4e6e-9627-0ceaffa7e402.png)

## Slack Notification
• Create a new workspace in Slack ```vprofile```. Create a channel ```jenkinscicd``` <br>

• Add the Jenkins app to Slack. Open your browser and type in ```Slack Apps```. Click the link and search for ```Jenkins CI```. Select add to Slack and pick the ```jenkinscicd``` channel <br>

• Go back to Jenkins. Configure system -> Slack <br>
```
Workspace:  vprofile
credential: slacktoken 
default channel: #jenkinscicd
```

• Create a Slack token
```
Kind: secret text
Secret: <token for the workspace you created>
name: slacktoken
description: slacktoken
```
![image](https://user-images.githubusercontent.com/31238382/230973137-1792dab2-b996-48c0-86f6-126a0e0b75b5.png)

• Add this to your Jenkinsfile
Color code for Slack, add on line 1
```
def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger'
]
```

```
post{
        always {
            echo 'Slack Notifications'
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
```

![image](https://user-images.githubusercontent.com/31238382/230973270-99c8ef45-05b0-4ce4-b1ce-afc6605ca619.png)

## Cleanup
• Delete the EC2 instances so you don't incur exorbitant charges


