# Intent Classifier Model - Local Deployment Demo

This project demonstrates how to train a lightweight text-classification model for intent detection and expose it through a local Flask API. It is useful as a small ML demo, a backend service for a chatbot, or a base for a simple local GUI or web app.

## Overview

The workflow is intentionally simple:

- A small training script builds a text classifier from example phrases.
- The trained pipeline is saved as a serialized model artifact.
- A Flask app loads the artifact and serves predictions through a `/predict` endpoint.
- The API can be integrated into a front-end or GUI layer for user interaction.

## Project structure

- `app.py` - Flask application that exposes prediction endpoints.
- `model/train.py` - Trains the intent classifier and saves the model artifact.
- `model/intent_model.py` - Loads the saved model and performs inference.
- `model/artifacts/intent_model.pkl` - Serialized trained model.
- `requirements.txt` - Python dependencies for the project.
- `wsgi.py` - WSGI entry point for deployment frameworks.

## Step 1: Create a virtual environment

From the project root, create and activate a Python virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

## Step 2: Install dependencies

```bash
pip install -r requirements.txt
```

This installs Flask, scikit-learn, joblib, pytest, and gunicorn.

## Step 3: Train the model

Run the training script:

```bash
python model/train.py
```

This creates the model file:

```text
model/artifacts/intent_model.pkl
```

The training script uses a simple `CountVectorizer` + `MultinomialNB` pipeline on a few example phrases such as greeting, complaint, question, and praise.

## Step 4: Start the Flask API

Launch the app locally:

```bash
python app.py
```

The service will run on:

```text
http://127.0.0.1:6000
```

## Step 5: Check service health

Use this endpoint to confirm the server is live:

```bash
curl http://127.0.0.1:6000/health
```

Example response:

```json
{"status":"ok"}
```

## Step 6: Send a prediction request

Example request:

```bash
curl -X POST http://127.0.0.1:6000/predict \
  -H "Content-Type: application/json" \
  -d '{"text":"I want to cancel my subscription"}'
```

Example response:

```json
{
  "intent": "complaint"
}
```

The actual model also exposes probability values internally through the model pipeline and can be extended to return them in the API response if needed.

## Optional: Run with WSGI

If you want a production-style server entry point:

```bash
gunicorn --bind 0.0.0.0:6000 wsgi:app
```

## Deploy the model on AWS (VPC, ALB, and Auto Scaling)

This walkthrough deploys the Flask API and trained intent model to Ubuntu EC2 instances in a new VPC. An internet-facing Application Load Balancer (ALB) accepts HTTP traffic and forwards it to instances managed by an Auto Scaling Group (ASG). The ALB health check uses `/health`; prediction requests use `POST /predict`.

```text
Client -> ALB (public subnets, port 80) -> EC2 instances (port 80/Nginx)
                                               -> Gunicorn (127.0.0.1:6000)
```

The EC2 instances are reachable on port 80 only from the ALB security group. SSH is not opened by this guide. HTTP is suitable for a short-lived demo; use HTTPS with an ACM certificate and an HTTPS listener before exposing a production service.

### Prerequisites

- AWS CLI v2 installed and configured with credentials allowed to create VPC, EC2, ELB, and Auto Scaling resources.
- An AWS Region selected below, and two available Availability Zones in that Region.
- The repository's `userdata.sh` available in the current project directory. EC2 runs this script at first boot to install the app, train the model, and start Gunicorn and Nginx.

Check that credentials work, set the region, and list usable AZs:

```bash
aws sts get-caller-identity
export AWS_REGION=us-east-1
aws ec2 describe-availability-zones \
  --region "$AWS_REGION" \
  --filters Name=state,Values=available \
  --query 'AvailabilityZones[].ZoneName' \
  --output table
```

Set `AZ1` and `AZ2` to two different AZ names returned above. Do not assume the names end in `a` and `b`; available names vary by account and region. Keep using the same terminal so the exported resource IDs remain available.

```bash
export AZ1=us-east-1a
export AZ2=us-east-1b
export VPC_CIDR=10.10.0.0/16
```

Replace the example region and AZs with values available in your account.

### 1. Create a VPC and two public subnets

