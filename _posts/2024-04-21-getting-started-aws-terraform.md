---
layout: post
title:  "Getting started with AWS using Terraform"
date: 2024-04-21 22:02:38 +0100
tags: [aws, cloud, terraform]
---

## Introduction

This post demonstrates how to quickly get started with [Amazon Web Services (AWS)](https://aws.amazon.com/) using the [infrastructure-as-code](https://en.wikipedia.org/wiki/Infrastructure_as_code) tool [Terraform](https://www.terraform.io/). We'll set up:

- A free-tier eligible [EC2](https://aws.amazon.com/ec2/) instance.
- A [CloudWatch](https://aws.amazon.com/cloudwatch/) alarm that triggers whenever billing charges exceed a predetermined threshold.
- An email subscription (using [SNS](https://aws.amazon.com/sns/)) to receive an email whenever the above alarm is triggered.

> A GitHub repo containing the full code for this tutorial can be found [here](https://github.com/fnery/hello-aws-terraform).
{: .prompt-tip }

## Install the AWS and Terraform CLIs

Start by [creating](https://portal.aws.amazon.com/billing/signup) an AWS account. Then, [create](https://docs.aws.amazon.com/cli/latest/userguide/cli-authentication-user.html#cli-authentication-user-create) an [IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html) with [programmatic access](https://docs.aws.amazon.com/workspaces-web/latest/adminguide/getting-started-iam-user-access-keys.html) and [get](https://docs.aws.amazon.com/cli/latest/userguide/cli-authentication-user.html#cli-authentication-user-get) access keys[^1] to use with [AWS CLI](https://aws.amazon.com/cli/) (a [requirement](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-build#prerequisites) to use Terraform). Then, [install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions) the AWS CLI and [enable](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html#turning_on_billing_metrics) billing alerts.

Now, configure the AWS CLI by running `aws configure` and entering your credentials:

```bash
aws configure
# AWS Access Key ID [None]: <paste access key here>
# AWS Secret Access Key [None]: <paste secret access key here>
# Default region name [None]: us-east-1
# Default output format [None]: json
```

This should update the `credentials` file (which on Linux is located on `~/.aws`):

```bash
cat ~/.aws/credentials
# [default]
# aws_access_key_id = <redacted>
# aws_secret_access_key = <redacted>
```

Where `[default]` is the profile name.

Next, [install](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) the Terraform CLI. If all goes well, you will see the following:

```bash
terraform --help
# Usage: terraform [global options] <subcommand> [args]
#
# The available commands for execution are listed below.
# The primary workflow commands are given first, followed by
# less common or more advanced commands.
#
# (...)
```

## Set up the Terraform configuration

Clone the [fnery/hello-aws-terraform](https://github.com/fnery/hello-aws-terraform) repo. Let's go through the contents of each of the terraform (`.tf`) files.

### main.tf

First, define the [AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs). Note we're using the `default` AWS CLI as we saw above.

```hcl
# Provider configuration
provider "aws" {
    profile = "default"
    region = "us-east-1"
}
```
{: file="main.tf (excerpt)" }

We'll want to [SSH](https://en.wikipedia.org/wiki/Secure_Shell) into the EC2 instance we're about to create. To enable this, we need to create:

1. A security group (using the default VPC configuration) to allow SSH traffic.
2. An RSA key pair. The private key for SSH access will be saved on `terraform-key.pem` which will be stored locally on the current working directory.

This is achieved by the following snippet:

```hcl
# Retrieve default VPC configuration from AWS
data "aws_vpc" "default" {
  default = true
}

# Security group to allow SSH traffic from any IP address
resource "aws_security_group" "allow_ssh" {
  name        = "allow-ssh"
  description = "Allow SSH inbound traffic"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Allow SSH"
  }
}

# Private RSA key for use in securing SSH access
resource "tls_private_key" "rsa_4096" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

# AWS key pair using the public key generated from the private RSA key
resource "aws_key_pair" "terraform_key" {
  key_name   = "terraform-key"
  public_key = tls_private_key.rsa_4096.public_key_openssh
}

# Save private RSA key locally for SSH access
resource "local_file" "private_key" {
  content  = tls_private_key.rsa_4096.private_key_pem
  filename = "${path.cwd}/terraform-key.pem"
  file_permission = "0400"
}
```
{: file="main.tf (excerpt)" }

> Allowing SSH access from any IP address (as done above) poses a security risk. While acceptable for this demo, restricting SSH access to trusted IP addresses improves security and is generally recommended.
{: .prompt-warning }

Finally, we'll deploy the EC2 instance:

```hcl
# Deploy EC2 instance
resource "aws_instance" "app_server" {
    ami = "ami-04e5276ebb8451442"
    instance_type = "t2.micro"
    key_name = aws_key_pair.terraform_key.key_name
    security_groups = [aws_security_group.allow_ssh.name]
    tags = {
        Name = "app-server"
    }
}
```
{: file="main.tf (excerpt)" }

The type of machine is specified using `ami` (Amazon Machine Image) and `instance_type` (hardware configuration)[^2]. We also specify the key and security group we previously set up.

Let's finish `main.tf` by setting up a [CloudWatch](https://aws.amazon.com/cloudwatch/) alarm that triggers whenever billing charges exceed $5 and an email subscription (using [SNS](https://aws.amazon.com/sns/)) to receive an email whenever the above alarm is triggered:

```hcl
# SNS topic for billing alerts
resource "aws_sns_topic" "billing_alarm" {
  name = "billing-alarm-topic"
}

# Subscribe email address to the SNS topic for billing alerts; requires confirmation
resource "aws_sns_topic_subscription" "billing_alarm_email" {
  topic_arn = aws_sns_topic.billing_alarm.arn
  protocol  = "email"
  endpoint  = var.email
}

# CloudWatch metric alarm to notify via SNS when estimated charges exceed a defined threshold
resource "aws_cloudwatch_metric_alarm" "billing_alarm" {
  alarm_name                = "billing-alarm"
  comparison_operator       = "GreaterThanThreshold"
  evaluation_periods        = 1
  metric_name               = "EstimatedCharges"
  namespace                 = "AWS/Billing"
  period                    = 21600
  statistic                 = "Maximum"
  threshold                 = 5
  alarm_description         = "Alert if estimated charges exceed $5.00"
  insufficient_data_actions = []
  actions_enabled           = true
  alarm_actions             = [aws_sns_topic.billing_alarm.arn]
  dimensions = {
      Currency = "USD"
  }
}
```
{: file="main.tf (excerpt)" }

### outputs.tf

The [outputs](https://developer.hashicorp.com/terraform/language/values/outputs) file exposes information about our deployment. In this case, we're exposing the public DNS name of the deployed EC2 instance, which we'll later use to SSH into it.

```hcl
# Output public DNS name of the deployed EC2 instance
output "public_dns" {
  value = aws_instance.app_server.public_dns
  description = "The public DNS name of the instance"
}
```
{: file="outputs.tf" }

### variables.tf

The [variables](https://developer.hashicorp.com/terraform/language/values/variables) file introduces an `email` variable which is injected in the `main.tf` configuration file.

```hcl
variable "email" {
  description = "Email to subscribe to SNS topic alarms"
  type        = string
}
```
{: file="variables.tf" }

As we can see, no email value is specified. We'll do this below, when applying the Terraform configuration.

## Deploy on AWS using Terraform

Assuming we have a terminal open in the root directory of the [fnery/hello-aws-terraform](https://github.com/fnery/hello-aws-terraform) repo, start by [initializing](https://developer.hashicorp.com/terraform/cli/commands/init) it as a Terraform working directory:

```bash
terraform init
# Initializing the backend...
#
# Initializing provider plugins...
# - Reusing previous version of hashicorp/aws from the dependency lock file
# - Reusing previous version of hashicorp/tls from the dependency lock file
# - Reusing previous version of hashicorp/local from the dependency lock file
# - Installing hashicorp/tls v4.0.5...
# - Installed hashicorp/tls v4.0.5 (signed by HashiCorp)
# - Installing hashicorp/local v2.5.1...
# - Installed hashicorp/local v2.5.1 (signed by HashiCorp)
# - Installing hashicorp/aws v5.46.0...
# - Installed hashicorp/aws v5.46.0 (signed by HashiCorp)
#
# Terraform has been successfully initialized!
```

Now, let's run `terraform plan` to check Terraform's deployment plan:

```bash
terraform plan
# var.email
#   Email to subscribe to SNS topic alarms
#
#   Enter a value: abc@xyz.com 👈
#
# data.aws_vpc.default: Reading...
# data.aws_vpc.default: Read complete after 1s
#
# Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
#   + create
#
# Terraform will perform the following actions:
#   (...)
```

Note how Terraform requested a value to be added for the `email` variable (highlighted by 👈). Above, we're showing a dummy value, `abc@xyz.com`, but you'd add the email where you'd like to get billing alarms. Another option would have been to specify the value of the `email` (and any other variables) via a [`terraform.tfvars` file](https://developer.hashicorp.com/terraform/language/values/variables#variable-definitions-tfvars-files), which would have looked like this[^3]:

```
email = "abc@xyz.com"
```
{: file="terraform.tfvars" }

> The [fnery/hello-aws-terraform](https://github.com/fnery/hello-aws-terraform) repo includes a `.gitignore` file which ensures files that typically contain senstive information (such as the `.tfvars` files) are not commited into version control[^4].
{: .prompt-warning }

Assuming we're happy about Terraform's plan, we can implement it:

```bash
terraform apply
# data.aws_vpc.default: Reading...
# data.aws_vpc.default: Read complete after 1s
#
# Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
#   + create
#
# Terraform will perform the following actions:
#
#   aws_cloudwatch_metric_alarm.billing_alarm will be created
#   (...)
#   aws_instance.app_server will be created
#   (...)
#   aws_key_pair.terraform_key will be created
#   (...)
#   aws_security_group.allow_ssh will be created
#   (...)
#   aws_sns_topic.billing_alarm will be created
#   (...)
#   aws_sns_topic_subscription.billing_alarm_email will be created
#   (...)
#   local_file.private_key will be created
#   (...)
#   tls_private_key.rsa_4096 will be created
#   (...)
#
# Plan: 8 to add, 0 to change, 0 to destroy.
#
# Changes to Outputs:
#   + public_dns = (known after apply)
#
# Do you want to perform these actions?
#   Terraform will perform the actions described above.
#   Only 'yes' will be accepted to approve.
#
#   Enter a value: yes 👈
#
# tls_private_key.rsa_4096: Creating...
# tls_private_key.rsa_4096: Creation complete after 0s [id=...]
# local_file.private_key: Creating...
# aws_key_pair.terraform_key: Creating...
# aws_sns_topic.billing_alarm: Creating...
# local_file.private_key: Creation complete after 0s [id=...]
# aws_security_group.allow_ssh: Creating...
# aws_key_pair.terraform_key: Creation complete after 1s [id=...]
# aws_sns_topic.billing_alarm: Creation complete after 1s [id=...]
# aws_sns_topic_subscription.billing_alarm_email: Creating...
# aws_cloudwatch_metric_alarm.billing_alarm: Creating...
# aws_sns_topic_subscription.billing_alarm_email: Creation complete after 1s [id=...]
# aws_cloudwatch_metric_alarm.billing_alarm: Creation complete after 1s [id=...]
# aws_security_group.allow_ssh: Creation complete after 4s [id=...]
# aws_instance.app_server: Creating...
# aws_instance.app_server: Still creating... [10s elapsed]
# aws_instance.app_server: Still creating... [20s elapsed]
# aws_instance.app_server: Still creating... [30s elapsed]
# aws_instance.app_server: Creation complete after 35s [id=...]
#
# Apply complete! Resources: 8 added, 0 changed, 0 destroyed.
#
# Outputs:
#
# public_dns = "ec2-44-204-167-153.compute-1.amazonaws.com"
```

After typing `yes` to accept the listed actions (see 👈), Terraform will created the specified resources. 

> The specified `email` will receive a "Subscription Confirmation" email from [SNS](https://aws.amazon.com/sns/) containing the link to confirm the subscription.
{: .prompt-tip }

Note how the value of the output variable we defined `public_dns` is printed in the Outputs section.

Let's use the AWS CLI to [describe](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/describe-instances.html) the instance we just deployed:

```bash
aws ec2 describe-instances
# {
#     "Reservations": [
#         {
#             "Groups": [],
#             "Instances": [
#                 {
#                     "AmiLaunchIndex": 0,
#                     "ImageId": "ami-04e5276ebb8451442",
#                     "InstanceId": "i-00f9dd988b75a8610",
#                     "InstanceType": "t2.micro",
#                     "KeyName": "terraform-key",
#                     "LaunchTime": "2024-04-22T14:37:55+00:00",
# (...)
```

Assuming `terraform apply` created the `terraform-key.pem` in the working directory, we're now be ready to SSH into the EC2 instance. On an Amazon Linux 2023 [AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html), the [default](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/managing-users.html) user is `ec2-user`. We already got the value of the host (Public DNS name of the instance) in the outputs of `terraform apply`, but we can also check it any time as follows:

```bash
terraform output public_dns
# "ec2-44-204-167-153.compute-1.amazonaws.com"
```

Let's SSH into the instance as follows:

```bash
ssh -i "terraform-key.pem" ec2-user@ec2-44-204-167-153.compute-1.amazonaws.com
# The authenticity of host 'ec2-44-204-167-153.compute-1.amazonaws.com (44.204.167.153)' can't be established.
# ED25519 key fingerprint is SHA256:X0P8sQRqB6lh1E0Qj2tns1uP2BvKCuG9R0vMdjRSLaQ.
# This key is not known by any other names
# Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
# Warning: Permanently added 'ec2-44-204-167-153.compute-1.amazonaws.com' (ED25519) to the list of known hosts.
#    ,     #_
#    ~\_  ####_        Amazon Linux 2023
#   ~~  \_#####\
#   ~~     \###|
#   ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
#    ~~       V~' '->
#     ~~~         /
#       ~~._.   _/
#          _/ _/
#        _/m/'
# [ec2-user@ip-172-31-84-77 ~]$ # We're in!
```

Finally, let's check our CloudWatch alarm:

```bash
aws cloudwatch describe-alarms
# {
#     "MetricAlarms": [
#         {
#             "AlarmName": "BillingAlarm",
#             "AlarmArn": "arn:aws:cloudwatch:us-east-1:637423553953:alarm:BillingAlarm",
#             "AlarmConfigurationUpdatedTimestamp": "2024-04-18T10:48:38.061000+00:00",
#             "ActionsEnabled": true,
#             "OKActions": [],
#             "AlarmActions": [
#                 "arn:aws:sns:us-east-1:637423553953:BillingAlarmTopic"
#             ],
# (...)
```

All looks good. Before ending the demo, let's clean up everything we set up. With Terraform, this is as easy as doing:

```bash
terraform destroy
```

---

[^1]: For security reasons, AWS [recommends](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html#ru-bp-access) using access keys attached to an IAM user instead of the root user.
[^2]: This combination is free tier eligible so this should cost nothing. In any case, we'll [destroy](https://developer.hashicorp.com/terraform/cli/commands/destroy) everything at the end of the demo.
[^3]: I suggest creating this file otherwise subsequent `terraform apply` commands will prompt for `email` again.
[^4]: See [here](https://developer.hashicorp.com/terraform/tutorials/configuration-language/sensitive-variables) for more on protecting sensitive Terraform input variables.
