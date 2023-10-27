# The Image Insertion App

The Image Insertion Application is a web application written in PHP that is designed to be deployed on AWS Cloud Infrastructure. This infrastructure includes an Application Load Balancer with an Auto Scaling Group of Elastic Compute Cloud (EC2) Instances and a Relational Database Service (RDS) on a defined VPC. Cloudfront and Route 53 services are also located in front of the architecture to manage traffic securely. Data is stored in EFS. Users can upload images, add descriptions to those images on their own page, and the images are kept in an S3 Bucket.

In this article, you will learn how to install and use the Image Insertion Application step-by-step. You will also learn more about the application's architecture and AWS cloud services.

![web page](images/web-page.jpg)
![failover page](images/failover.jpg)

## Step 1: Create dedicated VPC and whole components

![VPC](images/VPC.jpg)

### VPC
With Amazon Virtual Private Cloud (Amazon VPC), you can launch AWS resources in a logically isolated virtual network that you've defined. This virtual network closely resembles a traditional network that you'd operate in your own data center, with the benefits of using the scalable infrastructure of AWS. Now let's create a VPC and its components for our project.

- Create VPC:
    create a vpc named `cloud-project-vpc` CIDR blok is `10.10.0.0/16` 
    no ipv6 CIDR block
    tenancy: default
- select `cloud-project-vpc` VPC, click `Actions`, `Edit VPC settings` and `enable DNS hostnames` for the `cloud-project`. 


### Subnets
A subnet is a range of IP addresses in your VPC. A subnet must reside in a single Availability Zone. After you add subnets, you can deploy AWS resources in your VPC.

- Create Subnets
    - Create a public subnet named `cloud-project-subnet-public1-us-east-1a` under the vpc cloud-project in AZ us-east-1a with 10.10.10.0/24
    - Create a private subnet named `cloud-project-subnet-private1-us-east-1a` under the vpc cloud-project in AZ us-east-1a with 10.10.11.0/24
    - Create a public subnet named `cloud-project-subnet-public2-us-east-1b` under the vpc cloud-project in AZ us-east-1a with 10.10.20.0/24
    - Create a private subnet named `cloud-project-subnet-private2-us-east-1b` under the vpc cloud-project in AZ us-east-1a with 10.10.21.0/24


- Set `auto-assign IP` up for public subnets. Select each public subnets and click Modify "auto-assign IP settings" and select "Enable auto-assign public IPv4 address" 

Setting up the auto-assign IP feature for public subnets in a VPC is used to automatically assign a public IP address to EC2 instances in those subnets. When this feature is enabled, any EC2 instance that is launched in that subnet will automatically receive a public IP address. This process is necessary for EC2 instances in public subnets to be able to access the internet.

### Internet Gateway

An AWS Internet Gateway (IGW) is a horizontally scaled, highly available, redundant VPC component that allows communication between your VPC and the internet. It is a public-facing gateway that can be attached to your VPC to provide Internet access to your resources.

- Click Internet gateway section on left hand side. Create an internet gateway named `cloud-project-igw` and create.

- ATTACH the internet gateway `cloud-project-igw` to the newly created VPC `cloud-project-vpc`. Go to VPC and select newly created VPC and click action ---> Attach to VPC ---> Select `cloud-project-vpc` VPC 

### Route Table

Route tables are used to determine where network traffic from your subnet or gateway is routed.

- Go to route tables on left hand side. We have already one route table as main route table. Change it's name as `cloud-project-rtb-public` 
- Create a route table and give a name as `cloud-project-rtb-private`.
- Add a rule to `cloud-project-rtb-public` in which destination 0.0.0.0/0 (any network, any host) to target the internet gateway `cloud-project-igw` in order to allow access to the internet.
- Select the private route table, come to the subnet association subsection and add private subnets to this route table. Similarly, we will do it for public route table and public subnets. 
    
### Endpoint

An Amazon Web Services (AWS) endpoint is a logical connection between your VPC and a service or resource that is hosted outside of the VPC. This could be an AWS service, such as Amazon Simple Storage Service (S3), or a resource that is hosted on-premises.

Endpoints allow you to communicate with resources outside of your VPC without having to expose them to the public internet. This can help to improve the security of your VPC. We will create an endpoint to access the s3 buckets we will create in our project without going to the public internet.

