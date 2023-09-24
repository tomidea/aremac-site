 ## AWS Cloud Solution For 2 Company Websites Using A Reverse Proxy Technology
## SET UP A VIRTUAL PRIVATE NETWORK (VPC)

1. Create a VPC
<img width="1013" alt="VPC" src="https://user-images.githubusercontent.com/51254648/166460718-d5734227-ed10-43b7-8ef6-3cb4aba1500c.png">

3. Create subnets as shown in the architecture
<img width="1008" alt="Subnets" src="https://user-images.githubusercontent.com/51254648/166460722-798c2431-058d-43ea-bb30-423534c48442.png">


5. Create a route table and associate it with public subnets
6. Create a route table and associate it with private subnets

<img width="1019" alt="route tables" src="https://user-images.githubusercontent.com/51254648/166460729-80e3381b-a16f-4ef3-aa67-9e1596087126.png">

8. Create an Internet Gateway

<img width="1009" alt="internet gateway" src="https://user-images.githubusercontent.com/512546<img width="1157" alt="VPC" src="https://github.com/tomidea/aremac-site/assets/51254648/cf01b9bf-3625-4be6-b31f-f0c91cc69267">
48/166460733-26d83341-6652-4ca4-bb49-094089ee22b9.png">

10. Edit a route in public route table, and associate it with the Internet Gateway. (This is what allows a public subnet to be accessible from the Internet)
11. Create 3 Elastic IPs

<img width="1024" alt="elastic IP" src="https://user-images.githubusercontent.com/51254648/166460705-7fd4d93e-0069-4ac5-9b1d-6c2f79e0f7dd.png">

13. Create a Nat Gateway and assign one of the Elastic IPs (*The other 2 will be used by Bastion hosts)

<img width="1002" alt="NAT gateway" src="https://user-images.githubusercontent.com/51254648/166460743-63456edd-5e0b-4ee9-9a90-a9246e6277e8.png">

15. Create a Security Group for:
  - Nginx Servers: Access to Nginx should only be allowed from a Application Load balancer (ALB). At this point, we have not created a load balancer, therefore we will update the rules later. For now, just create it and put some dummy records as a place holder.
  - Bastion Servers: Access to the Bastion servers should be allowed only from workstations that need to SSH into the bastion servers. Hence, you can use your workstation public IP address. To get this information, simply go to your terminal and type curl www.canhazip.com
  - Application Load Balancer: ALB will be available from the Internet
  - Webservers: Access to Webservers should only be allowed from the Nginx servers. Since we do not have the servers created yet, just put some dummy records as a place holder, we will update it later.
  - Data Layer: Access to the Data layer, which is comprised of Amazon Relational Database Service (RDS) and Amazon Elastic File System (EFS) must be carefully desinged – only webservers should be able to connect to RDS, while Nginx and Webservers will have access to EFS Mountpoint.

## Proceed With Compute Resources

#### Set Up Compute Resources for Nginx
1. Create an EC2 Instance based on CentOS Amazon Machine Image (AMI) in any 2 Availability Zones (AZ) in any AWS Region (it is recommended to use the Region that is closest to your customers). Use EC2 instance of T2 family (e.g. t2.micro or similar)

2. Ensure that it has the following software installed:
python
ntp
net-tools
vim
wget
telnet
epel-release
htop

3. Create an AMI out of the EC2 instance
<img width="1040" alt="AMI" src="https://user-images.githubusercontent.com/51254648/166460688-8c2a1d97-2da6-42c0-8f3a-13c39d089f19.png">

Prepare Launch Template For Nginx (One Per Subnet)
- Make use of the AMI to set up a launch template
- Ensure the Instances are launched into a public subnet
- Assign appropriate security group
- Configure Userdata to update yum package repository and install nginx

Configure Target Groups
- Select Instances as the target type
- Ensure the protocol HTTPS on secure TLS port 443
- Ensure that the health check path is /healthstatus
- Register Nginx Instances as targets
- Ensure that health check passes for the target group
<img width="964" alt="target groups" src="https://user-images.githubusercontent.com/51254648/166460708-56c46ffa-9006-4057-bf73-99057a256944.png">
 
Configure Autoscaling For Nginx
- Select the right launch template
- Select the VPC
- Select both public subnets
- Enable Application Load Balancer for the AutoScalingGroup (ASG)
- Select the target group you created before
- Ensure that you have health checks for both EC2 and ALB
- The desired capacity is 2
- Minimum capacity is 2
- Maximum capacity is 4
- Set scale out if CPU utilization reaches 90%
- Ensure there is an SNS topic to send scaling notifications

#### Set Up Compute Resources for Bastion

Provision the EC2 Instances for Bastion
1. Create an EC2 Instance based on CentOS Amazon Machine Image (AMI) per each Availability Zone in the same Region and same AZ where you created Nginx server
2. Ensure that it has the following software installed
python
ntp
net-tools
vim
wget
telnet
epel-release
htop
3. Associate an Elastic IP with each of the Bastion EC2 Instances
4. Create an AMI out of the EC2 instance
5. Prepare Launch Template For Bastion (One per subnet)
- Make use of the AMI to set up a launch template
- Ensure the Instances are launched into a public subnet
- Assign appropriate security group
- Configure Userdata to update yum package repository and install Ansible and git

6. Configure Target Groups
- Select Instances as the target type
- Ensure the protocol is TCP on port 22
- Register Bastion Instances as targets
- Ensure that health check passes for the target group

