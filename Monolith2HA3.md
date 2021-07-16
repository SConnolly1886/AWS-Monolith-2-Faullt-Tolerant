# Creating an EFS File System for the Application Tier

<img align="left" src="https://user-images.githubusercontent.com/62077185/126012542-80eb80e8-c785-4eed-bc69-c010caa120a7.png">

# Creating an EFS File System for the Application Tier
The next thing to do is to create an EFS file system for the Wordpress media that is currently stored on the EC2 instance. This area stores any media for posts uploaded when creating the post as well as theme data. Using EFS, you can share the media across all of the instances. The EFS, like the RDS Database, can outlive the Wordpress application. 

### File System Settings
Go the EFS Console and create a new EFS file system. This will take a while too so get comfortable.<br/>
Name the EFS file system: `Wordpress-Content`. Make sure that `Enable Automatic Backups` is enabled. Select the cheapest/standard options and for lifecycle management.`Enable encryption of data at rest` isn't required in this example. 

### Network Settings
Now the EFS `Mount Targets` need to be configured. These are the network interfaces in the VPC that your instances will connect with.  

In the `Virtual Private Cloud (VPC)` dropdown select `WORDPRESSVPC`  
You should see 3 rows.  
Make sure `us-east-1a`, `us-east-1b` & `us-east-1c` are selected in each row.  
In `us-east-1a` row, select `app-subnet-A` in the subnet ID dropdown, and in the security groups dropdown select `WORDPRESSVPC-SGEFS` & remove the default security group  
In `us-east-1b` row, select `app-subnet-B` in the subnet ID dropdown, and in the security groups dropdown select `WORDPRESSVPC-SGEFS` & remove the default security group  
In `us-east-1c` row, select `app-subnet-C` in the subnet ID dropdown, and in the security groups dropdown select `WORDPRESSVPC-SGEFS` & remove the default security group  

Click `next`  
Leave all these options as default and click `next`  
We won't be setting a file system policy so click `Create`  

The file system will start in the `Creating` State and then move to `Available` once it does..  
Click on the file system to enter it and click `Network`  
Scroll down and all the mount points will show as `creating` keep hitting refresh and wait for all 3 to show as available before moving on.  

Note down the `fs-XXXXXXXX` file system ID once visible at the top of this screen, you will need it in the next step.  


## Add an fsid to SSM Parameter store

Now that the file system has created, you need to add another parameter to store the value for the file system ID. This will then be included in an updated user data for a Launch Template.  

`Create Parameter`  
Name: `/Wordpress/EFSFSID`<br/> 
Description: `File System ID for wp-content`<br/>  
Value: set the file system ID `fs-XXXXXXX`   

## Connect the EFS and the EC2 Instance
Use SSM Session Manager and connect to the `Wordpress-LT` instance. Type `bash` and `cd`.

Now install the amazon EFS utilities to allow the instance to connect to EFS

```
sudo yum -y install amazon-efs-utils
```

Great! Now the existing media needs to be migrated into EFS...

```
cd /var/www/html
sudo mv wp-content/ /tmp
sudo mkdir wp-content
```

Now use the EFS file system ID from SSM Parameter store

```
EFSFSID=$(aws ssm get-parameters --region us-east-1 --names /Wordpress/EFSFSID --query Parameters[0].Value)
EFSFSID=`echo $EFSFSID | sed -e 's/^"//' -e 's/"$//'`
```

Next, add a line to /etc/fstab to configure the EFS file system to mount as /var/www/html/wp-content/

```
echo -e "$EFSFSID:/ /var/www/html/wp-content efs _netdev,tls,iam 0 0" >> /etc/fstab
mount -a -t efs defaults
```

Finally, copy the origin content data back in and fix permissions

```
mv /tmp/wp-content/* /var/www/html/wp-content/
chown -R ec2-user:apache /var/www/
reboot
```

Once the EC2 restarts check the IPv4 to make sure the pictures have loaded properly! Awesome! Now we can fix the Launch Template to include the EFS!


## Updating Launch Template with EFS
Now the Launch Template can be changed to automatically mount the EFS while the instance is being provisioned. This will allow the instances to be added to an auto-scaling group and be able to share the same media. 
 
Select the `Wordpress` launch template, select `Actions` and select `Modify Template (Create New Version)`  
for `Template version description` enter `App only, uses EFS filesystem defined in /Wordpress/EFSFSID`  

Paste in the below to edit the user data: 

```
#!/bin/bash -xe

EFSFSID=$(aws ssm get-parameters --region us-east-1 --names /Wordpress/EFSFSID --query Parameters[0].Value)
EFSFSID=`echo $EFSFSID | sed -e 's/^"//' -e 's/"$//'`

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

yum install -y mariadb-server httpd wget amazon-efs-utils
amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
amazon-linux-extras install epel -y
yum install stress -y

systemctl enable httpd
systemctl start httpd

mkdir -p /var/www/html/wp-content
chown -R ec2-user:apache /var/www/
echo -e "$EFSFSID:/ /var/www/html/wp-content efs _netdev,tls,iam 0 0" >> /etc/fstab
mount -a -t efs defaults

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

- `Create template version`  
- `View Launch Template`  
- Select the template again (don't click)
- `Actions` and select `Set Default Version`  
- `Template version` select `3`  
- `Set as default version`  

Great now the EFS is included in the launch template! The only problems now are:
- there's still no ASG and ALB
- there's a single RDS instance

So, let's take care of that in the next steps...
