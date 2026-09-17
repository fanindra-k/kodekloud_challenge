# Setting Up an Application Load Balancer for an EC2 Instance

## Problem Statement

The Nautilus DevOps team needs to expose the existing `datacenter-ec2` instance through an internet-facing Application Load Balancer. The load balancer must forward HTTP requests to the EC2 instance and use a health check to confirm that the instance is available.

As a member of the Nautilus DevOps Team, your task is to perform the following:

- Find the existing `datacenter-ec2` instance and its networking details.
- Create an internet-facing Application Load Balancer named `datacenter-alb`.
- Create a target group named `datacenter-targets` that uses HTTP on port `80`.
- Register the `datacenter-ec2` instance in the target group.
- Create an HTTP listener that forwards requests to the target group.
- Verify that the target is healthy and retrieve the load balancer DNS name.

The commands below use the AWS CLI and assume that the EC2 instance is running a web server on port `80` and that its security group allows HTTP traffic.

## Prerequisites & Architecture Overview

- ALB Name: `datacenter-alb`
- Target Group Name: `datacenter-tg` (Port `80`)
- ALB Security Group: `datacenter-sg` (Inbound Port `80` from `0.0.0.0/0`)
- EC2 Instance: `datacenter-ec2` (Traffic forwarded to Port `80`)
- EC2 Security Group: Must allow inbound traffic on Port `80` from `datacenter-sg`

In this architecture, the Application Load Balancer receives internet traffic on port `80`, forwards the request to the target group, and the target group sends it to the EC2 instance running the web application. The EC2 instance must allow traffic from the ALB security group so the request path is complete.

## Solution

### Method 1: AWS Management Console

#### Step 1: Create Security Group for ALB (`datacenter-sg`)

Go to the EC2 Console → Security Groups (under Network & Security).

Click Create security group.

Set:

- Security group name: `datacenter-sg`
- Description: `Security group for datacenter ALB`
- VPC: Select the same VPC where `datacenter-ec2` is deployed.

Under Inbound rules, click Add rule:

- Type: `HTTP`
- Port range: `80`
- Source: `Anywhere-IPv4 (0.0.0.0/0)`

Leave Outbound rules to default (All traffic).

Click Create security group.

#### Step 2: Update the EC2 Instance Security Group

To ensure the ALB can send traffic to Nginx on `datacenter-ec2`:

1. In the EC2 Console, go to Instances and select `datacenter-ec2`.
2. Under the Security tab, click the attached security group.
3. Click Actions → Edit inbound rules.
4. Click Add rule.
5. Set:
   - Type: `HTTP`
   - Port range: `80`
   - Source: Select Custom and search/select `datacenter-sg`
6. Click Save rules.

#### Step 3: Create Target Group (`datacenter-tg`)

Go to Target Groups (under Load Balancing).

Click Create target group.

Configure target group settings:

- Target type: `Instances`
- Target group name: `datacenter-tg`
- Protocol: `HTTP`
- Port: `80`
- IP address type: `IPv4`
- VPC: Select the VPC of `datacenter-ec2`
- Health checks: Protocol `HTTP`, Path `/`

Click Next.

Under Register targets:

- Find and select `datacenter-ec2`.
- Ensure port is `80`.
- Click Include as pending below.
- Click Create target group.

#### Step 4: Create Application Load Balancer (`datacenter-alb`)

Go to Load Balancers (under Load Balancing).

Click Create load balancer, then under Application Load Balancer, click Create.

Basic configuration:

- Load balancer name: `datacenter-alb`
- Scheme: `Internet-facing`
- IP address type: `IPv4`

Network mapping:

- VPC: Select the VPC where your instance is running.
- Mappings: Select at least two Availability Zones and their corresponding public subnets.

Security groups:

- Remove any default security group.
- Select `datacenter-sg`.

Listeners and routing:

- Protocol: `HTTP`
- Port: `80`
- Default action: Select Forward to → `datacenter-tg`

Review the configuration and click Create load balancer.

#### Step 5: Verification

Wait 2–3 minutes for the ALB state to change from Provisioning to Active.

Then:

1. Go to Target Groups → select `datacenter-tg` → click the Targets tab.
2. Confirm the health status of `datacenter-ec2` is Healthy.
3. Go to Load Balancers → copy the DNS name of `datacenter-alb`.

Run in your terminal or browser:

```bash
curl http://<ALB-DNS-Name>
```

You should see the sample Nginx welcome page.

### Method 2: AWS CLI

#### 1. Identify the VPC ID and Instance ID

```bash
VPC_ID=$(aws ec2 describe-vpcs --query "Vpcs[0].VpcId" --output text)
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId" --output text)
```

#### 2. Create a Security Group for the ALB