- Go to the endpoint section on the left hand menu
- select endpoint ---> click Create endpoint
```text
Name : cloud-project-vpc-s3
Service Category: AWS services
Service  : `com.amazonaws.us-east-1.s3` ---> Gateway
VPC           : `cloud-project-vpc`
Route Table   : private route tables
Policy        : `Full Access`
Create
```


## Step 2: Create Security Groups (ALB ---> EC2 ---> RDS ---> EFS)

A security group , it is a virtual firewall that allows you to specify the protocols, ports and source IP ranges that can reach your samples and the destination IP ranges to which your samples can connect. We will create 5 different security groups for the resources we will create in our project.

![Security Groups](images/sec-grp.jpg)

1. ALB Security Group
```text
Name            : cloud-project-ALB_Sec_Group
Description     : ALB Security Group allows traffic HTTP and HTTPS ports from anywhere 
Inbound Rules
VPC             : cloud-project-vpc
HTTP(80)    ----> anywhere
HTTPS (443) ----> anywhere
```
2. NAT Instance Security Group
```text
Name            : cloud-project-NAT_Sec_Group
Description     : NAT Security Group allows traffic HTTP and HTTPS and SSH ports from anywhere 
Inbound Rules
VPC             : cloud-project-vpc
HTTP(80)    ----> anywhere
HTTPS (443) ----> anywhere
SSH (22)    ----> anywhere
```
3. EC2 Security Groups
```text
Name            : cloud-project-EC2_Sec_Group
Description     : EC2 Security Groups only allows traffic coming from cloud-project-ALB_Sec_Group Security Groups for HTTP and HTTPS ports. In addition, ssh port is allowed from anywhere
VPC             : cloud-project-vpc
Inbound Rules
HTTP(80)    ----> cloud-project-ALB_Sec_Group
HTTPS (443) ----> cloud-project-ALB_Sec_Group
ssh         ----> cloud-project-NAT_Sec_Group
```
4. RDS Security Groups
```text
Name            : cloud-project-RDS_Sec_Group
Description     : EC2 Security Groups only allows traffic coming from cloud-project-EC2_Sec_Group Security Groups for MYSQL/Aurora port. 

VPC             : cloud-project-vpc
Inbound Rules
MYSQL/Aurora(3306)  ----> cloud-project-EC2_Sec_Group
```
5. EFS Security Groups
```text
Name            : cloud-project-EFS_Sec_Group
Description     : EFS Security Groups only allows traffic coming from cloud-project-EC2_Sec_Group Security Groups for NFS port.
VPC             : cloud-project-vpc
Inbound Rules
NFS(2049)    ----> cloud-project-EC2_Sec_Group
```
## Step 3: Create three S3 Buckets and set one of these as static website.

Amazon Simple Storage Service (Amazon S3) is an object storage service that offers industry-leading scalability, data availability, security, and performance.Amazon S3 provides management features so that you can optimize, organize, and configure access to your data to meet your specific business, organizational, and compliance requirements.

In our project, we will create three s3 buckets. in one of them, we will store the images that we will add with our application. in one of them, we will use these images that we store to store them by minimizing them with the lambda function. in one of them, we will present a static web page that will be presented to users in case of failover.

Now let's create our s3 buckets together.

![S3 Buckets](images/S3.jpg)

1. S3 Bucket that images added

- Click Create Bucket
```text
Bucket Name : <name>cloudproject
Region      : N.Virginia
create bucket
```
- Click Create folder
```text
Folder Name : images
create folder
```
2. S3 Bucket that images resized
- Click Create Bucket
```text
Bucket Name : <name>cloudproject-resized
Region      : N.Virginia
Block Public Access settings for this bucket
Block all public access : Unchecked
Other Settings are keep them as are
create bucket
```

Bucket Policy

A bucket policy is a resource-based AWS Identity and Access Management (IAM) policy that you can use to grant access permissions to your bucket and the objects in it. Only the bucket owner can associate a policy with a bucket. The permissions attached to the bucket apply to all of the objects in the bucket that are owned by the bucket owner. Bucket policies are limited to 20 KB in size.
Bucket policies use JSON-based access policy language that is standard across AWS. You can use bucket policies to add or deny permissions for the objects in a bucket. Bucket policies allow or deny requests based on the elements in the policy, including the requester, S3 actions, resources, and aspects or conditions of the request.

