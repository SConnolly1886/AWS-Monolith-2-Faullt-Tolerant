# Wordpress Web App - Single Server to an ASG
Now that the user data for the Launch Template has been updated to include EFS, we can go forward and create an auto-scaling group (ASG). From there, an application load balancer (ALB) can be placed in front of the ASG to handle traffic and perform health checks to guarantee no customer interruption.

![ec2asg](https://user-images.githubusercontent.com/62077185/126012540-798a2819-6a9d-4125-a51d-e05552e2e18b.png)

# Create the Application Load Balancer (ALB)
In the EC2 console move to `Load Balancers` and create a load balancer. 
Click `Create` under `Application Load Balancer`  
Under name enter `WORDPRESSALB`  
Ensure `internet-facing` is selected  
ensure `ipv4` selected for `IP Address type`  

Under `Listeners` `HTTP` and `80` should be selected for `Load Balancer Protocol` and `Load Balancer Port`  

Scroll down, under `Availability Zones` 
for VPC ensure `WORDPRESSVPC` is selected  
Check boxes next to `us-east-1a`, `us-east-1b` and `us-east-1c`  
Select `public-subnet-A`, `public-subnet-B` and `public-subnet-C` for each.  

Scroll down and click `Next: Configure Security Settings`  
because we're not using HTTP we can move past this  
Click `Next: Configure Security Groups`  
Check `Select an existing security group` and select `WORDPRESSVPC-SGLoadBalancer` it will have some random at the end and that's ok.  

Click `Next: Configure Routing`  

for `Target Group` choose `New Target Group`  
for Name choose `WORDPRESSALBTG`  
for `Target Type` choose `Instance`  
For `Protocol` choose `HTTP`  
For `Port` choose `80`  
Under `Health checks`
for `Protocol` choose `HTTP`
and for `Path` choose `/`  
Click `Next: Register Targets`  
We won't register any right now, click `Next: Review`  
Click `Create`  

Click on the `WORDPRESSALB` link  
Scroll down and copy the `DNS Name` into your clipboard  

## Create a new Parameter store value with the ALB DNS name
`Create Parameter`  
Name: `/Wordpress/ALBDNSNAME`<br/>
Description: `DNS Name of the Application Load Balancer for Wordpress`<br/>
Value: set the DNS name of the load balancer you copied

### Update the Launch Template for Wordpress with the ALB DNS
Select the `Wordpress` Launch Template, select `Actions` and select `Modify Template (Create New Version)`  
For `Template version description` enter<br/> `App only, uses EFS filesystem defined in /Wordpress/EFSFSID, ALB home added to WP Database`  
Replace the `User Data` with this:

```
#!/bin/bash -xe

ALBDNSNAME=$(aws ssm get-parameters --region us-east-1 --names /Wordpress/ALBDNSNAME --query Parameters[0].Value)
ALBDNSNAME=`echo $ALBDNSNAME | sed -e 's/^"//' -e 's/"$//'`

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

cat >> /home/ec2-user/update_wp_ip.sh<< 'EOF'
#!/bin/bash
source <(php -r 'require("/var/www/html/wp-config.php"); echo("DB_NAME=".DB_NAME."; DB_USER=".DB_USER."; DB_PASSWORD=".DB_PASSWORD."; DB_HOST=".DB_HOST); ')
SQL_COMMAND="mysql -u $DB_USER -h $DB_HOST -p$DB_PASSWORD $DB_NAME -e"
OLD_URL=$(mysql -u $DB_USER -h $DB_HOST -p$DB_PASSWORD $DB_NAME -e 'select option_value from wp_options where option_id = 1;' | grep http)

ALBDNSNAME=$(aws ssm get-parameters --region us-east-1 --names /Wordpress/ALBDNSNAME --query Parameters[0].Value)
ALBDNSNAME=`echo $ALBDNSNAME | sed -e 's/^"//' -e 's/"$//'`

$SQL_COMMAND "UPDATE wp_options SET option_value = replace(option_value, '$OLD_URL', 'http://$ALBDNSNAME') WHERE option_name = 'home' OR option_name = 'siteurl';"
$SQL_COMMAND "UPDATE wp_posts SET guid = replace(guid, '$OLD_URL','http://$ALBDNSNAME');"
$SQL_COMMAND "UPDATE wp_posts SET post_content = replace(post_content, '$OLD_URL', 'http://$ALBDNSNAME');"
$SQL_COMMAND "UPDATE wp_postmeta SET meta_value = replace(meta_value,'$OLD_URL','http://$ALBDNSNAME');"
EOF

chmod 755 /home/ec2-user/update_wp_ip.sh
echo "/home/ec2-user/update_wp_ip.sh" >> /etc/rc.local
/home/ec2-user/update_wp_ip.sh

```

- `Create template version`  
- `View Launch Template`  
- Select the template again (don't click)
- `Actions` and select `Set Default Version`  
- `Template version` select `4`  
- `Set as default version`  


## Create an auto scaling group (without scaling - for now)
This will show the steps to create ans ASG with only 1 instance. It's similar to a monolith EC2 except that if it fails a health check it will be replaced. That's much better! 

So move to the EC2 console. Select `Auto Scaling`. 
- `Auto Scaling Groups`  
- `Create an Auto Scaling Group`  
- `Auto Scaling group name` enter `WORDPRESSASG`  
- `Launch Template` select `Wordpress`  
- `Version` select `Latest`  
- `Next`  
- `Purchase options and instance types` leave the default of `Adhere to launch template` selected.  
- `Network` `VPC` select `WORDPRESSVPC`  
- `Subnets` select `public-subnet-A`, `public-subnet-C` and `public-subnet-C`  


## Integrate ASG with the ALB
Now let's integrate the ASG with the ALB. Load balancers actually work (for EC2) with static instance registrations. An ASG links with a target group, any instances provisioned by the ASG are added to the target group, anything terminated is removed.  

Check the `Enable Load balancing` box  
Ensure `Application Load Balancer or Network Load Balancer` is selected.  
For `Choose a target group for your load balancer` select `WORDPRESSALBTG`  
Under `health Checks - Optional` choose `ELB`  
Scroll down and click `Next`  
- Set the desired, min and max to 1. 
- Tag the instance `Key` enter `Name` and for `Value` enter `Wordpress-ASG` 
- `Create Auto Scaling Group` 

Now that the ASG has been created terminate the EC2 instance. It will relaunch with the correct template (v3) and be known as `Wordpress-ASG` instead of `Wordpress-LT`

## Now to Add Scaling
Select the `Automatic Scaling` Tab in the Auto Scaling Groups. Next a scale in and scale out policy will be added. 

### SCALEOUT when CPU usage on average is above 40%

- `Add Policy`  
- `Policy type` select `Simple scaling`  
- `Scaling Policy name` enter `HIGHCPU`  
- `Create a CloudWatch Alarm`  
- `Select Metric`  
- `EC2'  
- `By Auto Scaling Group`
- Check `WORDPRESSASG CPU Utilization`  
- `Select Metric`  
- select `Greater` and enter `40` in the `than` box and click `next`
- `Remove` next to notification
- `Next`
- `WordpressHIGHCPU` in `Alarm Name`  
- `Next`  
- `Create Alarm`  
- Go back to the AutoScalingGroup tab and click the `Refresh Symbol` next to CloudWatch Alarm  
- Click the dropdown and select `WordpressHIGHCPU`  
- `Take the action` choose `Add` `1` Capacity units  
- `Create`


### SCALEIN when CPU usage on average ie below 40%

- `Add Policy`  
- `Policy type` select `Simple scaling`  
- `Scaling Policy name` enter `LOWCPU`  
- `Create a CloudWatch Alarm`  
- `Select Metric`  
- `EC2`  
- `By Auto Scaling Group`
- `WORDPRESSASG CPU Utilization`  
- `Select Metric`  
- Scroll Down... select `Lower` and enter `40` in the `than` box and click `next`
-  `Remove` next to notification
-  `Next`
- `WordpressLOWCPU` in `Alarm Name`  
- `Next`  
- `Create Alarm`  
- Go to the AutoScalingGroup tab and click the `Refresh Symbol` next to CloudWatch Alarm  
- Click the dropdown and select `WordpressLOWCPU`  
- `Take the action` choose `Remove` `1` Capacity units  
- `Create`

### Re-Adjust ASG Values

- `Details Tab`  
- `Edit`  
- `Desired 1`, Minimum `1` and Maximum `3`  
- `Update`  

### Test Scaling & Self Healing
Now to simulate some load on the Wordpress Instance. Select an instance and SSM into it like before. Type `sudo bash` and `cd`. Now to run a stress test. (Stress was downloaded in the user data).

run `stress -c 2 -v -t 3000`

This will increase the CPU and test the ASG to see if it scales out and in again. Did it work? Great! Now to address the single RDS instance. Let's turn it into an Aurora cluster to make the database highly available and resilient. 


[Step5-RDS-2-Aurora](https://github.com/SConnolly1886/AWS-Monolith-2-Fault-Tolerant/blob/main/Monolith2HA5.md)