```bash
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name datacenter-sg \
  --description "Security group for datacenter-alb" \
  --vpc-id $VPC_ID \
  --query "GroupId" --output text)
```

Allow port `80` from the public internet:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

#### 3. Allow Traffic from the ALB Security Group to the EC2 Instance

```bash
INSTANCE_SG_ID=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $INSTANCE_SG_ID \
  --protocol tcp \
  --port 80 \
  --source-group $ALB_SG_ID
```

#### 4. Create the Target Group

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name datacenter-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --query "TargetGroups[0].TargetGroupArn" --output text)
```

Register the EC2 instance:

```bash
aws elbv2 register-targets \
  --target-group-arn $TG_ARN \
  --targets Id=$INSTANCE_ID,Port=80
```

#### 5. Fetch Two Public Subnets for the ALB

```bash
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0:2].SubnetId" --output text)
```

#### 6. Create the Application Load Balancer

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name datacenter-alb \
  --subnets $SUBNET_IDS \
  --security-groups $ALB_SG_ID \
  --scheme internet-facing \
  --type application \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)
```

#### 7. Create the HTTP Listener

```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

#### 8. Verify the Target Health and Retrieve the ALB DNS Name

```bash
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```

```bash
aws elbv2 describe-load-balancers \
  --names datacenter-alb \
  --query "LoadBalancers[0].DNSName" \
  --output text
```

Test the endpoint:

```bash
curl http://$(aws elbv2 describe-load-balancers \
  --names datacenter-alb \
  --query "LoadBalancers[0].DNSName" \
  --output text)
```

You should see the sample Nginx welcome page.

## Command Explanations

### `aws ec2 describe-vpcs`

This command retrieves the VPC details for the instance environment.

- `aws`: starts the AWS CLI.
- `ec2`: indicates EC2 commands.
- `describe-vpcs`: lists the VPCs in the region.
- `--query "Vpcs[0].VpcId"`: fetches the first VPC ID from the list.
- `--output text`: prints the value in a simple text format.

This helps identify the correct VPC where the EC2 instance and ALB should be created.

### `aws ec2 describe-instances`

This command retrieves details about EC2 instances in your AWS account.

- `describe-instances`: fetches instance metadata such as instance IDs, VPC IDs, security groups, and state.
- `--filters`: narrows the search to the instance named `datacenter-ec2` and only includes the running instance.
- `--query`: extracts just the instance ID from the response.
- `--output text`: provides the output in a clean text format for scripts.

This step is critical because the load balancer must know which EC2 instance to target.

### `aws ec2 create-security-group`

This command creates a security group for the load balancer.

- `create-security-group`: creates a new security group in the selected VPC.
- `--group-name datacenter-sg`: sets the security group name.
- `--description`: adds a description of the purpose.
- `--vpc-id $VPC_ID`: associates the SG with the same VPC as the EC2 instance.
- `--query "GroupId" --output text`: returns only the group ID.

This security group allows the ALB to accept public HTTP traffic.

### `aws ec2 authorize-security-group-ingress`

This command adds inbound rules to a security group.

- `authorize-security-group-ingress`: opens a port on the selected security group.
- `--group-id`: identifies which security group to update.
- `--protocol tcp`: uses the TCP protocol.
- `--port 80`: allows HTTP traffic.
- `--cidr 0.0.0.0/0`: allows access from all IPv4 addresses.
- `--source-group $ALB_SG_ID`: allows the ALB security group to reach the EC2 instance.

This step ensures both the public ALB and the private instance can communicate over port `80`.

### `aws elbv2 create-target-group`

This command creates the target group used by the ALB.

- `elbv2`: indicates Elastic Load Balancing v2 commands.
- `create-target-group`: creates a new target group.
- `--name datacenter-tg`: sets the target group name.
- `--protocol HTTP`: uses HTTP for health checks and traffic forwarding.
- `--port 80`: specifies the backend port.
- `--vpc-id $VPC_ID`: sets the target group in the same VPC.
- `--target-type instance`: registers EC2 instances as targets.

This target group is the destination where the ALB sends requests.

### `aws elbv2 register-targets`

This command registers the EC2 instance with the target group.

- `register-targets`: adds a target to the group.
- `--target-group-arn $TG_ARN`: selects the target group.
- `--targets Id=$INSTANCE_ID,Port=80`: registers the instance and the port it listens on.

Without this step, the load balancer has no EC2 instance to forward traffic to.

### `aws elbv2 create-load-balancer`

This command creates the Application Load Balancer.

- `create-load-balancer`: creates the actual ALB.
- `--name datacenter-alb`: sets the ALB name.
- `--subnets $SUBNET_IDS`: deploys it in public subnets.
- `--security-groups $ALB_SG_ID`: attaches the ALB security group.
- `--scheme internet-facing`: makes the ALB publicly reachable.
- `--type application`: creates an ALB.

This is the main AWS resource that receives client requests.

### `aws elbv2 create-listener`

This command creates the listener for HTTP traffic.

- `create-listener`: defines how incoming requests are handled.
- `--load-balancer-arn $ALB_ARN`: identifies the ALB.
- `--protocol HTTP`: enables HTTP traffic.
- `--port 80`: listens on port `80`.
- `--default-actions Type=forward,TargetGroupArn=$TG_ARN`: forwards requests to the target group.

This completes the routing flow from the public ALB to the EC2 instance.

### `aws elbv2 describe-target-health`

This command checks whether the EC2 instance is healthy.

- `describe-target-health`: shows the status of each target in the group.
- `--target-group-arn $TG_ARN`: selects the target group to inspect.

The target should eventually report Healthy before the ALB is considered ready.

### `curl`

This command is used to test the final endpoint.

```bash
curl http://$(aws elbv2 describe-load-balancers \
  --names datacenter-alb \
  --query "LoadBalancers[0].DNSName" \
  --output text)
