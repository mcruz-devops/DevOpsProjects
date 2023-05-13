# Project 8 - Ansible for AWS VPC

### Step 1 - Ansible Setup for AWS
Switch to the Ohio region (us-east-2) and create an EC2 instance with these details
```
AMI: Ubuntu 20.04
Instance Type: t2.micro
Security Group: allow SSH on port 22
Create a new keypair
UserData:
#!/bin/bash
apt update
apt install ansible -y
```

SSH into the newly created instance and check the Ansible version <br>
```
ansible --version
```
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/6f3579d0-e405-40c9-8caf-771b4dc54078)

Install the AWS CLI on the instance
```
sudo apt install awscli -y
aws sts get-caller-identity
```

### Step 2 - Warmup for AWS Cloud Playbooks
Create a new directory on the instance, switch to that directory and create a test playbook
```
mkdir vpc-stack-vprofile
cd vpc-stack-vprofile
nano test-aws.yml
```
Paste this in your test-aws.yml file
```
- hosts: localhost
  connection: local
  gather_facts: False
  tasks:
    - name: sample ec2 key
      ec2_key:
        name: sample
        region: us-east-2
```

Try to run the playbook. This will likely return an error saying Python Boto is required. Install it with these commands
```
ansible-playbook test-aws.yml
sudo apt search boto
sudo apt install python3-boto3 -y
```

Now try running the playbook again after boto3 installation
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/ab738ce5-d625-4ec9-b435-5e1a414c53a8)

Add a extra task to the playbook 
```
- hosts: localhost
  connection: local
  gather_facts: False
  tasks:
    - name: sample ec2 key
      ec2_key:
        name: my_keypair
        region: us-east-2
      register: keyout

    - debug:
        var: keyout
```

When you run the playbook again, you can now see your key
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/50fc2ffd-f137-449b-bb14-6aff32ce1112)

Add one more task to store the key
```
- hosts: localhost
  connection: local
  gather_facts: False
  tasks:
    - name: sample ec2 key
      ec2_key:
        name: my_keypair
        region: us-east-2
      register: keyout

    - debug:
        var: keyout

    - name: store login key
      copy:
        content: "{{keyout.key.private_key}}"
        dest: ./sample-key.pem
      when: keyout.changed
```

### Step 3 - VPC Variables
Create a public GitHub repository to store the Ansible playbooks. Clone that repo into your local machine and start creating new files.
Create a ```vpc_setup``` file with the following:
```
vpc_name: "Vprofile-vpc"

#VPC Range
vpcCidr: '172.20.0.0./16'

#Subnets Range
PubSub1Cidr: 172.20.1.0/24
PubSub2Cidr: 172.20.2.0/24
PubSub3Cidr: 172.20.3.0/24
PrivSub1Cidr: 172.20.4.0/24
PrivSub2Cidr: 172.20.5.0/24
PrivSub3Cidr: 172.20.6.0/24

#Region Name
region: "us-east-2"

#Zone Names
zone1: us-east-2a
zone2: us-east-2b
zone3: us-east-2c

state: present
```

Create a ```bastion_setup``` file with the following:
```
bastion_ami: ami-083eed19fc801d7a4 #Amazon Linux-2 AMI-ID from us-east-2 region
region: us-east-2
MYIP: IP_address_of_your_laptop/32
keyName: vprofile-key
instanceType: t2.micro
```
Push these files to the GitHub repository and clone them into the EC2 Ansible server

### Step 4 - VPC Play
Create a new playbook ```vpc_setup.yml``` with this content
```
- hosts: localhost
  connection: local
  gather_facts: False
  tasks:
    - name: Import VPC Variables
      include_vars: vars/vpc_setup

    - name: Create Vprofile VPC
      ec2_vpc_net:
        name: "{{vpc_name}}"
        cidr_block: "{{vpcCidr}}"
        region: "{{region}}"
        dns_support: yes
        dns_hostnames: yes
        tenancy: default
        state: "{{state}}"
      register: vpcout
```
Commit/push the new file to GitHub. Pull into the Ansible server and run the playbook
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/9f294ec6-6029-4e91-99d6-47f48cec642e)