In our project, we will give the necessary permissions to the s3 bucket where we will store the images by minimizing them. This policy allows global read access to all objects in the bucket. This means that anyone can download any object in the bucket, whether they have an AWS account or not.

- Click Permissions
```text
Bucket policy Edit Click :
```
```json
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Sid": "AllowPublicRead",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::mucahitcloudproject-resized/*"
        }
    ]
}
```

- Save changes

To configure your bucket to allow cross-origin requests, you create a CORS configuration. The CORS configuration is a document with rules that identify the origins that you will allow to access your bucket, the operations (HTTP methods) that you will support for each origin, and other operation-specific information. You can add up to 100 rules to the configuration.

Again we will define a CORS policy on the same s3 bucket. This CORS policy allows put, post and get methods, all headers and all origins. It means that any resource can be accessed from any website using any method with any header.

- Cross-origin resource sharing (CORS) Edit Click :

```json
[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "PUT",
            "POST",
            "GET"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": []
    }
]
```
- Save changes

3. S3 Bucket for failover scenario

- Click Create Bucket
```text
Bucket Name : mucahitcloud
Region      : N.Virginia
Object Ownership
    - ACLs enabled
        - Bucket owner preferred
Block Public Access settings for this bucket
Block all public access : Unchecked
Please keep other settings as are
```
- create bucket

- Selects created `cloud.<YOUR DNS NAME>` bucket ---> Properties ---> Static website hosting
```text
Static website hosting : Enable
Hosting Type : Host a static website
Index document : index.html
save changes
```
- Select `cloud.<YOUR DNS NAME>` bucket ---> select Upload and upload `index.html` and `sorry.jpg` files from given folder---> Permissions ---> Access control list (ACL) ---> Choose from predefined ACLs ---> Grant public-read access  ( This allows Read for object)

## Step 4: Create IAM Role

AWS Identity and Access Management (IAM) is a web service that helps you securely control access to AWS resources. With IAM, you can centrally manage permissions that control which AWS resources users can access. You use IAM to control who is authenticated (signed in) and authorized (has permissions) to use resources.

We need 2 roles for lambda function to access s3 bucket and modify objects in it and for ec2 instances to access s3 bucket and add images. Let's create them now.

Go to the IAM Consol and create two roles.

![IAM Role2](images/IAM-Role-Lambda.jpg)

![IAM Role1](images/IAM-Role-EC2.jpg)

1. Lambda Role
- Click Create Roles and Create role
```text
Trusted entity type     :   AWS service
Use case                :   Lambda
Permissions policies    :   AWSLambdaExecute
Role name               :   Lambda-S3
Click Create Role
```
2. EC2 Role
- Click Create Roles and Create role
```text
Trusted entity type     :   AWS service
Use case                :   EC2
Permissions policies    :   AWSS3FullAccess
Role name               :   Ec2-S3
Click Create Role
```
## Step 5: Create Lambda-Function

AWS Lambda is a compute service that lets you run code without provisioning or managing servers.
Lambda runs your code on a high-availability compute infrastructure and performs all of the administration of the compute resources, including server and operating system maintenance, capacity provisioning and automatic scaling, and logging. With Lambda, all you need to do is supply your code in one of the language runtimes that Lambda supports.
You organize your code into Lambda functions. The Lambda service runs your function only when needed and scales automatically. You only pay for the compute time that you consume—there is no charge when your code is not running.

In our project, we need a lambda function to transfer the images in the s3 bucket that stores the images that users will add to another s3 bucket by minimizing them.

Go to the Lambda Consol and create a function.

![Lambda-Function](images/Lambda-Function.jpg)

- Click Create funtion
- Author from scratch
```text
Basic information
Function name   : imagesresize
Runtime         : Node.js 14.x
Permissions
Use an existing role
Existing role : Lambda-S3
Click Create function

- Click Code source ---> Upload  from ---> .zip file
    - Upload ImageResize.zip
- Click Configuration ---> Edit General configuration
    - Timeout 15 sec

- Click Add trigger
    - Trigger configuration
    - S3
    - Bucket : mucocloudproject
    - Event types :All object create events
    - Recursive invocation select "I acknowledge that using the same S3 bucket for both input and output is not recommended and that this configuration can cause recursive invocations, increased Lambda usage, and increased costs."
    - Click Add
```
## Step 6: Create NAT Instance in Public Subnet