```

- `curl`: sends an HTTP request.
- `http://...`: connects to the ALB DNS name.
- `$(...)`: resolves the DNS name dynamically from AWS.

If configured correctly, the command returns the Nginx default page from the EC2 instance.

## Final Note

The workflow follows a secure and reliable pattern:

1. Create a dedicated ALB security group.
2. Allow public HTTP access to the ALB.
3. Permit inbound traffic from the ALB security group on the EC2 instance.
4. Create a target group and register the instance.
5. Create the ALB and attach an HTTP listener.
6. Verify the target health and test the endpoint with the ALB DNS name.

This ensures the EC2 instance is reachable through the ALB while preserving proper security and health checks.
```bash
VPC_ID=$(aws ec2 describe-vpcs --query "Vpcs[0].VpcId" --output text)
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].InstanceId" --output text)
```

If you want to confirm which subnets are available in the same VPC, you can also run:

```bash
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0:2].SubnetId" --output text)
```

This gives you the network context needed to create the load balancer in the correct VPC and subnets.

### 2. Create a Security Group for the ALB

Create a security group that allows inbound HTTP traffic from the internet.

```bash
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name datacenter-sg \
  --description "Security group for datacenter-alb" \
  --vpc-id $VPC_ID \
  --query "GroupId" --output text)
```

Allow port `80` from anywhere so the ALB can be reachable publicly:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

### 3. Allow Traffic from the ALB to the EC2 Instance

The EC2 instance must accept HTTP traffic coming from the ALB security group.

```bash
INSTANCE_SG_ID=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $INSTANCE_SG_ID \
  --protocol tcp \
  --port 80 \
  --source-group $ALB_SG_ID
```

This allows the ALB to route requests to the backend EC2 instance on port `80` without exposing the instance to the public directly.

### 4. Create the Target Group

Create a target group named `datacenter-targets` in the same VPC and register the EC2 instance as a target.

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name datacenter-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --query "TargetGroups[0].TargetGroupArn" --output text)
```

```bash
aws elbv2 register-targets \
  --target-group-arn $TG_ARN \
  --targets Id=$INSTANCE_ID,Port=80
```

The target group tells AWS where to send incoming traffic and performs health checks on the registered instance.

### 5. Create the Application Load Balancer

Create the internet-facing ALB and attach it to the selected public subnets.

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name datacenter-alb \
  --subnets $SUBNET_IDS \
  --security-groups $ALB_SG_ID \
  --scheme internet-facing \
  --type application \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)
```

This creates the load balancer in the VPC and makes it available to the public through its DNS name.

### 6. Create the HTTP Listener

Create a listener that listens on port `80` and forwards traffic to the target group.

```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### 7. Verify the Health of the Target and Retrieve the ALB DNS Name

Check if the target is healthy before testing the application.

```bash
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```

Get the ALB DNS name:

```bash
aws elbv2 describe-load-balancers \
  --names datacenter-alb \
  --query "LoadBalancers[0].DNSName" \
  --output text
```

Now test the service from a browser or terminal:

```bash
curl http://$(aws elbv2 describe-load-balancers \
  --names datacenter-alb \
  --query "LoadBalancers[0].DNSName" \
  --output text)
