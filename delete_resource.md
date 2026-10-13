# Delete all the resources to avoid billing

1️⃣ Auto Scaling Group (ASG)

- ASG creates and manages EC2 instances → must go first.
- Detach Target Groups from ASG
- Set Desired, Min, Max = 0
- Delete ASG (force-delete recommended)

This terminates all ASG-created EC2 instances.

2️⃣ EC2 Instances (if any still remain)

- Terminate ALL remaining EC2 instances manually.
- Sometimes ASG leaves “detached” stopped instances — clean those.

3️⃣ Application Load Balancer (ALB)

ALB cannot be deleted while listeners or target groups exist.

- Delete ALB listeners
- Delete ALB
- Wait until ALB state = deleted

4️⃣ Target Groups

After ALB has been removed:

- Deregister any targets (optional)
- Delete the Target Group

5️⃣ Launch Template

Safe to delete now because no ASG references it anymore.

6️⃣ Security Groups

You must delete in this order:

- ALB Security Group
- Instance Security Group

If SG deletion fails, it's usually because ENIs (network interfaces) still exist.
But in your setup, EC2 + ALB ENIs disappear once those resources are deleted.

7️⃣ Subnets

Delete:

- Subnet 1 (AZ-a)
- Subnet 2 (AZ-b)

8️⃣ Internet Gateway (IGW)

Must be detached BEFORE deletion:

- Detach IGW from VPC
- Delete IGW

9️⃣ VPC

Now the VPC will have zero dependencies, so deletion will succeed.

- Delete the VPC.