A NAT instance provides network address translation (NAT). You can use a NAT instance to allow resources in a private subnet to communicate with destinations outside the virtual private cloud (VPC), such as the internet or an on-premises network. The resources in the private subnet can initiate outbound IPv4 traffic to the internet, but they can't receive inbound traffic initiated on the internet.We use NAT instances to allow resources on a private subnet to access the internet while keeping them hidden from the outside world.

To launch NAT instance, go to the EC2 console and click the create button.

![NAT](images/NAT.jpg)

```text
Name : cloud-project-NAT-EC2
write "NAT" into the filter box
select NAT Instance `amzn-ami-vpc-nat-2018.03.0.20211001.0-x86_64-ebs` 
Instance Type: t2.micro
Configure Instance Details  
    - Network : cloud-project-vpc
    - Subnet  : cloud-project-subnet-public1-us-east-1a (Please select one of your Public Subnets)
    - Other features keep them as are
Storage ---> Keep it as is
Configure Security Group
    - Select an existing security group: cloud-project-NAT_Sec_Group
Review and select our own pem key
Click Launch instances


!!!IMPORTANT!!!
- select newly created NAT instance and enable stop source/destination check
- go to private route table and write a rule

Destination : 0.0.0.0/0
Target      : instance ---> Select NAT Instance
Save
```

## Step 7: Create EC2 to be used as template

First we will create an instance where our application will run. we will convert this instance into a template to be able to use it for later.Amazon Elastic Compute Cloud (Amazon EC2) provides on-demand, scalable computing capacity in the Amazon Web Services (AWS) Cloud. Using Amazon EC2 reduces hardware costs so you can develop and deploy applications faster. You can use Amazon EC2 to launch as many or as few virtual servers as you need, configure security and networking, and manage storage. 

Go to the EC2 Consol and create a EC2.

![EC2-temlpate](images/EC2-temlpate.jpg)

- Click Instances ---> Launch an instance
```text
    - Name : cloud-project-EC2-Template
    - Amazon Machine Image (AMI) : Amazon Linux 2
    - Instance type : t2.micro
    - Key pair : muco
    - Edit Network Setting
        - VPC : cloud-project-vpc
        - Subnet : cloud-project-subnet-private1-us-east-1a
        - Security groups : cloud-project-EC2_Sec_Group
    - Advenced details
        - IAM instance profile : Ec2-S3
Click Launch instances
```
## Step 8: Connect to the NAT instance
Connect to the NAT instance. Put your pem file in it. Run chmod 400 xxxx.pem. Then connect Template-EC2 with Template-EC2's private ip. "ssh -i "xxx.pem" ec2-user@private.ip". Run the following commands in order.

```bash
- sudo -i
- yum update -y
- yum install -y httpd mysql
- amazon-linux-extras install -y php7.2
- systemctl start httpd
- systemctl enable httpd
- systemctl restart httpd
```

## Step 9: Create EFS

Amazon Elastic File System (Amazon EFS) provides serverless, fully elastic file storage so that you can share file data without provisioning or managing storage capacity and performance. Amazon EFS is built to scale on demand to petabytes without disrupting applications, growing and shrinking automatically as you add and remove files. Because Amazon EFS has a simple web services interface, you can create and configure file systems quickly and easily. The service manages all the file storage infrastructure for you, meaning that you can avoid the complexity of deploying, patching, and maintaining complex file system configurations.

We will mount an EFS in each instance to ensure that any file in an instance that will be created with auto scaling will also be in other instances.

Go to the EFS Consol and create a EFS.

![EFS](images/EFS.jpg)

- Click Create file system
```text
    - Name : cloud-project-EFS
    - VPC : cloud-project-vpc
    - Click Customize
        - Network
        - Mount targets ---> Availability zone
            - us-east-1a, us-east-1b
        - Mount targets ---> Subnet ID
            - private1, private2
        - Mount targets ---> Security groups
            - cloud-project-EFS_Sec_Group
Other Settings are keep them as are
Click Create
```
## Step 10: Connect to Template-EC2
Run the following commands in order.

