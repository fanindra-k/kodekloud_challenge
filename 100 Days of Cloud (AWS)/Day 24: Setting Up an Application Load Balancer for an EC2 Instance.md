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

## Solution

### 1. Set and Verify the AWS Region

Use the configured AWS region for the remaining AWS CLI commands.

```bash
REGION=$(aws configure get region)
echo "REGION=$REGION"
```

Example output:

```text
REGION=us-east-1
```

The `REGION` variable is required because AWS resources are regional. The load balancer, target group, and EC2 instance must be managed in the same region.

### 2. Define the Load Balancer Settings

Define the names and ports that will be used by the load balancer.

```bash
ALB_NAME=datacenter-alb
TARGET_GROUP_NAME=datacenter-targets
LISTENER_PORT=80
TARGET_PORT=80

echo "ALB_NAME=$ALB_NAME"
echo "TARGET_GROUP_NAME=$TARGET_GROUP_NAME"
echo "LISTENER_PORT=$LISTENER_PORT"
echo "TARGET_PORT=$TARGET_PORT"
```

The variables are required for the following reasons:

- `ALB_NAME` identifies the load balancer and must be unique in the region.
- `TARGET_GROUP_NAME` identifies the group of backend targets that receive traffic.
- `LISTENER_PORT` is the port on which the load balancer accepts client requests.
- `TARGET_PORT` is the port on which the EC2 application receives forwarded requests.

Keeping these values in variables prevents the commands from using inconsistent names or ports.

### 3. Find the EC2 Instance and VPC

Find the instance created in the previous EC2 task and save its identifiers.

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
	--region "$REGION" \
	--filters "Name=tag:Name,Values=datacenter-ec2" "Name=instance-state-name,Values=running" \
	--query 'Reservations[0].Instances[0].InstanceId' \
	--output text)

VPC_ID=$(aws ec2 describe-instances \
	--region "$REGION" \
	--instance-ids "$INSTANCE_ID" \
	--query 'Reservations[0].Instances[0].VpcId' \
	--output text)

SECURITY_GROUP_ID=$(aws ec2 describe-instances \
	--region "$REGION" \
	--instance-ids "$INSTANCE_ID" \
	--query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
	--output text)

echo "INSTANCE_ID=$INSTANCE_ID"
echo "VPC_ID=$VPC_ID"
echo "SECURITY_GROUP_ID=$SECURITY_GROUP_ID"
```

Example output:

```text
INSTANCE_ID=i-xxxxxxxxxxxxxxxxx
VPC_ID=vpc-xxxxxxxxxxxxxxxxx
SECURITY_GROUP_ID=sg-xxxxxxxxxxxxxxxxx
```

The variables are required for the following reasons:

- `INSTANCE_ID` identifies the EC2 instance that will receive traffic.
- `VPC_ID` identifies the virtual network where the target group belongs.
- `SECURITY_GROUP_ID` controls network traffic to the load balancer. The selected security group must allow inbound HTTP traffic on port `80`.

The `describe-instances` commands query AWS instead of requiring these IDs to be copied manually. If `INSTANCE_ID` is `None`, confirm that the instance exists, is running, and has the `datacenter-ec2` name tag.

### 4. Select Two Subnets in Different Availability Zones

An Application Load Balancer requires subnets in at least two Availability Zones. Select two available subnets from the instance's VPC.

```bash
SUBNET_ID_1=$(aws ec2 describe-subnets \
	--region "$REGION" \
	--filters "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" \
	--query 'Subnets | sort_by(@, &AvailabilityZone)[0].SubnetId' \
	--output text)

SUBNET_ID_2=$(aws ec2 describe-subnets \
	--region "$REGION" \
	--filters "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" \
	--query 'Subnets | sort_by(@, &AvailabilityZone)[1].SubnetId' \
	--output text)

AVAILABILITY_ZONE_1=$(aws ec2 describe-subnets \
	--region "$REGION" \
	--subnet-ids "$SUBNET_ID_1" \
	--query 'Subnets[0].AvailabilityZone' \
	--output text)

AVAILABILITY_ZONE_2=$(aws ec2 describe-subnets \
	--region "$REGION" \
	--subnet-ids "$SUBNET_ID_2" \
	--query 'Subnets[0].AvailabilityZone' \
	--output text)

