# Terraform Notes

My notes from going through the [Udemy course Terraform for the Absolute Beginners with Labs](https://www.udemy.com/course/terraform-for-the-absolute-beginners/) and associated hands-on labs to try to learn the new tool. These could later be broken up into nicer, logical sections divided between pages. For now, most of the content is within this page, except for the AWS section which I pulled into another markdown file.

## Getting Started

Terraform is a command-line tool that you can use to manage your infrastructure as code. You follow the instructions here to download the CLI for your corresponding architecture: [https://developer.hashicorp.com/terraform/install](https://developer.hashicorp.com/terraform/install).

## Vocab

- `Resource` - an object that Terraform manages. It could be a local file, a virtual machine on the cloud such as EC2, or any other cloud resource.
- `Provider` - specific plugins for different types of infrastructure that you can manage via TF (e.g. one provider exists for each major cloud platform, one for local, many others). Providers can be official (owned and maintained by Hashicorp), partner (owned and maintained by 3rd party, but approved by Hashicorp), and community (contributed by individual developers).
- `Variables` - can be input or output, pretty self explanatory. Allows you to assign variables for parameters that can be overwritten while applying.

## HCL Basics

Terraform uses configuration files that are written in [HCL](https://developer.hashicorp.com/terraform/language/syntax/configuration) to deploy infrastructure resources. These files have a `.tf` file extension. Any files with a `.tf` extension within a given directory will be tracked by TF. Common to have one configuration file called `main.tf`.

HCL syntax uses the following syntax:

```
<block> <parameters> {
    key1 = value1
    key2 = value2
}
```

And a specific example:

```
resource "local_file" "pet" {
    filename = "/root/pets.txt"
    content = "We love pets!"
}
```

In the example above:

- Block name = resource. We use this to create a given resource in TF.
- The first paramater = resource type. This is split on the first "\_" into the provider and resource that we want to create. In the example above, we want to create a "file" resource from the provider "local".
- The second parameter = resource name. Logical name to identify the resource, can be anything. In this case, the name is "pet".
- Within the curly braces, we add the arguments. The args are specific to the resource type that you are trying to create. As we are creating a file here, we add in args for the filename and content to be created within that file.

Another basic example to create an ec2 instance [docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance):

```
resource "aws_instance" "webserver" {
    ami = "ami-0c2f25c1f66a1ff4d"
    instance_type = "t2.micro"
}
```

## Terraform commands:

There are 4 primary commands that you need to run to create and manage infrastructure with Terraform after writing the configuration:

- `terraform init` - will check your configuration files and initialize the working directory containing your `.tf` files. Will install the necessary provider plugins in order to carry out the work. Only needs to be run initially and when you add or change providers.
- `terraform plan` - will show the actions that will be carried out by TF, but does not actually take any actions on the infra.
- `terraform apply` - will ask the user to confirm if they want to proceed by typing `yes`, and then will actually execute the plans. Can be re-run to apply any updates.
- `terraform destroy` - will destroy all resources created within the given terraform directory.

Other useful commands:

- `terraform show` - see the details of any resources that have been created.
- `terraform validate` - check syntax of your config files.
- `terraform fmt` - nicely format your config files.
- `terraform providers` - show all providers in use.
- `terraform output` - show all values for output variables.
- `terraform refresh` - manually force an update to the state file. Run by default for each terraform plan or apply.
- `terraform graph | dot -Tsvg > graph.svg` - creates a `graph.svg` file for a visual representation of your terraform resources and dependencies.
- `terraform console` - opens a console where you can run functions against the current state.
- `terraform workspace new <Name>`, `terraform workspace list`, `terraform workspace select <Name>` - see Workspaces

## Providers

Providers are plug-ins for a particular type of infrastructure. For example, a provider `local` exists to create local files. Provider `aws` exists to create AWS cloud-managed resources. A full list of providers can be found [here](https://registry.terraform.io/browse/providers).

In order to install provider plug-ins, you can start by using them within your configuration. When introducing a new provider, run `terraform init` in order to install the necessary plugins.

You may need to ensure that a specific version of a provider. To use a specific provider, you can add a block like the following into your configuration file:

```
terraform {
    required_providers {
        local = {
            source = "hashicorp/local"
            version = "1.4.0"
        }
    }
}
```

A block like this will ensure that you are pinning to the 1.4.0 version of the `local` provider for your configuration directory. You can easily find instructions for this under the terraform docs for a particular provider. See [here](https://registry.terraform.io/providers/hashicorp/aws/latest) for an example for AWS - click the "Use Provider" dropdown in order to easily copy/paste this block.

You can use operators `<`, `>`, `!=`, `>=`, `<=`, `~>` (greater than, within a major version) in your required_providers block. For example:

```
terraform {
    required_providers {
        local = {
            source = "hashicorp/local"
            version = "> 1.4.0"
        }
    }
}
```

## Input variables

To avoid hard-coding parameters to your resources, use input variables. Common to have a separate file for these called `variables.tf`. Example variable syntax:

```
variable "filename" {
    default = "/root/pets.txt"
    type = string
    description = "the path of the local file"
}

variable "content" {
    default = "We love pets!"
}
```

Type and description are optional. If type is omitted, it will be set to `any`.

### Types and definitions

Common types of variables that you can create:

- string
- number
- bool
- any
- list
- map
- set - list, but cannot have duplicate elements
- object
- tuple

To use an input variable, replace argument values with variable names prefixed with `var`. Example:

```
resource "local_file" "pet" {
    filename = var.filename
    content = var.content
}
```

Some specific usages of the more complex data types:

**List**:

```
variable "prefix" {
    default = ["Mr", "Mrs", "Sir"]
    type = list(string)
}

resource "random_pet" "my-pet" {
    prefix = var.prefix[0]
}
```

**Map**

```
variable "file-content" {
    type = map(string)
    default = {
        "statement1" = "We love pets!"
        "statement2" = "We love animals!"
    }
}

resource "local_file" "my-pet" {
    filename = "/root/pets.txt"
    content = var.file-content["statement2"]
}
```

**Objects** - combine any/all data types

```
variable "bella" {
    type = object({
        name = string
        color = string
        age = number
        food = list(string)
        favorite_pet = bool
    })
    default = {
        name = "bella"
        color = "brown"
        age = 7
        food = ["fish", "chicken", "turkey"]
        favorite_pet = true
    }
}
```

**Tuple** - like a list, but allows for defined set mixed variable types

```
variable "kitty" {
    type = tuple([string, number, bool])
    default = ["cat", 7, true]
}
```

### Using tf variables

There are a number of methods that you can use to change variables from their defaults when running tf apply:

**Command line flags**

Use `-var "<variable name>=<overriden variable value>` when running tf apply. Example:

```shell
terraform apply -var "filename=/root/pets.txt" -var "content=We love Pets!"
```

**Environment variables**

Env variables can be created with format `TF_VAR_<variable name>` before running terraform apply. Example:

```shell
export TF_VAR_filename="/root/pets.txt"
export TF_VAR_content="We love pets!"
terraform apply
```

**Definition files**

Variable definition files can be created that manage the variable overrides. Can be named anything, but need to use extension `.tfvars` or `.tfvars.json`. Such a file would look like:

`terraform.tfvars`

```
filename = "/root/pets.txt"
content = "We love pets!"
```

Files with names - `terraform.tfvars`, `terraform.tfvars.json`, `*.auto.tfvars`, `*.auto.tfvars.json` will be automatically loaded if detected. If you use any other filename (e.g. `variable.tfvars`), you will need to explicitly pass into the CLI when running apply. Example:

```shell
terraform apply -var-file variables.tfvars
```

**Precedence**

As there are multiple ways to define variables, here is how the CLI will take precedence if there are multiple definitions for variables across different files:

1. Command line flag `-var` or `-var-file`
2. `*.auto.tfvars` (alphabetical order)
3. `terraform.tfvars`
4. `Environment variables`

## Resource attributes and dependencies

Imagine you want to reference a given attribute of one terraform resource in the creation of another. As an example - say you have one resource to generate a TLS private key for a certficate, and would like to take the contents from that resource and write it into a file.

Each provider and resource type has a list of supported attributes that will be created/associated with the resource. Here for example is the list of attributes for the `tls_private_key` resource [link](https://registry.terraform.io/providers/hashicorp/tls/latest/docs/resources/private_key#read-only). These resources can be accessible from other resources using the syntax `resource-type.resource-name.attribute`. For example:

```
resource "tls_private_key" "my-key" {
    algorithm "RSA"
    rsa_bits = 4096
}

resource "local_file" "priv-key" {
    filename = "/root/privkey.pem"
    content = tls_private_key.my-key.private_key_pem
}
```

Using resource attributes in your configuration creates an `implicit dependency`. This means that the `my-key` resource must be created before the `priv-key` file because it needs the contents to create the file in the first place. In this case, terraform cannot delete the `my-key` resource while the `priv-key` resource still exists, as it is dependent on it existing to fulfill the contents. When deleting all resources, terraform will delete `priv-key` before deleting `my-key`.

Implicit dependencies will be created automatically by the use of resource attributes. Explicit dependencies can be defined with the following `depends-on` block and syntax:

```
resource "local_file" "pet" {
    filename = var.filename
    content = "My favorite pet is Mr. Cat"
    depends_on [
        "random_pet.my-pet"
    ]
}

resource "random_pet" "my-pet" {
    prefix = var.prefix
    separator = var.separator
    length = var.length
}
```

## Output variables

To create output variables in TF, you can use the following syntax:

```
resource "random_pet" "my-pet" {
    prefix = var.prefix
    separator = var.separator
    length = var.length
}

output pet-name {
    value = random_pet.my-pet.id
    description = "Record the value of the pet ID generated by the random_pet resource
}
```

When you run `terraform apply`, the output variable will be printed on the screen. You can also use `terraform output` to print these variables at any time, or `terraform output <variable-name>` to fetch a specific value on-demand.

Output variables are mostly used to quickly display details about a given variable on-screen, or to pass to other tools (e.g. Ansible, shell scripts). You do not need these to pass variables between terraform resources in a given config directory.

## Terraform state

Terraform manages state in order to keep track of what changes have already been applied and calculate diffs between plans. This state is stored in the file `terraform.tfstate`, which gets created on the initial run of `terraform apply` as json. Terraform by default refreshes state upon each `terraform plan` or `terraform apply` in order to figure out what diffs it needs to roll out based on config or manual infra changes.

If you are managing hundreds or thousands of resources, it may not be feasible for terraform to reconcile state upon each operation. You can use the state file as the source of truth if it is not advantageous to reconcile upon each run using the flag `--refresh=false`.

Using a state file allows multiple devs on a team to collaborate more easily. All that is needed is to fetch the latest state file, and then each user can pick up from the latest source of truth and make changes as needed. It is important that different people are not making changes to the statefile at once as to avoid conflicts, so it is common practice to store the state file in a remote data store (e.g. S3, Terraform Cloud). As values in the state file may be sensitive (e.g. it has direct addresses and identifiers to all of your created resources, could contain passwords), it should always be stored in a secure storage (e.g. never directly in Github where the config files may be stored).

**You should never manually make changes to a state file**.

### Remote state through S3

To configure remote storage for your state file through S3, you can use a `terraform` block like the following. You must first run `terraform init` after including this to initialize the remote backend. After this, you can delete your local statefile.

```
terraform {
    backend "s3" {
        bucket = "terraform-state-bucket-01"
        key = "finance/terraform.tfstate"
        region = "us-west-1"
        dynamodb_table = "state-locking"
    }
}
```

The dynamo DB table must be included to implement state locking. This is important so that multiple people cannot apply TF at the same time, which would lead to an indeterminate state.

## Lifecycle rules and mutability

Different resources and attributes can either be `mutable` or `immutable`. If a given resource or attribute is immutable, terraform will delete and recreate it upon any changes made.

You can use lifecycle rules to define behavior around deletion + recreation. For instance, if you wanted to create a new resource before deleting the original resource, you add a lifecycle rule with the following syntax:

```
resource "local_file" "pet" {
    filename = "/root/pets.txt"
    content = "We love pets!"
    file_permission = "0700"

    lifecycle {
        create_before_destroy = true
    }
}
```

Common lifecycle rules include:

- create_before_destroy
- prevent_destroy
- ignore_changes

Specific note - when using the `create_before_destroy` lifecycle rule, ensure that your ID for your resource is unique. For instance, in the local_file example above, the ID is taken from the filename. If we use the create_before_destroy in that example, the following will occur:

- Update is made to local_file config
- Because of the lifecycle rule, it will first try to create the new file, which will update the file in place.
- Then, it will delete the original resource. As it has the same ID, it will delete the file that you just created.
- As a result, you will be left with no file after running the command, which was not your intent.

## Datasources

If you would like to reference attributes that were not created by your given terraform configuration directory, you can use `datasources`. For instance, say you created an file on your machine manually called `dogs.txt`, and would like to reference the contents of that file in your terraform configuration for `local_file.pet`. You can create read only references using the syntax of terraform resources using a data source like the following:

```
data "local_file" "dog" {
    filename = "/root/dogs.txt"
}

resource "local_file" "pet" {
    filename = "/root/pets.txt"
    content = data.local_file.dog.content
}
```

## Creating multiple instances

### Using `count`

`count` is a meta argument that you can use to create multiple copies of your resource. When using count > 1, it is important to ensure that each resource that you create has a unique ID. As an example, the following block would simply create 3 copies of the same file, resulting in one file created:

```
resource "local_file" "pet" {
    filename = var.filename
    count = 3
}

variable "filename" {
    default = "/root/pets.txt"
}
```

To modify this, you can use a list for your variable type, and use the syntax `count.index` in order to iterate through this list as you create each copy. This will result in 3 unique files being created.

```
resource "local_file" "pet" {
    filename = var.filename[count.index]
    count = 3
}

variable "filename" {
    default = [
        "/root/pets.txt",
        "/root/dogs.txt",
        "/root/cats.txt"
    ]
}
```

If you want the count to update based on the length of your list, you can use the `length()` function.

```
resource "local_file" "pet" {
    filename = var.filename[count.index]
    count = length(var.filename)
}
```

However, there is one catch here. If you were to remove `/root/pets.txt`, you would see the following behavior:

- file 0: modify from /root/pets.txt -> /root/dogs.txt
- file 1: modify from /root/dogs.txt -> /root/cats.txt
- file 2: delete file

### Using `for_each`

A smoother approach when working with dynamic lists is to use the `for_each` keyword. Here is the above example modified with `for_each`.

```
resource "local_file" "pet" {
    filename = each.value
    for_each = var.filename
}

variable "filename" {
    type = set(string)
    default = [
        "/root/pets.txt",
        "/root/dogs.txt",
        "/root/cats.txt"
    ]
}
```

The `for_each` keyword only works with the `set` or `map` objects. This is why we needed to explicitly set the type in the variable to be a set rather than a list.

If you remove `/root/pets.txt` from the variable above, you will see the behavior that 2 files for dogs.txt and cats.txt are untouched, but pets.txt will be deleted. This is because the `for_each` keyword identifies its items using their unique keys rather than their indices.

## Provisioners

Provisioners are hooks that can be run by TF upon creation. There are a few kinds of provisioners of note.

**Remote Exec**

A remote exec provisioner runs on some sort of remote resource once it has been created. For instance - after creating an EC2 instance, run a script on that instance (or somewhere other than the local machine). You can use the following syntax to define the script. Note that the connection field is necessary as you must define where the remote exec is being run.

```
resource "aws_instance" "webserver" {
    ami = "ami-0edab43b6fa892279"
    instance_type = "t2.micro"
    tags = {
        Name = "webserver"
        Description = "An Nginx WebServer on Ubuntu"
    }
    provisioner "remote-exec" {
        inline = [
            "sudo apt update",
            "sudo apt install nginx -y",
            "sudo systemctl enable nginx",
            "sudo systemctl start nginx"
        ]
    }
    connection {
        type = "ssh"
        host = self.public_ip
        user = "ubuntu"
        private_key = file("/root/.ssh/web")
    }

    key_name = aws_key_pair.web.id
    vpc_security_group_ids = [ aws_security_group.ssh-access.id ]
}
```

**Local exec**

This is used to run tasks on the local machine where the terraform binary run. For instance, if you wanted to save the public IP of an EC2 instance onto a local file:

```
resource "aws_instance" "webserver" {
    ami = "ami-0edab43b6fa892279"
    instance_type = "t2.micro"
    tags = {
        Name = "webserver"
        Description = "An Nginx WebServer on Ubuntu"
    }
    provisioner "local-exec" {
        command = "echo ${aws_instance.webserver .public_ip} >> /tmp/ips.txt"
    }

    key_name = aws_key_pair.web.id
    vpc_security_group_ids = [ aws_security_group.ssh-access.id ]
}
```

**Destroy time**

This is used when you want to run a provisioner before a resource is destroyed. Use the `when = destroy` keyword to denote a destroy time provisioner.

```
resource "aws_instance" "webserver" {
    ami = "ami-0edab43b6fa892279"
    instance_type = "t2.micro"
    tags = {
        Name = "webserver"
        Description = "An Nginx WebServer on Ubuntu"
    }
    provisioner "local-exec" {
        command = "echo ${aws_instance.webserver .public_ip} created >> /tmp/ips.txt"
    }

    provisioner "local-exec" {
        when = destroy
        command = "echo ${aws_instance.webserver .public_ip} destroyed >> /tmp/ips.txt"
    }

    key_name = aws_key_pair.web.id
    vpc_security_group_ids = [ aws_security_group.ssh-access.id ]
}
```

## Taints

If a resource fails to create via terraform apply, TF will mark the resource with a taint. In this case, terraform will recreate the entire resource in a subsequent apply, even if some of the infra for the particular resource was successfully created.

You can use taints in cases where you want resources to be recreated. You can taint a particular resource using the CLI. Example: `terraform taint aws_instance.webserver`. If you would like to remove a taint, use untaint. Example: `terraform untaint aws_instance.webserver`.

## Debugging

Change log level from TF using environment variables. Example: `export TF_LOG=<log level>`. Following log levels are available:

- INFO
- WARNING
- ERROR
- DEBUG
- TRACE

Logs can be dumped into a particular path with the env var `TF_LOG_PATH=<filepath>`.

## Import

If you would like to bring resources that were not created by TF to be TF managed, you can use `import`. Example:

`terraform import <resource_type>.<resource_name> <attribute>`
`terraform import aws_instance.webserver-2 i-026e13b310d5326f7`

When doing this via CLI, you also need to declare this resource within your configuration file. You can leave it blank initially, then add the arguments after importing. Pull these arguments using `terraform show`.

```
resource "aws_instance" "webserver-2" {
    # leave empty
}
```

## Modules

Configuration directories can be packaged into reusable modules. Modules can be referenced in a config file with the following syntax:

```
module "dev-webserver" {
    source = "../path-to-module"
}
```

Overrides for variables can be provided within the module reference:

```
module "us_payroll" {
    source = "../modules/payroll-app"
    app_region = "us-east-1"
    ami = "ami-24e140119877avm"
}
```

The above is an example of a local module - exists in some local directory. Modules can exist in remote stores, such as [the official Terraform registry](https://registry.terraform.io/browse/modules). Here is an example module usage of the official AWS security group module from Hashicorp.

```
module "security-group-ssh" {
    source = "terraform-aws-modules/security-group/aws/modules/ssh"
    version = "3.16.0"
    vpc_id = "vpc-1234567"
    ingress_cidr_blocks = ["0.0.0.0./0"]
    name = "ssh-access"
}
```

Once you have included a module, run `terraform init` or `terraform get` in order to fetch the files you'll need to run the module.

## Workspaces

Workspaces allow you to create manage state for multiple environments within the same configuration directory. Create new workspaces with `terraform workspace new <Workspace>`, and switch between them using `terraform workspace select ProjectA`.

You can use the workspace name to conditionally select between different values in your variables. For instance, let's say we have workspaces `ProjectA` and `ProjectB`:

```
variable region {
    default = "us-east-1"
}

variable instance_type {
    default = "t2.micro"
}

variable ami {
    type = map
    default = {
        "ProjectA" = "ami-0edab43b6fa892279",
        "ProjectB" = "ami-0c2f25c1ff66a1ff4d"
    }
}

resource "aws_instance" "server" {
    ami = lookup(var.ami, terraform.workspace)
    instance_type = var.instance_type
    tags = {
        Name = terraform.workspace
    }
}
```

State for different workspaces will be stored in a directory called `terraform.tfstate.d` in separate subfolders based on the workspace name.

## Misc functions

Other useful functions:

- `max(-1, 2, -10, 200, -25)`
- `max(var.array...)` - how to use this with an array value
- `ceil(num)` and `floor(num)`
- `split(",", "abc,xyz,efg")` - outputs a list ["abc", "xyz", "efg"]
- `lower(str)`, `upper(str)`, `title(str)`
- `substr(var.str, 0, 7)` - string, offset, substring length
- `index(var.array, "INDEX")` - outputs index of an element list
- `element(var.array, 2)` - outputs the element at that index
- `contains(var.array, "ABC")` - true/false if value in array
- `keys(var.map)` and `values(var.map)`
- `lookup(var.map, "ABC")` - get the value for a particular key

Conditional expressions:
`length = var.length < 8 ? 8 : var.length`