```bash
- yum install -y amazon-efs-utils
- cd /var/www
- ls
- sudo mount -t efs -o tls fs-07093c981dd032f4c:/ html
    - Replace fs-07093c981dd032f4c with the name of the efs you created
- cd /etc
- pwd
- nano fstab
    - add the "fs-0c11663b27e872b33:/ /var/www/html efs defaults,_netdev 0 0" command in the first line.Replace fs-07093c981dd032f4c with the name of the efs you created
- cd /var/www/html
- mkdir images
- chmod 777 images/
- ls -l
```

## Step 11: Create RDS

Amazon Relational Database Service (Amazon RDS) is a web service that makes it easier to set up, operate, and scale a relational database in the AWS Cloud.
We will create a mysql database with RDS service for our project. 

![RDS](images/RDS.jpg)

First we create a subnet group for our custom VPC. Click `subnet Groups` on the left hand menu and click `create DB Subnet Group` 
```text
Name               : cloud-project-rds-subnet
Description        : aws cloud project RDS Subnet Group
VPC                : cloud-project-vpc
Add Subnets
Availability Zones : Select 2 AZ in cloud-project-vpc
Subnets            : Select 2 Private Subnets in these subnets
```
- Go to the RDS console and click `create database` button
```text
Choose a database creation method : Standart Create
Engine Options  : Mysql
Version         : 8.0.32
Templates       : Free Tier
Settings        : 
    - DB instance identifier : projedbinstance
    - Master username        : projemaster
    - Password               : master1234 
DB Instance Class            : Burstable classes (includes t classes) ---> db.t2.micro
Storage                      : 20 GB and enable autoscaling(up to 40GB)
Connectivity:
    VPC                      : cloud-project-vpc
    Subnet Group             : cloud-project-rds-subnet
    Public Access            : No 
    VPC Security Groups      : Choose existing ---> cloud-project-RDS_Sec_Group
    Availability Zone        : No preference
    Additional Configuration : Database port ---> 3306
Database authentication ---> Password authentication
Additional Configuration:
    - Initial Database Name  : proje
    - Backup ---> Enable automatic backups
    - Backup retention period ---> 7 days
    - Select Backup Window ---> Select 03:00 (am) Duration 1 hour
    - Maintance window : Select window    ---> 04:00(am) Duration:1 hour
    - Log Exports: Audit log and Error log
create instance
```

## Step 12: Connect to Template-EC2
Run the following commands in order.

```bash
- mysql -h projedbinstance.cktjpclzgyei.us-east-1.rds.amazonaws.com -P 3306 -u projemaster -pmaster1234
    - Replace projedbinstance.cktjpclzgyei.us-east-1.rds.amazonaws.com with the Endpoint of the RDS you created
- USE proje;
- CREATE TABLE visitors (name VARCHAR(30), email VARCHAR(30), phone VARCHAR(30), photo VARCHAR(30));
- SHOW Tables;
- exit
```

## Step 13: Update add.php and view.php files
Open add.php and view.php files in Project/Php with text editor.

1. Update add.php
```text
    - $servername = "projedbinstance.cqub0g199gzf.eu-west-1.rds.amazonaws.com";
    - Replace projedbinstance.cktjpclzgyei.us-east-1.rds.amazonaws.com with the Endpoint of the RDS you created
```
2. Update view.php
```text
    - $servername = "projedbinstance.cqub0g199gzf.eu-west-1.rds.amazonaws.com";
    - Replace projedbinstance.cktjpclzgyei.us-east-1.rds.amazonaws.com with the Endpoint of the RDS you created
    - Echo "<img src=https://s3-eu-west-1.amazonaws.com/PROJECTNAME-resized/resized-images/"
```
## Step 14: Connect to Template-EC2
Run the following commands in order.

```bash
- cd /var/www/html
- nano index.html
    - Copy the contents of the index.html file in Project/Php here and save
- nano add.php
    - Copy the contents of the add.php file in Project/Php here and save
- nano view.php
    - Copy the contents of the view.php file in Project/Php here and save
- nano s3.sh
    - #!/bin/bash
      aws s3 sync /var/www/html/images s3://mucahitcloudproject/images
    - Replace mucahitcloudproject with the name of the S3 Bucket you created
- crontab -e
    - click "i" 
    - */2 * * * * /var/www/html/s3.sh
    - click "esc"
    - :x
```

## Step 15: Create AMI from Template EC2

An Amazon Machine Image (AMI) is a supported and maintained image provided by AWS that provides the information required to launch an instance. You must specify an AMI when you launch an instance. You can launch multiple instances from a single AMI when you require multiple instances with the same configuration. That's what we're going to do.