echo "SUBNET_ID_1=$SUBNET_ID_1"
echo "SUBNET_ID_2=$SUBNET_ID_2"
echo "AVAILABILITY_ZONE_1=$AVAILABILITY_ZONE_1"
echo "AVAILABILITY_ZONE_2=$AVAILABILITY_ZONE_2"
```

The variables are required for the following reasons:

- `SUBNET_ID_1` and `SUBNET_ID_2` place the load balancer across two subnets for availability and AWS ALB requirements.
- `AVAILABILITY_ZONE_1` and `AVAILABILITY_ZONE_2` verify that the selected subnets are in different Availability Zones.

The two Availability Zone values must be different. If they match, choose another subnet for `SUBNET_ID_2`; otherwise, ALB creation will fail.

### 5. Allow HTTP Traffic to the EC2 Instance

The target must accept the same HTTP traffic that the load balancer forwards. Add an inbound rule for port `80` when it is not already present.

```bash
aws ec2 authorize-security-group-ingress \
	--region "$REGION" \
	--group-id "$SECURITY_GROUP_ID" \
	--protocol tcp \
	--port "$TARGET_PORT" \
	--cidr 0.0.0.0/0 2>/dev/null || echo "HTTP ingress may already exist"
```

The `authorize-security-group-ingress` command adds an inbound security group rule. The `--protocol`, `--port`, and `--cidr` options define TCP HTTP traffic from any IPv4 client. The fallback message prevents an already-existing rule from stopping the rest of the procedure.

For production workloads, restrict the rule to the load balancer security group instead of `0.0.0.0/0` and use HTTPS where appropriate.

### 6. Create the Target Group

Create a target group for the EC2 instance and configure an HTTP health check.

```bash
TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
	--region "$REGION" \
	--name "$TARGET_GROUP_NAME" \
	--protocol HTTP \
	--port "$TARGET_PORT" \
	--vpc-id "$VPC_ID" \
	--health-check-protocol HTTP \
	--health-check-port traffic-port \
	--health-check-path / \
	--query 'TargetGroups[0].TargetGroupArn' \
	--output text)

echo "TARGET_GROUP_ARN=$TARGET_GROUP_ARN"
```

The `TARGET_GROUP_ARN` variable is required because later commands use the target group's full Amazon Resource Name to register the instance and configure the listener.

The `create-target-group` command creates the backend pool. The health check requests `/` over HTTP. A target is considered healthy only when it responds successfully according to the target group's health-check settings.

### 7. Register the EC2 Instance

Add the running instance to the target group.

```bash
aws elbv2 register-targets \
	--region "$REGION" \
	--target-group-arn "$TARGET_GROUP_ARN" \
	--targets Id="$INSTANCE_ID",Port="$TARGET_PORT"
```

The `register-targets` command connects the EC2 instance to the target group. The `Id` value identifies the instance, while `Port` tells the load balancer where the application is listening.

### 8. Create the Application Load Balancer

Create an internet-facing load balancer in the two selected subnets.

```bash
LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
	--region "$REGION" \
	--name "$ALB_NAME" \
	--scheme internet-facing \
	--type application \
	--security-groups "$SECURITY_GROUP_ID" \
	--subnets "$SUBNET_ID_1" "$SUBNET_ID_2" \
	--query 'LoadBalancers[0].LoadBalancerArn' \
	--output text)

LOAD_BALANCER_DNS_NAME=$(aws elbv2 describe-load-balancers \
	--region "$REGION" \
	--load-balancer-arns "$LOAD_BALANCER_ARN" \
	--query 'LoadBalancers[0].DNSName' \
	--output text)

