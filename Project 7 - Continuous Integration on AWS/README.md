## Project 7 - Continuous Integration on AWS

![image](https://user-images.githubusercontent.com/31238382/231545952-f3bbd9a5-b5c4-4a13-b3e1-a668521afe08.png)

### AWS CodeCommit Setup
• Search for CodeCommit on the AWS console and create a repository <br>
```vprofile-code-repo```

• Go to IAM. Create a new user and allow codeCommitFullAccess <br>
```Name: vprofile-code-admin-repo-fullaccess```

• Generate an SSH key to connect to CodeCommit by following these rules
![image](https://user-images.githubusercontent.com/31238382/231552269-6d0d63d5-5c7f-4de8-bb39-e7d744fe2eb0.png)

• Test your connection<br>
```ssh git-codecommit.us-east-1.amazonaws.com```
![image](https://user-images.githubusercontent.com/31238382/231552620-041a608c-e3d4-4632-b831-850109551a26.png)

• Clone the project repository from your machine to CodeCommit. Switch to the directory containing the project and run these:<br>
```
git checkout master
git branch -a | grep -v HEAD | cut -d'/' -f3 | grep -v master > /tmp/branches
for i in `cat  /tmp/branches`; do git checkout $i; done
git fetch --tags
git remote rm origin
git remote add origin ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/vprofile-code-repo
cat .git/config
git push origin --all
git push --tags
```

• Check CodeCommit, you should see your code<br>
![image](https://user-images.githubusercontent.com/31238382/231553118-3f7a89c7-c94b-47a6-ab74-5de82da5f358.png)

## CodeArtifact
• Search for CodeArtifact on the AWS console and create a repository <br>
```
Name: vprofile-maven-repo
Public upstraem Repo: maven-central-store
This AWS account
Domain name: visualpath
```
![image](https://user-images.githubusercontent.com/31238382/231553582-55ecd57a-d420-4048-9608-86fb43741b8c.png)

• Follow the connection instructions given in CodeArtifact for maven-central-repo
![image](https://user-images.githubusercontent.com/31238382/231554031-082c4617-c53b-4f0e-90ea-c58fd827f680.png)

• Create an IAM user for CodeArtifact and then configure AWS CLI using its credentials <br>
```aws configure```
![image](https://user-images.githubusercontent.com/31238382/231554627-5b84cee4-c292-488e-a208-ee9f168bb3eb.png)

• Get the CodeArtifact Token <br>
```export CODEARTIFACT_AUTH_TOKEN=`aws codeartifact get-authorization-token --domain visualpath --domain-owner 556298987240 --region us-east-1 --query authorizationToken --output text````

• Update the pom.xml and settings.xml files changing the url, domain name to match what's on CodeArtifact <br>

• Push the changes to CodeCommit <br>
```
git add .
git commit -m "Edited settings and pom files"
git push origin ci-aws
```

### SonarCloud
• Go to SonarCloud and create an account if you don't have one <br>

• Go to My Account -> Security and generate token with the name ```vprofile-sonartoken``` <br>

• Create a Project. Click the ```+``` -> Analyse project -> Create project manually <br>
```
Organization: sonar-devops-projects
Project key: vprofile-sonar-repo
Public
```
![image](https://user-images.githubusercontent.com/31238382/231556256-158450f7-8d4c-4cda-983d-1742f2a84895.png)

### Systems Manager Parameter Store
• Create parameters on the parameter store <br>
```
CODEARTIFACT_TOKEN - SecureString	
HOST - https://sonarcloud.io
ORGANIZATION - rumeysa-devops-projects
PROJECT - vprofile-sonar-repo
sonartoken - SecureString
```

### CodeBuild for SonarQube Analysis
• Go to CodeBuild on the console and create a new project <br>
```
ProjectName: vprofile-build
Source: CodeCommit
Branch: ci-aws
Environment: Ubuntu
runtime: standard:5.0
New service role
Insert build commands from foler aws-files/sonar_buildspec.yml
Logs-> GroupName: vprofile-buildlogs
StreamName: sonarbuildjob
```

• We need to add a policy to the service role created for this Build project. Find name of role from Environment, go to IAM add policy as below:
![image](https://user-images.githubusercontent.com/31238382/231557045-1b4a9ddf-5d71-4451-bb4e-b56621d32e14.png)

• Build the project
![image](https://user-images.githubusercontent.com/31238382/231557192-0a64c780-1548-46f7-ae10-f695595e3fff.png)

• Check SonarCloud too
![image](https://user-images.githubusercontent.com/31238382/231557379-18a9ff1b-a157-45af-a514-6caf67a5d746.png)

### CodeBuild for CodeArtifact
• Go to CodeBuild and create a new BuildProject<br>
```
ProjectName: vprofile-Build-Artifact
Source: CodeCommit
Branch: ci-aws
Environment: Ubuntu
runtime: standard:5.0
Use existing role from previous build
Insert build commands from foler aws-files/build_buildspec.yml
Logs-> GroupName: vprofile-buildlogs
StreamName: buildjob
```

• Build the project
![image](https://user-images.githubusercontent.com/31238382/231557878-8a7e6800-056d-478d-88d3-f9b40e2ce81c.png)

### AWS CodePipeline & Notification with SNS
• Go to SNS on the console, create a new topic and subscribe to that topic via email
![image](https://user-images.githubusercontent.com/31238382/231558085-7b7c4bef-2749-477a-a411-e3f3c59b04b4.png)

• Create an S3 bucket to hold artifacts
![image](https://user-images.githubusercontent.com/31238382/231558332-9ace5e9f-630f-43d8-98cf-d51ec99c2cdf.png)

• Create your pipeline
```
Name: vprofile-CI-Pipeline
SourceProvider: Codecommit
branch: ci-aws
Change detection options: CloudWatch events
Build Provider: CodeBuild
ProjectName: vprofile-Build-Aetifact
BuildType: single build
Deploy provider: Amazon S3
Bucket name: vprofile98-build-artifact
object name: pipeline-artifact
```

• Add test and deploy stages to your pipeline
![image](https://user-images.githubusercontent.com/31238382/231558714-4496a633-c6e7-409b-845f-678fdb300f40.png)
![image](https://user-images.githubusercontent.com/31238382/231558781-edefcdbb-bf5e-4467-a794-76eef2e46e8c.png)

• Check your S3 bucket to see if the artifacts are there
![image](https://user-images.githubusercontent.com/31238382/231558920-65b371de-6ed7-453d-89da-07ff80242f0a.png)

### Cleanup
• Delete the AWS resources you created