Go to the EC2 Consol and create a AMI from Template EC2.

![AMI](images/AMI.jpg)

1. Volumes
```text
    - Attached Instances ---> i-00f335e3ca2e38113 (cloud-project-EC2-Template)
    - Click Action ---> Create snapshot
    - Description : cloud-project-snapshot
    - Tags Key ---> Name, Value ---> cloud-project-snapshot
    - Click Create snapshot
```
2. Snapshot
```text
    - Click Action ---> Create image from snapshot
    - Image name : cloud-project-AMI
    - Description : AMI created for the project
    - Virtualization type : hardware-assisted virtualization
```
## Step 16: Create Target Group and Application Load Balancer

Elastic Load Balancing automatically distributes your incoming traffic across multiple targets, such as EC2 instances, containers, and IP addresses, in one or more Availability Zones. 

Target groups route requests to individual registered targets, such as EC2 instances, using the protocol and port number that you specify. You can register a target with multiple target groups.  

Go to the EC2 Consol and create Target Group and Application Load Balancer.

![Target-Grp](images/Target-Grp.jpg)

![Load-Balancer](images/Load-Balancer.jpg)

1. Target groups
```text
    - Click Create target groups
    - target type : Instances
    - Target group name : cloud-project-TG
    - VPC : cloud-project-vpc
    - Click Create target group
```
2. Application Load Balancer
```text
    - Click Create
    - Load balancer name : cloud-project-ALB
    - VPC : cloud-project-vpc
    - Mappings : us-east-1a, us-east-1b
    - Security groups : cloud-project-ALB_Sec_Group
    - Listener : HTTP:80
    - Default action : cloud-project-TG
    - Click Create load balancer
```

## Step 17: Create Launch Templates and Auto Scaling groups

An AWS Launch Template is a template that you can use to create an EC2 instance. An initialization template defines the properties of an EC2 instance, such as AMI, instance type, key pair, security groups, and other parameters. Using initialization templates makes it faster and easier to create EC2 instances. We will create a launch template using the AMI we created earlier.

Amazon EC2 Auto Scaling helps ensure that you have the right number of Amazon EC2 instances that can handle the load of your application. Amazon EC2 Auto Scaling can start or end instances as demand in your application increases or decreases.

Translated with www.DeepL.com/Translator (free version)
Go to the EC2 Consol and create Launch Templates and Auto Scaling groups.

![Launch-Templates](images/Launch-template.jpg)

![ASG](images/ASG.jpg)

1. Launch Templates
```text
    - Click Create launch Templates
    - Name : cloud-project-LT
    - Application and OS Images (Amazon Machine Image)
        - My AMIs : cloud-project-AMI
    - Instance type : t2.micro
    - IAM instance profile : Ec2-S3
    - Assign a security group : cloud-project-EC2_Sec_Group
    - Key pair : muco
    - Firewall (security groups): cloud-project-EC2_Sec_Group
    - Advanced details
        - IAM instance profile : Ec2-S3
    - Click Create Launch Templates
```
2. Auto Scaling groups
```text
    - Click Create Auto Scaling group
    - Name : cloud-project-ASG
    - Launch Template : cloud-project-LT
    - VPC : cloud-project-vpc
    - Availability Zones and subnets : cloud-project-subnet-private2-us-east-1a, cloud-project-subnet-private2-us-east-1b
    - Click Attach to an existing load balancer
    - Click Choose from your load balancer target groups
    - Existing load balancer target groups : cloud-project-TG
    - Select Turn on Elastic Load Balancing health checks
    - Desired capacity : 2
    - Minimum capacity : 2
    - Maximum capacity : 4
    - Click Target tracking scaling policy
    - Scaling policy name : Cloud Project Target Tracking Policy
    - Metric type : Average CPU utilization
    - Target value : 50
    - Tags Key ---> Name, Value ---> ASG-cloud-project-EC2
    - Click Create Auto Scaling group
```
## Step 18: Create AWS Certificate Manager (ACM) for Cloudfront

AWS Certificate Manager (ACM) handles the complexity of creating, storing, and renewing public and private SSL/TLS X.509 certificates and keys that protect your AWS websites and applications. You can provision certificates for your integrated AWS services by issuing them directly with ACM or by importing third-party certificates into the ACM management system. We will create a certificate for the domain name we will use in our application. 