Create the VPC and enable DNS features required for AWS resource names and the load balancer:

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block "$VPC_CIDR" \
  --query 'Vpc.VpcId' --output text --region "$AWS_REGION")

aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" \
  --enable-dns-support '{"Value":true}' --region "$AWS_REGION"
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" \
  --enable-dns-hostnames '{"Value":true}' --region "$AWS_REGION"
```

Create one subnet in each AZ. These CIDR ranges are within the example VPC range and do not overlap:

```bash
export SUBNET_ID1=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" --cidr-block 10.10.1.0/24 \
  --availability-zone "$AZ1" \
  --query 'Subnet.SubnetId' --output text --region "$AWS_REGION")
export SUBNET_ID2=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" --cidr-block 10.10.2.0/24 \
  --availability-zone "$AZ2" \
  --query 'Subnet.SubnetId' --output text --region "$AWS_REGION")

aws ec2 modify-subnet-attribute --subnet-id "$SUBNET_ID1" \
  --map-public-ip-on-launch --region "$AWS_REGION"
aws ec2 modify-subnet-attribute --subnet-id "$SUBNET_ID2" \
  --map-public-ip-on-launch --region "$AWS_REGION"
```

Create and attach an Internet Gateway, then create a route table with a default route to it. Associate both subnets with that route table to make them public:

```bash
export IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' --output text --region "$AWS_REGION")
aws ec2 attach-internet-gateway --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" --region "$AWS_REGION"

export ROUTE_TABLE_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --query 'RouteTable.RouteTableId' --output text --region "$AWS_REGION")
aws ec2 create-route --route-table-id "$ROUTE_TABLE_ID" \
  --destination-cidr-block 0.0.0.0/0 --gateway-id "$IGW_ID" --region "$AWS_REGION"
aws ec2 associate-route-table --route-table-id "$ROUTE_TABLE_ID" \
  --subnet-id "$SUBNET_ID1" --region "$AWS_REGION"
aws ec2 associate-route-table --route-table-id "$ROUTE_TABLE_ID" \
  --subnet-id "$SUBNET_ID2" --region "$AWS_REGION"
```

### 2. Create separate ALB and EC2 security groups

The ALB security group allows public HTTP. The instance security group allows HTTP only from the ALB security group; there is no direct public rule for the EC2 instances.

```bash
export ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name intent-alb-sg --description "Public HTTP to intent ALB" \
  --vpc-id "$VPC_ID" --query 'GroupId' --output text --region "$AWS_REGION")
aws ec2 authorize-security-group-ingress --group-id "$ALB_SG_ID" \
  --protocol tcp --port 80 --cidr 0.0.0.0/0 --region "$AWS_REGION"

export INSTANCE_SG_ID=$(aws ec2 create-security-group \
  --group-name intent-instance-sg --description "HTTP from intent ALB only" \
  --vpc-id "$VPC_ID" --query 'GroupId' --output text --region "$AWS_REGION")
aws ec2 authorize-security-group-ingress --group-id "$INSTANCE_SG_ID" \
  --protocol tcp --port 80 --source-group "$ALB_SG_ID" --region "$AWS_REGION"
```

The default outbound rules let instances download packages and clone the repository during bootstrap.

### 3. Find an Ubuntu AMI and create a Launch Template

Look up the latest Canonical Ubuntu 22.04 x86_64 AMI in the selected region:

```bash
export AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate)[-1].ImageId' \
  --output text --region "$AWS_REGION")
```

The Launch Template sets the instance image, size, security group, and startup script. The script is Base64 encoded as required by EC2:

```bash
export LAUNCH_TEMPLATE_NAME=intent-model-template
export INSTANCE_TYPE=t3.micro
USER_DATA=$(base64 < userdata.sh | tr -d '\n')

aws ec2 create-launch-template \
  --launch-template-name "$LAUNCH_TEMPLATE_NAME" \
  --version-description "Intent model app" \
  --launch-template-data "{\"ImageId\":\"$AMI_ID\",\"InstanceType\":\"$INSTANCE_TYPE\",\"SecurityGroupIds\":[\"$INSTANCE_SG_ID\"],\"UserData\":\"$USER_DATA\"}" \
  --region "$AWS_REGION"
