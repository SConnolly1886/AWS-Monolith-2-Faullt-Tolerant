# From RDS to Aurora
In this last step the RDS instance will be converted to a 3 AZ Aurora cluster to create a highly available and resilient database tier.

![rdsaurora](https://user-images.githubusercontent.com/62077185/126012537-2675e822-0d58-4c36-94bf-643a2e4ac996.png)

## Take a Snapshot of the RDS instance
`Take Snapshot` of the RDS instance and wait until the process finishes. This is really all it takes to convert from RDS to Aurora 

# Setup Aurora

- Select the snapshot you just created by clicking the checkbox.  
- `Actions` and then `Migrate Snapshot`  
- `Migrate to DB Engine` ... select `aurora`  
- Leave `DB Engine Version` as default  
- `DB Instance Class` select `db.t2.small` (for example it's all you need)  
- `DB Instance Identifier` enter `wordpress-aurora`  
- `Virtual Private Cloud (VPC)` select `WORDPRESSVPC`  
- `Subnet group` select `wordpressrdssubnetgroup`  
- `Availability zone` select `No preference`  
- `VPC security groups` select `Choose existing VPC security groups`
- Remove `Default (VPC)` and select `WORDPRESSVPC-SGDatabase` (there will be random after this, thanks ok)  
- Ensure for `Encryption` its set to `Disable encryption`  
- `Migrate`  

So easy to do! From RDS to Aurora with a few clicks! This might take some time but all that's needed now is patience.

# Create Additional Readers

To improve the resiliency of the solution we can create additional replicas. To do that select the `wordpress-aurora-cluster` Aurora cluster  
- `Actions` and then click `Add Reader`  
- instance specifications select the `db.t2.small` instance class  
- `networking and security` set `Availability zone` to `No Preference` and `No` for publicly accessible. 
- `Disable Encryption` under `Encryption`  
- `Settings` and under `DB instance Identifier` enter `wordpress-aurora-reader1`  
- Leave everything else as default
- `Add Reader`  

Next add another reader for 3 AZ resilience  
- `Actions` and then click `Add Reader`  
- instance specifications select the `db.t2.small` instance class  
- `networking and security` set `Availability zone` to `No Preference` and `No` for publicly accessible. 
- `Disable Encryption` under `Encryption`  
- `Settings` and under `DB instance Identifier` enter `wordpress-aurora-reader2`  
- Leave everything else as default
- `Add Reader`  

In a few steps you now have a 3 AZ resilient database cluster.  

# Change the Parameter for DBENDPOINT
Make note of the `Endpoints` section in the `wordpress-aurora-cluster` Aurora Cluster.<br/>Copy down the `endpoint name` for the `Writer` Endpoint.<br/> 
Now move to the SSM Parameter store and delete `/Wordpress/DBEndpoint` parameter.

Great now recreate the parameter:<br/>
`Create Parameter`<br/> 
Name: `/Wordpress/DBEndpoint`<br/> 
Description: `Wordpress Endpoint`<br/>  
Value: enter the Aurora endpoint you just copied   

# Refresh the ASG
move to the auto scaling group
- `start instance refresh`
- set minimum healthy to `0%`
- `start instance refresh`
- wait for a new instance to be created
- test it by opening new instance

![monolith2HA](https://user-images.githubusercontent.com/62077185/126010646-89b388de-6c9d-4fe1-a399-f0d81dcaa47b.png)

# All Done!
From a monolith architecture to a highly available fault tolerant environment? Ahh yes please! Enjoy! Be sure to delete everything you've made if you want to avoid costs. 










