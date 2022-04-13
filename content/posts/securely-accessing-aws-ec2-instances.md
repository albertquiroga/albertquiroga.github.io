---
title: Securely accessing AWS EC2 instances
date: 2022-04-13T21:00:00+09:00
draft: false
tags: 
    - cloud
    - aws
category: cloud
keywords: 
    - aws
    - ec2
---

Accessing EC2 instances in an AWS account is a cumbersome, repetitve process that can even compromise your security if done careless. Let's say I wanted to launch a new instance right now. I would need to:

1. Create an EC2 keypair and download it to the client machine
1. Run a `create-instance` command
2. Figure out the public hostname or IP address of the new instance
3. Create or modify the rules of the attached security group to allow SSH traffic from my IP address
4. Log into the instance

These 4 steps might not seem like a very long process, but it can get repetitive and cumbersome if you're constantly launching instances to test a deployment. If you are working through a VPN like most WFH users do today, you are also at risk of your public IP address changing and losing permissions to connect through the security group, which means you have to be constantly monitoring and updating rules. 

Additionally, this presents a challenge from a security standpoint: all of the above presume the instance is in a public VPC subnet. Ideally, anything that contains critical information or a critical service should not be running in a public subnet, but that means either users can't access it or they have to go through edge nodes, which add complexity to the architecture. As an admin, you are also trusting users to properly manage their downloaded EC2 keypairs, not sharing them with others and not uploading them to storage services.

Luckily, AWS provides a solution for that: the Session Manager feature in AWS Systems Manager.

## AWS Systems Manager

[Systems Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html) is a very wide AWS service with tons of features to control, manage and configure AWS resources. Some of its features are in fact distinct enough that it could probably be broken down to several sub-services, but all of them relate to managing EC2 instances - with things like deploying patches, centralizing configuration management or doing inventory.

The way Systems Manager (abbreviated to SSM) works is by installing a service that runs on all EC2 instances that you want to manage through it. This service (known as SSM Agent) then communicates with SSM, sending status updates and receiving commands. SSM Agent comes pre-installed in several AWS AMIs (such as Amazon Linux), which means you're probably already running it unknowingly. 

There's two features of SSM that are related to our use case:

* [Run Command](https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html), which lets you pass commands to run to your EC2 instances through the AWS console, CLI or SDK.
* [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html), which lets you create SSH sessions to EC2 instances through the AWS console, CLI or SDK.

Run Command is basically a baby version of Session Manager, which also serves our use case but will obviously not work for interactive sessions. When using Session Manager, the SSM Agent in the EC2 instance is acting as a liason between you and the instance, setting up a terminal session and routing it through the AWS infrastructure. This comes with the following benefits:

* You no longer have to deal with the networking side of it. No public subnets, no retrieving IP addresses, no security groups. All you need is a route between your computer and the public SSM endpoint. The `create-instance` command returns an instance ID and you can connect with that right away.
* You no longer have to deal with EC2 keypairs. Authentication is done through IAM, which also means admins can now manage access to instances via IAM users and policies.
* You are not losing any functionality. You can have a shell session through the web console (ew), but also through the AWS CLI or even with an SSH command. And that includes SSH tunnels too.

## Making it work

There's two sides of this, the client (your computer, where you're connecting from) and the server (the EC2 instance).

### Client-side requirements

There are outlined in the AWS documentation [here](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-prerequisites.html), but basically you'll need to:

* Have AWS CLI version 1.16.12 or higher
* Install the Sessions Manager plugin, instructions [here](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

### Server-side requirements

Your EC2 instance basically needs to:

* Have the SSM Agent installed and running. If you are running Amazon Linux 2 instances, you already have this.
* Have an EC2 Instance Profile (IAM role) with the right permissions attached to it. These permissions are pre-included in the `AmazonSSMManagedInstanceCore` AWS-managed policy if you want to add them easily.
* The EC2 instance needs to be able to reach the regional ec2messages, ssm and ssmmessages AWS endpoints. Since your instance will be running in a private subnet, you'll need to create VPC endpoints within it for this happen.

Once these are met on both ends, you should be able to connect.

## Connecting via SSM

SSM lets you connect to instances in three different ways:

* Using the AWS Web Console. This will create a web-based terminal session. Useful for quick/small sessions, but probably not the best experience if you're used to SSH terminal commands.
* Using the AWS CLI. This lets you create sessions even if you don't have access to an SSH client/server. 
* Using the SSH command. This will let you connect with an instance-id rather than an IP address and will let you use SSM to manage logging and permissions, but you will need a keypair.

### Using the AWS CLI

Simply run the `start-session` command to start an interactive session:

```bash
aws ssm start-session --target <instance-id>
```

You can even create SSH tunnels to access web servers (or any other server) running in the instance:

```bash
aws ssm start-session --target <instance-id> --document-name AWS-StartPortForwardingSession --parameters '{"portNumber":["10"], "localPortNumber":["12345"]}'
```

Whenever you connect, you'll be logged in as the `ssm-user` user. You can easily switch to `ec2-user` (or any other user) with a sudo command:

```bash
sudo -u ec2-user -s
```

This also happens with web console connections.

### Using the SSH client

The Sessions Manager plugin also updates SSH to be able to connect to instance ids rather than IP addresses or hostnames:

```bash
ssh -i <key.pem> <username>@<instance-id>
```

You will need an EC2 keypair and the username must be the one linked to that keypair. In the case of EC2 instances running Amazon Linux, that's `ec2-user` by default. 

You can even use SCP to move files:

```bash
scp -i <key.pem> <sample_file> <username>@<instance-id>
```
