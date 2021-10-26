# Highload Architecture. Practice 2.
### by Ilia Poeta


VPCID=$(aws ec2 create-vpc --cidr-block 10.10.0.0/18 --no-amazon-provided-ipv6-cidr-block |  grep -oP '(?<="VpcId": ")[^"]*')

SUBNETID1=$(aws ec2 create-subnet --vpc-id="$VPCID" --cidr-block 10.10.0.0/24 --availability-zone us-east-1a |  grep -oP '(?<="SubnetId": ")[^"]*')
SUBNETID2=$(aws ec2 create-subnet --vpc-id="$VPCID" --cidr-block 10.10.1.0/24 --availability-zone us-east-1a |  grep -oP '(?<="SubnetId": ")[^"]*')
SUBNETID3=$(aws ec2 create-subnet --vpc-id="$VPCID" --cidr-block 10.10.2.0/24 --availability-zone us-east-1a |  grep -oP '(?<="SubnetId": ")[^"]*')

IGID=$(aws ec2 create-internet-gateway |  grep -oP '(?<="InternetGatewayId": ")[^"]*')
aws ec2 attach-internet-gateway --internet-gateway-id $IGID --vpc-id $VPCID

KEYPAIR=$(aws ec2 create-key-pair --key-name MyKeyPair --query "KeyMaterial" --output text)
SGID=$(aws ec2 create-security-group --group-name InstancesSG --description "Security group for instances" --vpc-id $VPCID  |  grep -oP '(?<="GroupId": ")[^"]*')

aws ec2 authorize-security-group-ingress --group-id $SGID --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SGID --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SGID --protocol tcp --port 443 --cidr 0.0.0.0/0

INSTANCEID=$(aws ec2 run-instances --image-id ami-02e136e904f3da870 --count 1 --instance-type t2.micro --key-name MyKeyPair --security-group-ids $SGID --subnet-id $SUBNETID1  |  grep -oP '(?<="InstanceId": ")[^"]*')
VOLID=$(aws ec2 create-volume --size 15 --availability-zone us-east-1a  |  grep -oP '(?<="VolumeId": ")[^"]*')

aws ec2 attach-volume --device /dev/sdh --instance-id $INSTANCEID --volume-id $VOLID 

aws autoscaling create-auto-scaling-group --auto-scaling-group-name kma-asg --min-size 1 --max-size 3 --instance-id $INSTANCEID --vpc-zone-identifier "$SUBNETID1,$SUBNETID2,$SUBNETID3"


aws ec2 detach-volume --volume-id $VOLID --instance-id $INSTANCEID
aws ec2 terminate-instances --instance-ids $INSTANCEID
aws ec2 delete-volume --volume-id $VOLID
aws ec2 delete-key-pair --key-name MyKeyPair

aws autoscaling terminate-instance-in-auto-scaling-group --instance-id $INSTANCEID  --should-decrement-desired-capacity
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name kma-asg
aws autoscaling delete-launch-configuration --launch-configuration-name kma-asg

aws ec2 detach-internet-gateway --internet-gateway-id $IGID --vpc-id $VPCID
aws ec2 delete-internet-gateway --internet-gateway-id $IGID

aws ec2 delete-subnet --subnet-id $SUBNETID1
aws ec2 delete-subnet --subnet-id $SUBNETID2
aws ec2 delete-subnet --subnet-id $SUBNETID3

aws ec2 delete-security-group --group-id $SGID

aws ec2 delete-vpc --vpc-id $VPCID