```

If the web server is running correctly, the load balancer should return the default page from the EC2 instance.

## Command Explanations

### `aws ec2 describe-instances`

This command retrieves details about EC2 instances in your AWS account.

- `aws`: starts the AWS CLI.
- `ec2`: indicates that the command is for Amazon EC2.
- `describe-instances`: fetches instance metadata such as instance IDs, VPC IDs, security groups, and state.
- `--filters`: narrows the search to the instance named `datacenter-ec2` and only includes running instances.
- `--query`: extracts just the instance ID from the response.
- `--output text`: provides the output in a clean text format for shell scripts.

This step is important because the load balancer must know which EC2 instance to target and which VPC it belongs to.

### `aws ec2 create-security-group`

This command creates a new security group for the ALB.

- `create-security-group`: creates a new security group in the specified VPC.
- `--group-name datacenter-sg`: sets the security group name.
- `--description`: adds a human-readable description of the group.
- `--vpc-id $VPC_ID`: associates the security group with the same VPC as the EC2 instance.
- `--query "GroupId" --output text`: returns only the group ID so it can be reused in later commands.

This step ensures the ALB can accept incoming HTTP traffic from the internet.

### `aws ec2 authorize-security-group-ingress`

This command adds an inbound rule to a security group.

- `authorize-security-group-ingress`: opens a port on the selected security group.
- `--group-id`: identifies which security group to update.
- `--protocol tcp`: uses the TCP protocol.
- `--port 80`: allows HTTP traffic.
- `--cidr 0.0.0.0/0`: permits access from any IPv4 address.
- `--source-group $ALB_SG_ID`: allows traffic from the ALB security group itself.

This is required to allow the public ALB to receive requests and to allow ALB-to-instance communication.

### `aws elbv2 create-target-group`

This command creates the target group used by the ALB.

- `elbv2`: indicates Elastic Load Balancing v2 commands.
- `create-target-group`: creates a new target group.
- `--name datacenter-targets`: names the target group.
- `--protocol HTTP`: uses HTTP for health checks and traffic forwarding.
- `--port 80`: sends requests to port `80` on the instance.
- `--vpc-id $VPC_ID`: places the target group in the same VPC as the EC2 instance.
- `--target-type instance`: registers EC2 instances as targets.

This target group is the backend destination for the ALB.

### `aws elbv2 register-targets`

This command registers the EC2 instance with the target group.

- `register-targets`: adds a backend target to the target group.
- `--target-group-arn $TG_ARN`: selects the target group.
- `--targets Id=$INSTANCE_ID,Port=80`: registers the instance and tells the load balancer to send traffic to port `80` on it.

Without this step, the ALB has no backend to forward traffic to.

### `aws elbv2 create-load-balancer`

This command creates the Application Load Balancer itself.

- `create-load-balancer`: provisions the load balancer resource.
- `--name datacenter-alb`: assigns the ALB a name.
- `--subnets $SUBNET_IDS`: places the ALB in public subnets.
- `--security-groups $ALB_SG_ID`: binds the ALB to the security group that allows HTTP.
- `--scheme internet-facing`: makes the ALB publicly reachable.
- `--type application`: creates an ALB instead of a Network Load Balancer.

This is the main infrastructure resource that receives external traffic and forwards it to the instance.

### `aws elbv2 create-listener`

This command creates an ALB listener that receives incoming HTTP traffic.

- `create-listener`: defines how the ALB should handle requests on a port.
- `--load-balancer-arn $ALB_ARN`: selects the ALB created earlier.
- `--protocol HTTP`: listens for HTTP requests.
- `--port 80`: accepts traffic on port `80`.
- `--default-actions Type=forward,TargetGroupArn=$TG_ARN`: forwards all requests to the target group.

This completes the routing path from a public URL to the EC2 instance.

### `aws elbv2 describe-target-health`

This command checks whether the EC2 instance is healthy and ready to receive traffic.

- `describe-target-health`: shows the health status of all targets in a target group.
- `--target-group-arn $TG_ARN`: specifies the target group to inspect.

The target should show an `initial` or `healthy` state before you rely on the ALB in production or for testing.

### `curl`

This command is used to test the final endpoint.

```bash
curl http://$(aws elbv2 describe-load-balancers \
  --names datacenter-alb \
  --query "LoadBalancers[0].DNSName" \
  --output text)
```

- `curl`: sends an HTTP request.
- `http://...`: connects to the ALB's DNS name.
- `$(...)`: resolves the ALB DNS name dynamically from AWS.

If the ALB is working correctly, this command returns the instance's web content, which confirms the HTTP route is functioning.

## Final Note

The workflow follows a secure and reliable pattern:

1. Find the EC2 instance and its VPC.
2. Create a dedicated security group for the ALB.
3. Allow traffic from the ALB to the EC2 instance over port `80`.
4. Create a target group and register the instance.
5. Create the ALB and attach the listener.
6. Verify target health and test the endpoint with the ALB DNS name.

This ensures the instance is reachable through a public ALB while still keeping the backend service in a controlled, health-checked setup.