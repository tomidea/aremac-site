## AWS Cloud Solution For 2 Company Websites Using A Reverse Proxy Technology
## SET UP A VIRTUAL PRIVATE NETWORK (VPC)
1. Create a VPC
<img width="1157" alt="VPC" src="https://github.com/tomidea/aremac-site/assets/51254648/b76f76b9-6cad-49a7-840e-a21a756dd381">

3. Create subnets as shown in the architecture
<img width="1143" alt="Subnets" src="https://github.com/tomidea/aremac-site/assets/51254648/d6704ef1-5053-4650-a8e4-11680f7e95a1">

5. Create a route table and associate it with public subnets
6. Create a route table and associate it with private subnets
<img width="1146" alt="ROUTE TABLE" src="https://github.com/tomidea/aremac-site/assets/51254648/c2e1efda-e278-44ef-9ac6-579fe11d8283">


8. Create an Internet Gateway
<img width="1151" alt="internet gw" src="https://github.com/tomidea/aremac-site/assets/51254648/7341cf22-779e-42da-8932-fd8ab800d313">

10. Edit a route in public route table, and associate it with the Internet Gateway. (This is what allows a public subnet to be accessible from the Internet)
11. Create 3 Elastic IPs

13. Create a Nat Gateway and assign one of the Elastic IPs (*The other 2 will be used by Bastion hosts)
<img width="1149" alt="nat-gw" src="https://github.com/tomidea/aremac-site/assets/51254648/e37d09b9-6b30-4bdd-9f80-55774d2d1734">

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
<img width="1171" alt="instances" src="https://github.com/tomidea/aremac-site/assets/51254648/4a875da5-6fc2-495d-ad85-c7baff0d9730">

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
<img width="1089" alt="target groups" src="https://github.com/tomidea/aremac-site/assets/51254648/7e384c2f-93d9-4135-8090-0fcdf3dee663">
 
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
<img width="1095" alt="autoscalling group" src="https://github.com/tomidea/aremac-site/assets/51254648/f9036aca-4335-427b-8e55-ac01304f634b">


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
<img width="1143" alt="launch template" src="https://github.com/tomidea/aremac-site/assets/51254648/27afd243-a979-49bf-a850-4c086d077602">

#### TLS Certificates From Amazon Certificate Manager (ACM)
You will need TLS certificates to handle secured connectivity to your Application Load Balancers (ALB).
- Navigate to AWS ACM
- Request a public wildcard certificate for the domain name you registered in Freenom
- Use DNS to validate the domain name
- Tag the resource

<img width="1076" alt="aremac-cert" src="https://github.com/tomidea/aremac-site/assets/51254648/8d3e5a47-9227-4e9e-8cfd-03d699d9dbb4">

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
<img width="1147" alt="load balancers" src="https://github.com/tomidea/aremac-site/assets/51254648/a2fd98f8-656b-4f66-be09-fbdbdfb39e50">

###Healthy Health Checks
<img width="1110" alt="healthy wordpress" src="https://github.com/tomidea/aremac-site/assets/51254648/1b0613fa-cb8e-4537-8006-4c382db35144">
<img width="1111" alt="healthy tooling" src="https://github.com/tomidea/aremac-site/assets/51254648/7c14644a-6d76-45e7-9828-841692450176">
<img width="1101" alt="healthy nginx" src="https://github.com/tomidea/aremac-site/assets/51254648/0089307c-c562-4c45-82f2-612d6cfb87cb">


### Setup EFS
- Create an EFS filesystem
- Create an EFS mount target per AZ in the VPC, associate it with both subnets dedicated for data layer
- Associate the Security groups created earlier for data layer.
- Create an EFS access point. (Give it a name and leave all other settings as default)
<img width="1108" alt="filesystem" src="https://github.com/tomidea/aremac-site/assets/51254648/754cc399-a7bf-4e92-9048-de565042bac6">

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


### Configuring DNS with Route53
You need to ensure that the main domain for the WordPress website can be reached, and the subdomain for Tooling website can also be reached using a browser.
- Create other records such as CNAME, alias and A records.

NOTE: You can use either CNAME or alias records to achieve the same thing. But alias record has better functionality because it is a faster to resolve DNS record, and can coexist with other records on that name. 

- Create an alias record for the root domain and direct its traffic to the ALB DNS name.
- Create an alias record for tooling.<yourdomain>.com and direct its traffic to the ALB DNS name.
<img width="999" alt="route 53 records" src="https://github.com/tomidea/aremac-site/assets/51254648/6d382b76-9cc1-48fa-acdb-2b3be3d89756">

###Wordpress Site
<img width="1440" alt="wp dashboard" src="https://github.com/tomidea/aremac-site/assets/51254648/932be208-1682-47dc-9333-eae22afe67b6">


###Tooling Site
<img width="1432" alt="tooling page" src="https://github.com/tomidea/aremac-site/assets/51254648/c8401fc3-b605-4d59-82b4-8d3f900b25d0">