7. Configure Autoscaling For Bastion
- Select the right launch template
- Select the VPC
- Select both public subnets
- Enable Application Load Balancer for the AutoScalingGroup (ASG)
- Select the target group you created before
- Ensure that you have health checks for both EC2 and ALB
- The desired capacity is 2
- Minimum capacity is 2
- Maximum capacity is 4
- Set scale out if CPU utilization reaches 90%
- Ensure there is an SNS topic to send scaling notifications
<img width="984" alt="auto scaling groups" src="https://user-images.githubusercontent.com/51254648/166460702-00d0d5fb-f5ba-4bab-884b-d204942d12e4.png">

#### Set Up Compute Resources for Webservers
Provision the EC2 Instances for Webservers

Now, you will need to create 2 separate launch templates for both the WordPress and Tooling websites

1. Create an EC2 Instance (Centos) each for WordPress and Tooling websites per Availability Zone (in the same Region).
2. Ensure that it has the following software installed
- python
- ntp
- net-tools
- vim
- wget
- telnet
- epel-release
- htop
- php

3. Create an AMI out of the EC2 instance
Prepare Launch Template For Webservers (One per subnet)
- Make use of the AMI to set up a launch template
- Ensure the Instances are launched into a public subnet
- Assign appropriate security group
- Configure Userdata to update yum package repository and install wordpress (Only required on the WordPress launch template)
<img width="990" alt="launch templates" src="https://user-images.githubusercontent.com/51254648/166460699-42ff12ef-d5a9-42e4-b166-424a8f7f4812.png">

#### TLS Certificates From Amazon Certificate Manager (ACM)
You will need TLS certificates to handle secured connectivity to your Application Load Balancers (ALB).
- Navigate to AWS ACM
- Request a public wildcard certificate for the domain name you registered in Freenom
- Use DNS to validate the domain name
- Tag the resource
<img width="1145" alt="certificate" src="https://user-images.githubusercontent.com/51254648/166460716-aeee2612-376e-475c-b135-d39ffd6dbf8f.png">

### CONFIGURE APPLICATION LOAD BALANCER (ALB)
#### Application Load Balancer To Route Traffic To NGINX
- Create an Internet facing ALB
- Ensure that it listens on HTTPS protocol (TCP port 443)
- Ensure the ALB is created within the appropriate VPC | AZ | Subnets
- Choose the Certificate from ACM
- Select Security Group
- Select Nginx Instances as the target group

### Application Load Balancer To Route Traffic To Web Servers
- Create an Internal ALB
- Ensure that it listens on HTTPS protocol (TCP port 443)
- Ensure the ALB is created within the appropriate VPC | AZ | Subnets
- Choose the Certificate from ACM
- Select Security Group
- Select webserver Instances as the target group
- Ensure that health check passes for the target group

#### NOTE: This process must be repeated for both WordPress and Tooling websites.
<img width="1037" alt="loadbalancers" src="https://user-images.githubusercontent.com/51254648/166460712-7a03271e-10d2-4185-940a-1bd99e44f686.png">

### Setup EFS
- Create an EFS filesystem
- Create an EFS mount target per AZ in the VPC, associate it with both subnets dedicated for data layer
- Associate the Security groups created earlier for data layer.
- Create an EFS access point. (Give it a name and leave all other settings as default)

### Setup RDS
Pre-requisite: Create a KMS key from Key Management Service (KMS) to be used to encrypt the database instance.

To configure RDS, follow steps below:
- Create a subnet group and add 2 private subnets (data Layer)
- Create an RDS Instance for mysql 8.*.*
- To satisfy our architectural diagram, you will need to select either Dev/Test or Production Sample Template. But to minimize AWS cost, you can select the Do not create a standby instance option under Availability & durability sample template (The production template will enable Multi-AZ deployment)
- Configure other settings accordingly (For test purposes, most of the default settings are good to go). In the real world, you will need to size the database appropriately. You will need to get some information about the usage. If it is a highly transactional database that grows at 10GB weekly, you must bear that in mind while configuring the initial storage allocation, storage autoscaling, and maximum storage threshold.
- Configure VPC and security (ensure the database is not available from the Internet)
- Configure backups and retention
- Encrypt the database using the KMS key created earlier
- Enable CloudWatch monitoring and export Error and Slow Query logs (for production, also include Audit)

<img width="964" alt="ACS-database" src="https://user-images.githubusercontent.com/51254648/166460753-395a9795-ade7-490b-bc9c-0f7f2398e8b3.png">

### Configuring DNS with Route53
You need to ensure that the main domain for the WordPress website can be reached, and the subdomain for Tooling website can also be reached using a browser.
- Create other records such as CNAME, alias and A records.

NOTE: You can use either CNAME or alias records to achieve the same thing. But alias record has better functionality because it is a faster to resolve DNS record, and can coexist with other records on that name. 

- Create an alias record for the root domain and direct its traffic to the ALB DNS name.
- Create an alias record for tooling.<yourdomain>.com and direct its traffic to the ALB DNS name.


<img width="1268" alt="tooling" src="https://user-images.githubusercontent.com/51254648/166460747-a253bdaf-11cd-4bcd-81ce-ef6baf74027c.png">
<img width="1279" alt="wordpress" src="https://user-images.githubusercontent.com/51254648/166460762-bac10d9e-99fe-41f4-b4aa-20cafbcdc580.png">
<img width="1278" alt="Screenshot 2022-05-03 at 12 43 40" src="https://user-images.githubusercontent.com/51254648/166462215-06b56151-8539-4457-b84d-66af646f09ba.png">