![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/e98173d5-c033-4d6c-a56c-134329b46061)

Confirm that the VPC is created in your AWS console
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/4efae6f1-e60b-4beb-a2fc-88b394923e13)

### Step 5 - Subnets Play
Add the following to the vpc_setup.yml file to create subnets. Make sure indentation is correct
```
    - name: create Public Subnet 1 in Zone1
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        az: "{{zone1}}"
        state: "{{state}}"
        cidr: "{{PubSub1Cidr}}"
        map_public: yes
        tags:
            Name: vprofile-pubsub1
      register: pubsub1_out

    - name: create Public Subnet 2 in Zone2
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        az: "{{zone2}}"
        state: "{{state}}"
        cidr: "{{PubSub2Cidr}}"
        map_public: yes
        tags:
          Name: vprofile-pubsub2
      register: pubsub2_out

    - name: create Public Subnet 3 in Zone3
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        az: "{{zone3}}"
        state: "{{state}}"
        cidr: "{{PubSub3Cidr}}"
        map_public: yes
        tags:
          Name: vprofile-pubsub3
      register: pubsub3_out

    - name: create Private Subnet 1 in Zone1
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        az: "{{zone1}}"
        state: "{{state}}"
        cidr: "{{PrivSub1Cidr}}"
        map_public: yes
        tags:
            Name: vprofile-privsub1
      register: privsub1_out

    - name: create Private Subnet 2 in Zone2
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        az: "{{zone2}}"
        state: "{{state}}"
        cidr: "{{PrivSub2Cidr}}"
        map_public: yes
        tags:
          Name: vprofile-privsub2
      register: privsub2_out

    - name: create Private Subnet 3 in Zone3
      ec2_vpc_subnet:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        az: "{{zone3}}"
        state: "{{state}}"
        cidr: "{{PrivSub3Cidr}}"
        map_public: yes
        tags:
          Name: vprofile-privsub3
      register: privsub3_out
```
Commit/push the changes to GitHub. Pull into the Ansible server and run the playbook
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/4b922d70-0110-4e52-8961-428f8ccfe14a)

Check the AWS console
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/54934460-29cf-4334-83ac-67611dae2c79)

### Step 6 - Internet Gateway and Route Table
```
- name: Internet Gateway Setup
      ec2_vpc_igw:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        state:  "{{state}}"
        tags:
          Name: vprofile-igw
      register: igw_out

    - name: Setup Public Subnet Route Table
      ec2_vpc_route_table:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        tags:
          Name: Vprofile-PubRT
        subnets:
          - "{{ pubsub1_out.subnet.id }}"
          - "{{ pubsub2_out.subnet.id }}"
          - "{{ pubsub3_out.subnet.id }}"
        routes:
          - dest: 0.0.0.0/0
            gateway_id: "{{ igw_out.gateway_id }}"
      register: pubRT_out
```
Commit/push the changes to GitHub. Pull into the Ansible server and run the playbook
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/4b69db37-e986-4bca-ab33-1a6d70a025bf)

Check the AWS console
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/62dcf3d6-d86b-41e7-af3d-d30b4e6ca079)
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/d81f6a79-85a7-4144-b5a7-eb2eda44e0b0)

