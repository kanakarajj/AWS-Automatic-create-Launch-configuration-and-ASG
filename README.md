# AWS-Automatic-create-Launch-configuration-and-ASG
#!/bin/bash

export AWS_CONFIG_FILE=/root/.aws/config
export PATH=$PATH:$EC2_HOME/bin

profile="default"
master_instance_id=i-44ad4cc5
timestamp=$(date +%Y%m%d)
#change the keyname in below line
keypair="flinnt"
description="Flinnt-WebApp-Server_Automated_AMI_Backup_"
#change the SG in the below line
security_group="sg-c8190fac"
image_master_prefix="Automated_ami_Flinnt-Web-scaled1_"

# Auto scaling parameters
#change the Launch config name in the below line
config_prefix="Flinnt-PROD-Webapp-LC1"
instance_type="c5.large"
min_size="2"
max_size="15"
#change the Auto Scaling Group Name in the below line
as_group_name="Flinnt-prod-autoscaling-new"

#Do we have any pending or failed AMIs? If so abort.
pending=$(aws ec2 describe-images --owner self --filter '{"Name":"state", "Values":["pending", "failed"]}' | '/usr/bin/jq' '.Images | length')

if [ $pending -gt 0 ]
        then
        echo "Pending or failed AMIs found. Aborting..."
        exit
else
        echo "No Pending or Failed AMI found"
fi


# Create a new image and get the AMI id
echo $master_instance_id
master_ami=$(aws ec2 create-image --instance-id $master_instance_id --no-reboot --name $image_master_prefix$timestamp --description "${description}"timestamp --region ap-southeast-1| '/usr/bin/jq' '.ImageId' | sed s/\"//g)
echo "Master AMI Created"
sleep 4m

echo "Creating Launch configuration ${config_prefix}${timestamp} ..."
#create new launch configuration with new master AMI
aws autoscaling create-launch-configuration --launch-configuration-name $config_prefix$timestamp --instance-type $instance_type --security-groups $security_group --image-id $master_ami --key $keypair --instance-monitoring '{"Enabled": true}' --block-device-mappings "[{\"DeviceName\": \"/dev/xvda\",\"Ebs\":{\"DeleteOnTermination\":true,\"VolumeType\":\"gp2\"}}]" --iam-instance-profile instance-role --user-data file:///backup_scripts/userdata.txt

sleep 2m

echo "updating auto scaling group Flinnt-prod-autoscale"
#Update auto scaling group name

aws autoscaling update-auto-scaling-group --auto-scaling-group-name $as_group_name --launch-configuration-name $config_prefix$timestamp --min-size $min_size --max-size $max_size --availability-zones ap-southeast-1a ap-southeast-1b --health-check-type EC2  --health-check-grace-period 300

echo "save launch config file locally..."
#save the latest launch config to the file

echo "${config_prefix}${timestamp}" > /opt/launchconfigs/"${config_prefix}${timestamp}.txt"

