# Day 22: Configuring Secure SSH Access to an EC2 Instance

The Nautilus DevOps team needs to create an EC2 instance that can be accessed securely from the `aws-client` host.

The instance must:

- Use the `t2.micro` instance type.
- Have the name tag `datacenter-ec2`.
- Be reachable from `aws-client`.
- Allow passwordless SSH access for the root user using `/root/.ssh/id_rsa`.

## 1. Check the AWS Region

Use the configured AWS region for the remaining AWS CLI commands.

```bash
aws configure get region
```

Example output:

```text
us-east-1
```

## 2. Create the Initial EC2 Key Pair

Create an AWS EC2 key pair for the initial login to the new instance. Store the private key securely and restrict its permissions.

```bash
aws ec2 create-key-pair \
  --key-name datacenter-ec2-key \
  --query 'KeyMaterial' \
  --output text > datacenter-ec2-key.pem

chmod 400 datacenter-ec2-key.pem
```

## 3. Find an Amazon Linux AMI

Find the latest available Amazon Linux 2023 AMI in the configured region.

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-*" \
    "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate)[-1].[ImageId,Name]' \
  --output table
```

Example output:

```text
-----------------------------------------
| DescribeImages                        |
+----------------------+----------------+
| ami-xxxxxxxxxxxxxxx  | al2023-...     |
+----------------------+----------------+
```

Save the returned AMI ID for the next step.

## 4. Launch the EC2 Instance

Replace the placeholder values with a subnet ID and security group ID from the target VPC. The security group must allow SSH traffic from `aws-client`.

```bash
aws ec2 run-instances \
  --image-id ami-xxxxxxxxxxxxxxx \
  --instance-type t2.micro \
  --key-name datacenter-ec2-key \
  --security-group-ids sg-xxxxxxxxxxxxxxx \
  --subnet-id subnet-xxxxxxxxxxxxxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datacenter-ec2}]'
```

The important instance settings are:

```text
Instance type: t2.micro
Name tag:      datacenter-ec2
```

## 5. Check and Create the SSH Key on aws-client

Run this section on `aws-client`, not inside the EC2 instance. The script checks whether the private key already exists before creating anything, so an existing key is never overwritten.

```bash
mkdir -p /root/.ssh
chmod 700 /root/.ssh

if [ -f /root/.ssh/id_rsa ]; then
  echo "Private key already exists: /root/.ssh/id_rsa"
else
  echo "Private key not found. Creating /root/.ssh/id_rsa"
  ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa -N ''
fi

if [ -f /root/.ssh/id_rsa.pub ]; then
  echo "Public key already exists: /root/.ssh/id_rsa.pub"
else
  echo "Public key not found. Deriving it from the private key"
  ssh-keygen -y -f /root/.ssh/id_rsa > /root/.ssh/id_rsa.pub
fi

chmod 600 /root/.ssh/id_rsa
chmod 644 /root/.ssh/id_rsa.pub
ls -l /root/.ssh/id_rsa /root/.ssh/id_rsa.pub
```

The permissions are intentional:

- `chmod 700 /root/.ssh` allows only root to access the SSH directory.
- `chmod 600 /root/.ssh/id_rsa` protects the private key so only root can read or modify it. SSH may reject a private key that is accessible to other users.
- `chmod 644 /root/.ssh/id_rsa.pub` allows the public key to be read because it is not secret. Only root can modify it.

Expected output includes both files:

```text
-rw------- 1 root root 3389 ... /root/.ssh/id_rsa
-rw-r--r-- 1 root root  743 ... /root/.ssh/id_rsa.pub
```

## 6. Find the Instance Public IP Address

Wait until the instance is running, then retrieve its public IP address.

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,PublicIpAddress]' \
  --output table
```

Example output:

```text
------------------------------------------------
| InstanceId          | State   | PublicIp      |
------------------------------------------------
| i-xxxxxxxxxxxx      | running | 54.xx.xx.xx   |
------------------------------------------------
```

## 7. Connect Using the Initial Key Pair

The following command is run on `aws-client`. It enters the EC2 instance using the temporary AWS key pair. Amazon Linux normally uses the `ec2-user` account for this initial login.

```bash
ssh -i datacenter-ec2-key.pem ec2-user@PUBLIC_IP_ADDRESS
```

Replace `PUBLIC_IP_ADDRESS` with the public IP returned in the previous step.

After the connection succeeds, the commands in the next section run inside the EC2 instance. Your shell prompt will change from the `aws-client` prompt to an EC2 prompt similar to `[ec2-user@ip-10-xxx-xxx-xxx ~]$`.

## 8. Prepare Root SSH Access on the EC2 Instance

Run these commands inside the EC2 instance to prepare the root user's SSH directory.

```bash
sudo mkdir -p /root/.ssh
sudo chmod 700 /root/.ssh
sudo touch /root/.ssh/authorized_keys
sudo chmod 600 /root/.ssh/authorized_keys
```

The permissions protect the SSH authorization files:

- `chmod 700 /root/.ssh` prevents other users from entering root's SSH directory.
- `chmod 600 /root/.ssh/authorized_keys` ensures only root can read or change the list of keys allowed to log in as root.

Exit the EC2 instance to return to `aws-client` before reading the new public key:

```bash
exit
```

Your prompt should return to the `aws-client` host.

## 9. Add the aws-client Public Key to EC2

Run the following command on `aws-client` to display the public key. The public key is safe to copy; never share or copy the contents of `/root/.ssh/id_rsa` because that is the private key.

```bash
cat /root/.ssh/id_rsa.pub
```

Example output:

```text
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQ... root@aws-client
```

Copy the complete public-key output. From `aws-client`, enter the EC2 instance again using the initial key pair:

```bash
ssh -i datacenter-ec2-key.pem ec2-user@PUBLIC_IP_ADDRESS
```

Once the prompt shows that you are inside EC2, append the copied public key to root's authorized keys:

```bash
echo 'PASTE_THE_COMPLETE_PUBLIC_KEY_HERE' | sudo tee -a /root/.ssh/authorized_keys
sudo chmod 600 /root/.ssh/authorized_keys
```

The `tee -a` command appends the key without deleting any existing authorized keys. The `600` permission is required because SSH must treat this file as a private, trusted access-control file.

Exit the EC2 instance to return to `aws-client`:

```bash
exit
```

If root SSH login is disabled by the image, update the SSH server configuration and restart the service according to the image's security policy.

## 10. Test Passwordless Root SSH from aws-client

At this point you should be back on `aws-client`. Run the command below from `aws-client`; this enters the EC2 instance directly as `root` using `/root/.ssh/id_rsa`, rather than the temporary AWS key pair.

```bash
ssh -i /root/.ssh/id_rsa root@PUBLIC_IP_ADDRESS
```

Successful output will look similar to this:

```text
[root@ip-10-xxx-xxx-xxx ~]#
```

The connection should not ask for a password or the initial EC2 key pair.

## SSH Authentication Summary

The AWS key pair is used for the initial connection. The new key created on `aws-client` is used for subsequent passwordless root access.

```text
aws-client /root/.ssh/id_rsa
        |
        | SSH authentication using the private key
        v
EC2 instance /root/.ssh/authorized_keys
        |
        | Public key matches the private key
        v
Linux root user
```
    