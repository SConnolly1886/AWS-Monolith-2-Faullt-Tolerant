# Monolith Architecture to Highly Available/Fault Tolerant Environment on AWS

![monolith](https://user-images.githubusercontent.com/62077185/126010635-7883a0c2-29d9-4bfb-b57e-c72f4e4e4657.png)


## Why change?
In this example there is a Wordpress Web application that is run using a monolith architecture conisting of a single EC2 instance on AWS. This means that the web service, files and database are all on the EC2 instance. It's a classic setup and something you might find if you've just performed a "lift and shift", migrating an application from your on-premises environment to AWS. 

If anything were to happen to the EC2 instance you would lose access to the web page, the files you've stored and the database attached to the Wordpress environment - in this case it's MariaDB. 

Separating the web, application and database into different tiers is a great way to create high availability and fault tolerance for monolith applications. This is achieved by separating each layer from one another and using multiple availability zones (AZ's). The application and database tiers will be placed in private subnets for increased security.  

### How Can This Be Done?
- create an auto-scaling group of EC2 instances in 3 public subnets behind an Application Load Balancer to distribute traffic for the web tier
- create an EFS shared file system across 3 private subnets for the application tier
- create an Aurora cluster across 3 private subnets for the database tier

![monolith2HA](https://user-images.githubusercontent.com/62077185/126010646-89b388de-6c9d-4fe1-a399-f0d81dcaa47b.png)

A CloudFormation template is provided to create the infrastructure needed to first create a monolith EC2, and then convert it into a highly available, fault tolerant infrastructure. <br/> What will be created:
- vpc
- internet gateway (IGW)
- 3 public, 6 private subnets in 3 AZ's
- routes and route tables for each subnet
- role for EC2 instance to assume
- security groups for ALB, ASG of EC2, EFS, Aurora cluster


## SSM Parameter Store for Database Credentials
Before a monolith Wordpress instance is created, database credentials for Wordpress will first be stored using the SSM Parameter Store. It's a great place to store credentials and passwords. Even though this is an example I didn't include the parameters in the CloudFormation template as it's best practice to store credentials safely (Parameter Store or Secrets Manager).

What needs to be created for the database:
- Database user (DBUser) - String
- Database name (DBName) - String
- Database endpoint (DBEndpoint) - String
- Database password (DBPassword) - Secure String
- Database root password (DBRootPassword) - Secure String


Using the `Parameter Store` create:

#### DBUser  
Name: `/Wordpress/DBUser`<br />
Description: `Wordpress DB User`<br />  
Value: `wordpressuser`<br />  

#### DBName 
Name: `Wordpress/DBName`<br />
Description: `Wordpress DB Name` <br /> 
Value: `wordpressdb` <br /> 

#### DBEndpoint
Name: `/Wordpress/DBEndpoint`<br />
Description: `Wordpress Endpoint` <br /> 
Value: `localhost`   (this value will change later!)

#### DBPassword   
Name: `/Wordpress/DBPassword`<br />
Description: `Wordpress DB Password` <br /> 
Type: `SecureString`  <br />
`KMS Key Source`: `Current Account`  <br />
`KMS Key ID`: default<br />
Value: PASSWORD_OF_YOUR_CHOICE

#### DBRootPassword (self-managed admin)  
Name: `/Wordpress/DBRootPassword`<br />
Description: `Wordpress DB Root Password`  <br />
Type: `SecureString`  <br />
`KMS Key Source`: `Current Account`  <br />
`KMS Key ID`: default<br />
Value: PASSWORD_OF_YOUR_CHOICE 

Great the database credentials have been created! Now to setup Wordpress on the EC2 instance.

## Creating the Monolith Wordpress Instance
The Wordpress Instance requires user data. This user data will:
- set the database credentials as environment variables
- install the necessary files using yum
- start and enable httpd and mariadb service
- set the db root password and edit db wp-config.php file
- fix the permissions
- create Wordpress user, password, and create the DB

Create an EC2 instance, select the newly created VPC, and public subnet A. Choose an AMI that has SSM Agent installed. Preferably `Amazon Linux 2 AMI (HVM)`. 
Name the EC2 instance `Wordpress-Monolith`. Select an existing security group (created by CloudFormation) called `WORDPRESS-SGWordpress`. There is no need for a key pair as SSM will be used over SSH. 

Enter the user data below in the `User Data` box

```
#!/bin/bash -xe

DBPassword=$(aws ssm get-parameters --region us-east-1 --names /Wordpress/DBPassword --with-decryption --query Parameters[0].Value)
DBPassword=`echo $DBPassword | sed -e 's/^"//' -e 's/"$//'`

DBRootPassword=$(aws ssm get-parameters --region us-east-1 --names /Wordpress/DBRootPassword --with-decryption --query Parameters[0].Value)
DBRootPassword=`echo $DBRootPassword | sed -e 's/^"//' -e 's/"$//'`

DBUser=$(aws ssm get-parameters --region us-east-1 --names /Wordpress/DBUser --query Parameters[0].Value)
DBUser=`echo $DBUser | sed -e 's/^"//' -e 's/"$//'`

DBName=$(aws ssm get-parameters --region us-east-1 --names /Wordpress/DBName --query Parameters[0].Value)
DBName=`echo $DBName | sed -e 's/^"//' -e 's/"$//'`

DBEndpoint=$(aws ssm get-parameters --region us-east-1 --names /Wordpress/DBEndpoint --query Parameters[0].Value)
DBEndpoint=`echo $DBEndpoint | sed -e 's/^"//' -e 's/"$//'`

yum -y update
yum -y upgrade

yum install -y mariadb-server httpd wget
amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
amazon-linux-extras install epel -y
yum install stress -y

systemctl enable httpd
systemctl enable mariadb
systemctl start httpd
systemctl start mariadb

mysqladmin -u root password $DBRootPassword

wget http://wordpress.org/latest.tar.gz -P /var/www/html
cd /var/www/html
tar -zxvf latest.tar.gz
cp -rvf wordpress/* .
rm -R wordpress
rm latest.tar.gz

sudo cp ./wp-config-sample.php ./wp-config.php
sed -i "s/'database_name_here'/'$DBName'/g" wp-config.php
sed -i "s/'username_here'/'$DBUser'/g" wp-config.php
sed -i "s/'password_here'/'$DBPassword'/g" wp-config.php
sed -i "s/'localhost'/'$DBEndpoint'/g" wp-config.php

usermod -a -G apache ec2-user   
chown -R ec2-user:apache /var/www
chmod 2775 /var/www
find /var/www -type d -exec chmod 2775 {} \;
find /var/www -type f -exec chmod 0664 {} \;

echo "CREATE DATABASE $DBName;" >> /tmp/db.setup
echo "CREATE USER '$DBUser'@'localhost' IDENTIFIED BY '$DBPassword';" >> /tmp/db.setup
echo "GRANT ALL ON $DBName.* TO '$DBUser'@'localhost';" >> /tmp/db.setup
echo "FLUSH PRIVILEGES;" >> /tmp/db.setup
mysql -u root --password=$DBRootPassword < /tmp/db.setup
rm /tmp/db.setup


```

The monolith Wordpress is set up and ready to go! Test it to make sure by using the public IPv4 of `Wordpress-Monolith` and see if there is a Wordpress welcome page!

### Perform Initial Configuration and make a post

in `Site Title` enter `Wordpress-Example`  
in `Username` enter `admin`
in `Password` feel free to use this or choose your own  
in `Your Email` enter your email address  
- `Install WordPress`
- `Log In`  
- `Username or Email Address` enter `admin`  
- `Password` enter the previously created password  
- `Log In`  

- Select `Posts` in the menu
- Select `Hello World!` 
- `Bulk Actions` and select `Move to Trash`
- `Apply`  

Select `Add New`.  
If you see any popups close them down  
For title: `Wordpress Example Picture(s)!` (or anything you like really)<br/>
Click the `+` under the title, select  `Gallery`<br/>
Click `Upload` <br/>
Select whatever pictures you'd like to use in this example<br/> 
Click `Publish` <br/>
Click `Publish`<br/>
Click `view Post`

This is all working as it should. A monolith Wordpress instance has been successfully created. Now the fun stuff begins! On to the next step. 
