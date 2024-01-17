# Terraform with AWS

From here, will be diving into specific AWS concepts and example terraform configuration for each.

## Configuring AWS provider

Your AWS provider will need to be configured with the right set of credentials to create resources in your account. This can be done within the `provider` block. Example:

```
provider "aws" {
    region = "us-west-2"
    access_key = "AKIAI44QH8DHBEXAMPLE"
    secret_key = "JE7MTGBCLWBF/2TK/H3YCO8N..."
}
```

However, hardcoding credentials in your configuration file is not good practice. If you simply configure your AWS CLI using `aws configure`, this will write your credentials locally to your `.aws/config/credentials` file. Terraform will automatically use credentials stored in this file.

Env variables can be used as well - e.g. `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

## IAM

IAM = identity and access management.

In order to be given permissions, `users` are granted `IAM policies`. `IAM groups` can be configured to map a preset list of policies to a set of users automatically. Policies can be granted to AWS services or external services via `IAM roles` in order to grant them permission to interact with other resources programatically. IAM is a global concept and is not specific to any region. Two forms of access can be granted through IAM - either through the management console, or programmatic access through the AWS CLI.

IAM terraform docs can be found [here](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user).

Example IAM user config:

```
resource "aws_iam_user" "admin-user" {
    name = "lucy"
    tags = {
        Description = "Technical Team Leader"
    }
}
```

Example IAM policy config. You will need to craft your IAM policy file in order to define what permissions a particular policy is given. This is an AWS specific json format.

```
resource "aws_iam_policy" "adminUser" {
    name = "AdminUsers"
    policy = <<EOF
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "*",
                "Resource": "*"
            }
        ]
    }
    EOF
}
```

The json policy can also be stored in a separate file in the directory, e.g. `admin-policy.json`. This can then be referenced like so:

```
resource "aws_iam_policy" "adminUser" {
    name = "AdminUsers"
    policy = file("admin-policy.json")
}
```

Example policy attachment giving `lucy` the `AdminUsers` policy.

```
resource "aws_iam_user_policy_attachment" "lucy-admin-access" {
    user = aws_iam_user.admin-user.name
    policy_arn = aws_iam_policy.adminUser.arn
}
```

## S3

Considerations when creating buckets:

- name must be globally unique
- name must be DNS compliant
- max file size is 5TB

Buckets are accessible at https://\<bucket-name\>.\<region\>.amazonaws.com, e.g. https://all-pets.us-west-1.amazonaws.com.

Example S3 resource in terraform:

```
resource "aws_s3_bucket" "finance" {
    bucket = "finance-21092020"
    tags = {
        Description = "Finance and Payroll"
    }
}
```

Example to upload a file to the bucket via TF.

```
resource "aws_s3_bucket_object" "finance-2020" {
    content = "/root/finance/finance-2020.doc"
    key = "finance-2020.doc"
    bucket = aws_s3_bucket.finance.id
}
```

Say there is an existing group `finance-analysts` that needs access to the bucket. The group was not created via TF. You can grant that access by first defining a datasource for the group, then creating a policy for it:

```
data "aws_iam_group" "finance-data" {
    group_name = "finance-analysts"
}

resource "aws_s3_bucket_policy" "finance-policy" {
    bucket = aws_s3_bucket.finance.id
    policy = <<EOF
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": "*",
                "Effect": "Allow",
                "Resource": "arn:aws:s3::;${aws_s3_bucket.finance.id}/*"
                "Principal": {
                    "AWS": [
                        "${data.aws_iam_group.finance-data.arn}"
                    ]
                }
            }
        ]
    }
    EOF
}
```

## DynamoDB

Example DynamoDB table created with TF:

```
resource "aws_dynamodb_table" "cars" {
    name = "cars"
    hash_key = "VIN"
    billing_mode = "PAY_PER_REQUEST"
    attribute {
        name = "VIN"
        type = "S"
    }
}
```

Example inserting items with TF:

```
resource "aws_dynamodb_table_item" "car-items" {
    table_name = aws_dynamodb_table.cars.name
    hash_key = aws_dynamodb_table.cars.hash_key
    item = <<EOF
    {
        "Manufacturer": {"S": "Toyota"},
        "Make": {"S": "Corrola"},
        "Year": {"N": "2004"}
        "VIN": {"S": "4Y1SL65848Z411439"}
    }
    EOF
}
```

## EC2

Example creating EC2 instance through TF. We will also create the keypair such that we can SSH into the server, and ensure that traffic on port 22 is allowed. We want to place the resulting public IP into an output variable so we know where to access the new host.

```
resource "aws_instance" "webserver" {
    ami = "ami-0edab43b6fa892279"
    instance_type = "t2.micro"
    tags = {
        Name = "webserver"
        Description = "An Nginx WebServer on Ubuntu"
    }
    user_data = <<-EOF
                #!/bin/bash
                sudo apt update
                sudo apt install nginx -y
                systemctl enable nginx
                systemctl start nginx
                EOF

    key_name = aws_key_pair.web.id
    vpc_security_group_ids = [ aws_security_group.ssh-access.id ]
}

resource "aws_key_pair" "web" {
    public_key = file("/root/.ssh/web.pub")
}

resource "aws_security_group" "ssh-access" {
    name = "ssh-access"
    description = "Allow SSH access from the Internet"
    ingress {
        from_port = 22
        to_port = 22
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
}

output publicip {
    value = aws_instance.webserver.public_ip
}
```