### Step 7 - NAT Gateway and Route Table
```
- name: Create NAT gateway and allocate new EIP if to it if it does not exist yet in the subnet
      ec2_vpc_nat_gateway:
        subnet_id: "{{ pubsub1_out.subnet.id }}"
        region: "{{region}}"
        state:  "{{state}}"
        wait: yes
        if_exist_do_not_create: yes
      register: NATGW_out

    - name: Setup Private Subnet Route Table
      ec2_vpc_route_table:
        vpc_id: "{{vpcout.vpc.id}}"
        region: "{{region}}"
        tags:
          Name: Vprofile-PrivRT
        subnets:
            - "{{ privsub1_out.subnet.id }}"
            - "{{ privsub2_out.subnet.id }}"
        routes:
          - dest: 0.0.0.0/0
            gateway_id: "{{ NATGW_out.nat_gateway_id }}"
      register: privRT_out

    - debug:
        var: "{{item}}"
      loop:
        - vpcout.vpc.id
        - pubsub1_out.subnet.id
        - pubsub2_out.subnet.id
        - pubsub3_out.subnet.id
        - privsub1_out.subnet.id
        - privsub2_out.subnet.id
        - privsub3_out.subnet.id
        - igw_out.gateway_id
        - pubRT_out.route_table.id
        - NATGW_out.nat_gateway_id
        - privRT_out.route_table.id

    - set_fact:
        vpcid: "{{ vpcout.vpc.id }}"
        pubsub1id: "{{ pubsub1_out.subnet.id }}"
        pubsub2id: "{{ pubsub2_out.subnet.id }}"
        pubsub3id: "{{ pubsub3_out.subnet.id }}"
        privsub1id: "{{ privsub1_out.subnet.id }}"
        privsub2id: "{{ privsub2_out.subnet.id }}"
        privsub3id: "{{ privsub3_out.subnet.id }}"
        igwid: "{{ igw_out.gateway_id }}"
        pubRTid: "{{ pubRT_out.route_table.id }}"
        NATGWid: "{{ NATGW_out.nat_gateway_id }}"
        privRTid: "{{ privRT_out.route_table.id }}"
        cacheable: yes

    - name: Create variables file for vpc output
      copy:
        content: "vpcid: {{ vpcout.vpc.id }}\npubsub1id: {{ pubsub1_out.subnet.id }}\npubsub2id: {{ pubsub2_out.subnet.id }}\npubsub3id: {{ pubsub3_out.subnet.id }}\nprivsub1id: {{ privsub1_out.subnet.id }}\nprivsub2id: {{ privsub2_out.subnet.id }}\nprivsub3id: {{ privsub3_out.subnet.id }}\nigwid: {{ igw_out.gateway_id }}\npubRTid: {{ pubRT_out.route_table.id }}\nNATGWid: {{ NATGW_out.nat_gateway_id }}\nprivRTid: {{ privRT_out.route_table.id }}"

        dest: vars/output_vars
```

### Step 8 - Bastion Host
Create a ```bastion-instance.yml``` file
```
---
- name: Setup Vprofile Bastion Host
  hosts: localhost
  connection: local
  gather_facts: no
  tasks:
    - name: Import VPC setup variable
      include_vars: vars/bastion_setup

    - name: Import VPC setup Variable
      include_vars: vars/output_vars

    - name: Create vprofile ec2 key
      ec2_key:
        name: "{{ keyName }}"
        region: "{{ region }}"
      register: key_out

    - name: Save private key into file bastion-key.pem
      copy:
        content: "{{ key_out.key.private_key }}"
        dest: "./bastion-key.pem"
        mode: 0600
      when: key_out.changed

    - name: Create Sec Grp for Bastion Host
      ec2_group:
        name: Bastion-host-sg
        description: Allow port 22 from everywhere and all port within sg
        vpc_id: "{{ vpcid }}"
        region: "{{ region }}"
        rules:
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: "{{ MYIP }}"
      register: BastionSG_out

    - name: Creating Bastion Host
      ec2:
        key_name: "{{ keyName }}"
        region: "{{ region }}"
        instance_type: "{{ instanceType }}"
        image: "{{ bastion_ami }}"
        wait: yes
        wait_timeout: 300
        instance_tags:
          Name: "Bastion_host"
          Project: Vprofile
          Owner: DevOps Team
        exact_count: 1
        count_tag:
          Name: "Bastion_host"
          Project: Vprofile
          Owner: DevOps Team
        group_id: "{{ BastionSG_out.group_id }}"
        vpc_subnet_id: "{{ pubsub1id }}"
        assign_public_ip: yes
      register: bastionHost_out
```
Commit/push the changes to GitHub. Pull into the Ansible server and run the playbook
![image](https://github.com/RolakeAnifowose/DevOpsProjects/assets/31238382/b0693d0b-e7eb-4d84-809a-30cd9e8c54a9)