Go to the AWS Certificate Manager (ACM) menu and click Request

![ACM](images/Certificate.jpg)

```text
Certificate type : Request a public certificate
Domain names     : <YOUR DNS NAME>
Add another name of this certificate : *.<YOUR DNS NAME>
Click Request
```

## Step 19: Create Cloudfront in front of ALB

Amazon CloudFront is a web service that accelerates the delivery of your static and dynamic web content such as .html, .css, .js and image files to your users. CloudFront delivers your content through a worldwide network of data centers called edge locations. When a user requests content that you deliver with CloudFront, the request is routed to the edge location that provides the lowest latency (time delay) so that the content is delivered with the best possible performance. For this, we will create a CloudFront. 

Go to the cloudfront menu and click start

![ASG](images/CloudFront.jpg)

- Origin Settings
```text
Origin Domain Name          : cloud-project-ALB
Origin Path                 : Leave empty (this means, define for root '/')
Protocol                    : HTTP only
HTTP Port                   : 80
Enable Origin Shield        : No
Additional settings         : Keep it as is
```
Default Cache Behavior Settings
```text
Path pattern                                : Default (*)
Compress objects automatically              : Yes
Viewer Protocol Policy                      : HTTP and HTTPS
Allowed HTTP Methods                        : GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
Cache key and origin requests
- Cache policy and origin request policy (recommended)
    - Cache policy : Caching Disable
```
- Web Application Firewall (WAF)
```text
Do not enable security protections
```
- Settings
```text
Price Class                             : Use all edge locations (best performance)
Alternate Domain Names                  : mucahitdevops.com
SSL Certificate                         : Custom SSL Certificate (example.com) ---> Select your certificate creared before
Other stuff                             : Keep them as are                  
```
Click Create distribution

## Step 20: Create Route 53 with Failover settings

Amazon Route 53 is a highly available and scalable Domain Name System (DNS) web service. In our project, we will manage our domain name with Route53 service. 

AWS Route 53 health checks monitor the health and performance of your web applications, web servers, and other resources. Route 53 health checks send requests to your resources at regular intervals that you specify. 

Come to the Route53 console and select Health checks on the left hand menu. Click create health check
Configure health check

![Route53](images/Route53.jpg)

```text
Name                : cloud project health check
What to monitor     : Endpoint
Specify endpoint by : Domain Name
Protocol            : HTTP
Domain Name         : Write cloudfront domain name
Port                : 80
Path                : leave it blank
Other stuff         : Keep them as are
```
- Click Hosted zones on the left hand menu

- Click your Hosted zone        : <YOUR DNS NAME>

- Click Create Record

---> First we'll create a primary record for cloudfront
```text
Record name             : cloud.<YOUR DNS NAME>
Record Type             : A - Routes traffic to an IPv4 address and some AWS resources
Alias                   : enable
Route traffic to        : Alias to Cloudfront distribution
Chose distribution

Routing policy          : Failover
Failover record type    : Primary
Health check ID         : cloud project health check
Record ID               : Cloudfront as Primary Record
----------------------------------------------------------------

---> Second we'll create secondary record for S3

Failover another record to add to your DNS ---> Define failover record

Record name             : cloud.<YOUR DNS NAME>
Record Type             : A - Routes traffic to an IPv4 address and some AWS resources
Alias                   : enable
Route traffic to        : Alias to S3 website endpoint
                          - Select Region
                          - Your created bucket name mucahitcloud ---> Select it
Failover record type    : Secondary
Health check            : No health check
Record ID               : S3 Bucket for Secondary record type
```

- click create records

## Resources
- [AWS EC2](https://docs.aws.amazon.com/ec2/)
- [AWS IAM](https://docs.aws.amazon.com/iam/)
- [AWS Lambda](https://docs.aws.amazon.com/lambda/)
- [AWS S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [AWS RDS](https://docs.aws.amazon.com/rds/)
- [AWS EFS](https://docs.aws.amazon.com/efs/)
- [AWS VPC](https://docs.aws.amazon.com/vpc/)
- [AWS Route 53](https://docs.aws.amazon.com/route53/)
- [AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)
- [AWS CloudFront](https://docs.aws.amazon.com/cloudfront/)
- [AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/index.html)