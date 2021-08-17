# First Steps a HA/Fault Tolerant Environment
Now that you have created the wordpress EC2 instance you can go ahead and terminate that instance. Say goodbye to those pictures. Monoliths are easy to setup by if one piece fails, so do all of the others. This is the reason why it's better to create a more resilient infrastructure that can respond to failure in a better way. 

![monolith2HAbegin](https://user-images.githubusercontent.com/62077185/126014019-40fe0e81-c363-4a27-8f1c-00a677864027.png)

What should be done next is to create a `Launch Template`. Use the user data that was used to create the monolith EC2.  

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

Create a Launch Template and use the user data. Use the same security group that was used before and for Instance Profile select `WORDPRESSVPC-WordpressInstanceProfile`

`Create Launch Template`  
`View Launch Templates`


## Launch an EC2 Instance using the Launch Template

Under tags select `Key` to `Name` and `Value` to `Wordpress-LT`.<br />
Scroll to the bottom and click `Launch Instance from template`  

Go through the same steps as before to setup your Wordpress Site. Only this time, don't terminate the instance!

A launch template will make it easier to automate things and going forward but there's still much to be done. The application still isn't highly available and the web, application and database tiers haven't been separated! Also, we are relying on the IPv4 of the EC2 instance. Not good idea as it can change with failures. An ALB should take care of that. But that's several steps away.

## The Database Tier
The next step is to separate the database from the EC2 instance. This will be done by changing over the MariaDB on the EC2 instance into an RDS instance of its own. With that done the Web and Database will be able to scale independently from one another. Now the data can outlive the application if necessary. 

### Creating RDS Subnets Groups

Subnets groups allow RDS to select subnets ranges. In this case it will be the 3 private DB subnets (DBSubnetA,B,C). 
At this point RDS can then decide which subnet to use. 
First create a database subnet group on the RDS console. 

Name: `WordpressRDSSubNetGroup`  
Description: `RDS Subnet Group for Wordpress Example`  
VPC: `WORDPRESSVPC`  

In `Add subnets`:
In `Availability Zones` select `us-east-1a` & `us-east-1b` & `us-east-1c`  
Under `Subnets` check the box next to 

- 10.16.16.0/20 (db-subnet-A)
- 10.16.80.0/20 (db-subnet-B)
- 10.16.144.0/20 (db-subnet-C)

## Create the RDS Instance
Use the RDS subnet group created previously and create an RDS instance. You can select Multi-AZ or single AZ for this is an example.  
Using the RDS Console: 
- `Databases`  
- `Create Database`  
- `Standard Create`  
- `MySql`  
- `Version` select `MySQL 5.6.46` (best Aurora compatibility for snapshot migrations - next steps)  

If you've selected Single AZ you can select `Free Tier` under templates
_this ensures there will be no costs for the database but it can only be for single AZ_

For `Db instance identifier` enter `Wordpress`
For `Master Username` enter the value from here /Wordpress/DBUser/ in Parameter store
For `Master Password` and `Confirm Password` enter the value from /Wordpress/DBPassword/ in Parameter store

Under `DB Instance size`, then `DB instance class`, then `Burstable classes (includes t classes)` make sure db.t2.micro is selected  
Scroll down, under `Connectivity`, `Virtual private cloud (VPC)` select `WORDPRESSVPC`  
Expand `Additional connectivity configuration` 
Ensure under `Subnet group` that `Name`: `WordpressRDSSubNetGroup` is selected  
Make sure `Publicly accessible` is set to `No`  
Under `Existing VPC security groups` add `WORDPRESSVPC-SG-Database` and remove `Default`  
Under `Availability Zone` set `us-east-1a`  
Scroll down and expand `Additional configuration`  
in the `Initial database name` box enter the value from /Wordpress/DBName/
Scroll to the bottom and click `create Database`  

**This is going to take a while which is why is step is done before the others. How long? Expect it to take at the minimum 20 minutes**

Great the RDS instance has been created! Now to migrate from MariaDB to RDS!

# Migrate Wordpress data from MariaDB to RDS
Use SSM Session Manager and connect to the `Wordpress-LT` instance. Type `bash` and `cd` before setting the environment variables.  

### Environment Variables
In this step an export of the SQL database will be made. 

Before you begin, populate the environment variables with values from the SSM Parameter store. 

```
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

```

Great now to take a backup of the DB on the EC2 instance. 

```
mysqldump -h $DBEndpoint -u $DBUser -p$DBPassword $DBName > Wordpress.sql
```
**Normally you shouldn't put the password in the CLI - it's a security risk (ps -aux can see it) This is done to speed things along**

Now that the backup has been taken it can be restored on the newly created RDS instance!

## Restore the Backup into RDS
In the RDS console select the `Wordpressdb` instance and copy the `endpoint`. 
Now go to the Parameter store and delete the parameter `/Wordpress/DBEndpoint` 
Now create a new parameter...<br/>
Name: `/Wordpress/DBEndpoint`  
Value: enter the RDS endpoint you copied  
`Create Parameter`  

Now in the EC2 instance update the DBEndpoint environment variable: 

```
DBEndpoint=$(aws ssm get-parameters --region us-east-1 --names /Wordpress/DBEndpoint --query Parameters[0].Value)
DBEndpoint=`echo $DBEndpoint | sed -e 's/^"//' -e 's/"$//'`
```

Restore the database export into RDS using

```
mysql -h $DBEndpoint -u $DBUser -p$DBPassword $DBName < Wordpress.sql 
```

Great! Now to change the config file for Wordpress to use RDS instead of MariaDB.

### Change the Wordpress config file

This will change `localhost` in the config file for the contents of `$DBEndpoint` which is the RDS instance endpoint you just created. After the RDS instance will be storing the data not the local MariaDB.

```
sed -i "s/'localhost'/'$DBEndpoint'/g" /var/www/html/wp-config.php
```

# Now Stop the MariaDB Service
No need to use MariaDB anymore...
```
sudo systemctl disable mariadb
sudo systemctl stop mariadb
```

Now check that Wordpress is still working by using the IPv4 address like before. Nice! Next steps...

## Editing the Launch Template
Now that the RDS instance is the database for the Wordpress app, let's remove the MariaDB info in the User Data in the launch template created. 


Use the updated data in the `User Data` box

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
systemctl start httpd

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

```

- `Create Template Version`  
- `View Launch Template`  
- Select the template again (don't click)
- `Actions` and select `Set Default Version`  
- `Template version` select `2`  
- `Set as default version`  

The advantage that Launch Templates have over Launch Configurations is the ability to have versions. Great so the DB and Web have been separated but many problems remain.
- web and application are still together on same instance
- single RDS instance
- no ALB

Let's concentrate on separating the web and application tier next: <br/>
[Step3-Create-EFS](https://github.com/SConnolly1886/AWS-Monolith-2-Fault-Tolerant/blob/main/Monolith2HA3.md)