echo "LOAD_BALANCER_ARN=$LOAD_BALANCER_ARN"
echo "LOAD_BALANCER_DNS_NAME=$LOAD_BALANCER_DNS_NAME"
```

The variables are required for the following reasons:

- `LOAD_BALANCER_ARN` identifies the new ALB for listener and verification commands.
- `LOAD_BALANCER_DNS_NAME` is the client-facing hostname used to test the application.

The `create-load-balancer` command creates an Application Load Balancer that can receive traffic from the internet. The ALB is placed in both subnets and uses the selected security group.

### 9. Create an HTTP Listener

Create a listener that accepts HTTP requests and forwards them to the target group.

```bash
LISTENER_ARN=$(aws elbv2 create-listener \
	--region "$REGION" \
	--load-balancer-arn "$LOAD_BALANCER_ARN" \
	--protocol HTTP \
	--port "$LISTENER_PORT" \
	--default-actions Type=forward,TargetGroupArn="$TARGET_GROUP_ARN" \
	--query 'Listeners[0].ListenerArn' \
	--output text)

echo "LISTENER_ARN=$LISTENER_ARN"
```

The `LISTENER_ARN` variable identifies the listener and is useful for later inspection or modification. The `create-listener` command defines the protocol and port used by clients and sets the default action to forward requests to the target group.

### 10. Verify the Load Balancer and Target Health

Check that the listener exists and that the EC2 target is healthy.

```bash
aws elbv2 describe-listeners \
	--region "$REGION" \
	--load-balancer-arn "$LOAD_BALANCER_ARN" \
	--query 'Listeners[].[ListenerArn,Protocol,Port,DefaultActions[0].TargetGroupArn]' \
	--output table

aws elbv2 describe-target-health \
	--region "$REGION" \
	--target-group-arn "$TARGET_GROUP_ARN" \
	--query 'TargetHealthDescriptions[].[Target.Id,Target.Port,TargetHealth.State,TargetHealth.Reason]' \
	--output table
```

The listener output confirms that the ALB accepts HTTP traffic and forwards it to the intended target group. The target health output should show the instance with a state of `healthy`.

The target can initially show `initial` while AWS performs its first health check. A state of `unhealthy` usually means that the web server is not running, the health-check path is incorrect, the instance is not listening on port `80`, or the security group blocks the traffic.

Test the endpoint after the target becomes healthy:

```bash
curl "http://$LOAD_BALANCER_DNS_NAME"
```

The `curl` command sends an HTTP request to the ALB DNS name. A response from the EC2 web server confirms that the complete path from the client through the listener and target group is working.

## Command Explanations

### `aws ec2 describe-instances`

This command retrieves information about EC2 instances. The filters select the running instance with the `datacenter-ec2` name tag, and the JMESPath queries extract the instance ID, VPC ID, and security group ID needed by later commands.

### `aws ec2 describe-subnets`

This command lists available subnets in the selected VPC. Sorting by Availability Zone makes the selection predictable, while choosing two entries allows the ALB to operate across two Availability Zones.

### `aws ec2 authorize-security-group-ingress`

This command adds an inbound rule to a security group. HTTP traffic must be allowed for the load balancer to reach the application. The rule should be narrowed to trusted sources in a production environment.

### `aws elbv2 create-target-group`

This command creates the backend target group. It defines the protocol, destination port, VPC, and health-check behavior used to determine whether registered targets can receive traffic.

### `aws elbv2 register-targets`

This command registers the EC2 instance with the target group. Registration is required before the load balancer can forward requests to that instance.

### `aws elbv2 create-load-balancer`

This command creates the Application Load Balancer. `--scheme internet-facing` makes it reachable through a public DNS name, `--type application` selects ALB behavior, and `--subnets` places it in the required Availability Zones.

### `aws elbv2 create-listener`

This command creates the ALB listener. A listener receives connections on a configured protocol and port and applies its default action. Here, HTTP requests on port `80` are forwarded to the target group.

### `aws elbv2 describe-target-health`

This command reports the health state of each registered target. It is the key verification step because a target can be registered but still unable to serve traffic.

## Final Note

The workflow follows a reliable ALB setup pattern:

1. Find the EC2 instance, VPC, security group, and subnets.
2. Define and echo every value used by the AWS CLI commands.
3. Allow the required HTTP traffic.
4. Create and configure the target group.
5. Register the EC2 instance as a target.
6. Create the internet-facing Application Load Balancer and listener.
7. Verify the listener, target health, and public DNS endpoint.

When the target reports `healthy` and `curl` returns the application response, the EC2 instance is successfully available through the Application Load Balancer.
