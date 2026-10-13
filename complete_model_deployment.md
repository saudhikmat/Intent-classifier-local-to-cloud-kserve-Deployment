# Deploy Intent Classifier on VMs (AWS VPC + ASG + ALB) — CLI Steps

This simple step-by-step guide shows how to deploy your Intent Classifier model on AWS using EC2 instances in an Auto Scaling Group (ASG) behind an Application Load Balancer (ALB).

### 1. Find a recent Ubuntu AMI for your region

A quick way to locate an official Ubuntu AMI (example uses AWS CLI):

```
aws ec2 describe-images \
--owners 099720109477 \
--filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*" "Name=state,Values=available" \
--query 'Images | sort_by(@, &CreationDate)[-1].ImageId' --output text --region $AWS_REGION
```

This prints the latest Ubuntu 20.04 AMI ID for the region. Adjust the name filter to whichever you need.

### 2. Create a VPC, public subnets (multi-AZ), and Internet Gateway

For an ALB you must provide at least two subnets in different Availability Zones. We'll create two public subnets (one per AZ).

- Create VPC

```
aws ec2 create-vpc --cidr-block 10.10.0.0/16 --query 'Vpc.VpcId' --output text --region $AWS_REGION
```

Save the returned VPC id in VPC_ID.

- Create two public subnets in different AZs (replace ${AWS_REGION}a/b if needed):

```
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.10.1.0/24 --availability-zone ${AWS_REGION}a --query 'Subnet.SubnetId' --output text --region $AWS_REGION
```

save as SUBNET_ID1

```
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.10.2.0/24 --availability-zone ${AWS_REGION}b --query 'Subnet.SubnetId' --output text --region $AWS_REGION
```

save as SUBNET_ID2

- Create and attach an Internet Gateway

```
aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text --region $AWS_REGION
```

save as IGW_ID

```
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID --region $AWS_REGION
```

- Create a route table and route 0.0.0.0/0 to IGW and associate with both subnets

```
aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text --region $AWS_REGION
```

save as RTB_ID

```
aws ec2 create-route --route-table-id $RTB_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID --region $AWS_REGION
```

```
aws ec2 associate-route-table --route-table-id $RTB_ID --subnet-id $SUBNET_ID1 --region $AWS_REGION
aws ec2 associate-route-table --route-table-id $RTB_ID --subnet-id $SUBNET_ID2 --region $AWS_REGION
```

Optional: enable auto-assign public IP for the subnets so instances get public addresses (helpful for debugging):

```
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_ID1 --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_ID2 --map-public-ip-on-launch
```

### 3. Create Security Group for instances

Create a security group that allows traffic from the ALB (on port 80) and SSH from your IP (or nothing for production):

```
aws ec2 create-security-group --group-name intent-sg --description "Allow app and ssh" --vpc-id $VPC_ID --query 'GroupId' --output text --region $AWS_REGION
```

save as SG_ID

Add rules:

```
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0 --region $AWS_REGION
```

Open SSH 

```
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0 --region $AWS_REGION
```

Note: allowing 0.0.0.0/0 to port 80 is simple for beginners but not ideal for production. The more secure pattern is:
Create an ALB security group that allows inbound HTTP/HTTPS from the internet. Configure the instance security group to only allow inbound traffic from the ALB security group's ID.

### 5. Create Launch Template (includes user-data)

A Launch Template defines how EC2 instances are launched (AMI, instance type, key pair, security groups, IAM instance profile, and user-data). Below are two straightforward ways to create a launch template from the CLI.

Prepare your user-data

Create a user-data.sh file with your startup script (refer userdata.sh). Make sure it is executable text.

- Ensure these are exported: AMI_ID, INSTANCE_TYPE, KEY_NAME, LAUNCH_TEMPLATE_NAME

```
USER_DATA=$(base64 -w0 userdata.sh)
aws ec2 create-launch-template \
--launch-template-name "$LAUNCH_TEMPLATE_NAME" \
--version-description "v1" \
--launch-template-data "{\"ImageId\":\"$AMI_ID\",\"InstanceType\":\"$INSTANCE_TYPE\",\"KeyName\":\"$KEY_NAME\",\"SecurityGroupIds\":[\"$SG_ID\"],\"UserData\":\"$USER_DATA\"}" \
--region $AWS_REGION
```

### 6. Create Target Group and Application Load Balancer (ALB)

- Create target group (instances will be registered automatically by ASG)

```
aws elbv2 create-target-group --name mlops-target-group --protocol HTTP --port 80 --vpc-id $VPC_ID --health-check-protocol HTTP --health-check-path /health --matcher HttpCode=200 --region $AWS_REGION
```

Save target group arn from output as TARGET_GROUP_ARN.

- Create an ALB (public) in the two subnets you created. The --subnets parameter must include at least two subnets in different AZs

```
aws elbv2 create-load-balancer --name model-deployment --subnets $SUBNET_ID1 $SUBNET_ID2 --security-groups $SG_ID --scheme internet-facing --type application --region $AWS_REGION
```

save as ALB_ARN

- Create a listener to forward traffic from port 80 to the target group

```
aws elbv2 create-listener --load-balancer-arn $ALB_ARN --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP_ARN --region $AWS_REGION
```

### 7. Create Auto Scaling Group that uses the Launch Template

Create the autoscaling group in both subnets and attach it to the target group so that instances are automatically registered.

```
aws autoscaling create-auto-scaling-group \
--auto-scaling-group-name mlops-autoscaling \
--launch-template LaunchTemplateName=mlops-template,Version=1 \
--min-size 1 --max-size 3 --desired-capacity 1 \
--vpc-zone-identifier "$SUBNET_ID1,$SUBNET_ID2" \
--region $AWS_REGION
```

### 8. Attach the Target Group to ASG 

This ensures instances are automatically registered with the ALB target group:

```
aws autoscaling attach-load-balancer-target-groups \
--auto-scaling-group-name mlops-autoscaling \
--target-group-arns "$TARGET_GROUP_ARN" \
--region $AWS_REGION
```