```

On first boot, `userdata.sh` clones this repository, installs `requirements.txt`, trains and saves `model/artifacts/intent_model.pkl`, starts Gunicorn as a systemd service on localhost port 6000, and configures Nginx on port 80. Bootstrap may take several minutes; EC2 user-data runs only when an instance is first launched.

### 4. Create the ALB target group and load balancer

The target group checks `/health` on port 80. Create the ALB across the two public subnets and attach the ALB-only security group:

```bash
export TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
  --name intent-model-targets --protocol HTTP --port 80 \
  --target-type instance --vpc-id "$VPC_ID" \
  --health-check-protocol HTTP --health-check-path /health \
  --matcher HttpCode=200 \
  --query 'TargetGroups[0].TargetGroupArn' --output text --region "$AWS_REGION")

export ALB_ARN=$(aws elbv2 create-load-balancer \
  --name intent-model-alb --scheme internet-facing --type application \
  --security-groups "$ALB_SG_ID" --subnets "$SUBNET_ID1" "$SUBNET_ID2" \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text --region "$AWS_REGION")
aws elbv2 wait load-balancer-available \
  --load-balancer-arns "$ALB_ARN" --region "$AWS_REGION"

aws elbv2 create-listener \
  --load-balancer-arn "$ALB_ARN" --protocol HTTP --port 80 \
  --default-actions "Type=forward,TargetGroupArn=$TARGET_GROUP_ARN" \
  --region "$AWS_REGION"
```

### 5. Create the Auto Scaling Group

Create an ASG spanning both subnets and associate it with the target group. The ASG launches one EC2 instance and registers it with the ALB automatically:

```bash
export ASG_NAME=intent-model-asg
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "$ASG_NAME" \
  --launch-template "LaunchTemplateName=$LAUNCH_TEMPLATE_NAME,Version=1" \
  --min-size 1 --max-size 3 --desired-capacity 1 \
  --vpc-zone-identifier "$SUBNET_ID1,$SUBNET_ID2" \
  --target-group-arns "$TARGET_GROUP_ARN" \
  --health-check-type ELB --health-check-grace-period 300 \
  --region "$AWS_REGION"
```

### 6. Wait for health checks and test the deployed model

Wait for EC2 bootstrap to finish. Then inspect target health; continue when the instance state is `healthy`:

```bash
aws elbv2 describe-target-health \
  --target-group-arn "$TARGET_GROUP_ARN" --region "$AWS_REGION"
```

Get the ALB's DNS name and check the API:

```bash
export ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns "$ALB_ARN" \
  --query 'LoadBalancers[0].DNSName' --output text --region "$AWS_REGION")

curl "http://$ALB_DNS/health"
curl -X POST "http://$ALB_DNS/predict" \
  -H "Content-Type: application/json" \
  -d '{"text":"I want to cancel my subscription"}'
```

The health response should be `{"status":"ok"}`. A prediction response should contain an `intent`, for example `{"intent":"complaint"}`.

### Troubleshooting and cleanup

- If targets stay unhealthy, inspect EC2 system and instance status checks, confirm the instance security group allows port 80 from the ALB security group, and review `/var/log/cloud-init-output.log` on the instance. The Nginx health route must stay `/health` so it reaches the Flask health endpoint.
- If the instance fails during launch, confirm the selected AMI exists in `AWS_REGION`, both subnets are in different AZs, and the VPC route table includes the Internet Gateway route.
- AWS charges for running resources. When finished, remove resources in dependency order: delete the ASG first (terminating its instances), then the ALB/listener and target group, launch template, security groups, subnets, Internet Gateway, and finally the VPC. See [delete_resource.md](./delete_resource.md).

## Troubleshooting

- If you see a `ModuleNotFoundError`, ensure your virtual environment is activated and dependencies are installed.
- If the model file is missing, run `python model/train.py` before starting the API.
- If port 6000 is already in use, change the port in `app.py` and restart the service.

## GUI architecture

A graphical interface can be added on top of this API using a browser-based frontend that sends text input to the Flask backend and displays the returned intent. See [GUI_ARCHITECTURE.md](./GUI_ARCHITECTURE.md) for the UI and interaction design.
