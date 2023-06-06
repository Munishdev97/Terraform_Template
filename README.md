# Terraform Best Practices 🌐

Terraform Best Practices for AWS users.

Welcome! This repo contains a set of best practices to be followed when contributing to the terraform projects.

## Table of Contents
  - [structuring](#structuring)
  - [Naming-conventions](#naming-conventions)
  - [Run terraform command with var-file](#run-terraform-command-with-var-file)
  - [TFLint](#tflint)
  - [Minimize Blast Radius](#minimize-blast-radius)
  - [Enable version control on terraform state files bucket](#enable-version-control-on-terraform-state-files-bucket)
  - [Manage S3 backend for tfstate files](#manage-s3-backend-for-tfstate-files)
  - [Retrieve state meta data from a remote backend](#retrieve-state-meta-data-from-a-remote-backend)
  - [Use shared modules](#use-shared-modules)
  - [Use terraform import to include as many resources you can](#use-terraform-import-to-include-as-many-resources-you-can)
  - [Avoid hard coding the resources](#avoid-hard-coding-the-resources)
  - [validate and format terraform code](#validate-and-format-terraform-code)
  - [Generate README for each module with input and output variables](#generate-readme-for-each-module-with-input-and-output-variables)
  - [Update terraform version](#update-terraform-version)
  - [Run terraform in docker container](#run-terraform-in-docker-container)
  - [Usage of variable "self"](#usage-of-variable-self)
  - [One more use case](#one-more-use-case)
  - [Terraform Compliance](#terraform-compliance)
  - [TERRASPACE](#terraspace)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

> The README for terraform version 0.11 and less has been renamed to [README.0.11.md](README.0.11.md)


## structuring
When you are working on a large production infrastructure project using Terraform, you must follow a proper directory structure to take care of the complexities that may occur in the project. It would be best if you had separate directories for different purposes.
   
- Example directory staructure :
```
├── dev
│ ├── main.tf
│ ├── outputs.tf
│ └── variables.tf
├── modules
│ ├── ec2
│ │ ├── ec2.tf
│ │ └── main.tf
│ └── vpc
│ ├── main.tf
│ └── vpc.tf
├── prod
│ ├── main.tf
│ ├── outputs.tf
│ └── variables.tf
└── stg
├── main.tf
├── outputs.tf
└── variables.tf
```


## Naming conventions
- Use _ (underscore) instead of - (dash) everywhere (in resource names, data source names, variable names, outputs, etc).
- Prefer to use lowercase letters and numbers (even though UTF-8 is supported).
- Resource name should be named this if there is no more descriptive and general name available, or if the resource module creates a single resource of this type (eg, in AWS VPC module there is a single resource of type aws_nat_gateway and multiple resources of typeaws_route_table, so aws_nat_gateway should be named this and aws_route_table should have more descriptive names - like private, public, database).
- Always use singular nouns for names.
- Use - inside arguments values and in places where value will be exposed to a human (eg, inside DNS name of RDS instance).
- Include argument count / for_each inside resource or data source block as the first argument at the top and separate by newline after it.
- Include argument tags, if supported by resource, as the last real argument, following by depends_on and lifecycle, if necessary. All of these   should be separated by a single empty line.
- When using conditions in an argumentcount / for_each prefer boolean values instead of using length or other expressions.


## Run terraform command with var-file

```bash
$ cat config/dev.tfvars

name = "dev-stack"
s3_terraform_bucket = "dev-stack-terraform"
tag_team_name = "hello-world"

$ terraform plan -var-file=config/dev.tfvars
```

With `var-file`, you can easily manage environment (dev/stag/uat/prod) variables.

With `var-file`, you avoid running terraform with long list of key-value pairs ( `-var foo=bar` )

## Enable version control on terraform state files bucket

Always set backend to s3 and enable version control on this bucket.

[s3-backend](s3-backend) to create s3 bucket and dynamodb table to use as terraform backend.

## TFLint

- TFLint is a framework and each feature is provided by plugins, the key features are as follows:

- Find possible errors (like illegal instance types) for Major Cloud providers (AWS/Azure/GCP).
- Warn about deprecated syntax, unused declarations.
- Enforce best practices, naming conventions.
- Installation on Linux :
  $ curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
- Installation on Windows :  -$ choco install tflint
(Run TFLint is simple:tflint)
Getting Started
Terraform is a great tool for Infrastructure as Code. However, many of these tools don't validate provider-specific issues. For example, see the following configuration file:
```
resource "aws_instance" "foo" {
  ami           = "ami-0ff8a91507f77f867"
  instance_type = "t1.2xlarge" # invalid type!
}
```
Since t1.2xlarge is an invalid instance type, an error will occur when you run terraform apply. But terraform validate and terraform plan cannot find this possible error in advance. That's because it's an AWS provider-specific issue and it's valid as the Terraform Language.

The goal of this ruleset is to find such errors:
By running TFLint with this ruleset in advance, you can fix the problem before the error occurs in production CI/CD pipelines.

## Minimize Blast Radius

The blast radius is nothing but the measure of damage that can happen if things do not go as planned.

For example, if you are deploying some terraform configurations on the infrastructure and the configuration do not get applied correctly, what will be the amount of damage to the infrastructure.

So, to minimize the blast radius, it is always suggested to push a few configurations on the infrastructure at a time. So, if something went wrong, the damage to the infrastructure will be minimal and can be corrected quickly. Deploying plenty of configurations at once is very risky.

## Manage S3 backend for tfstate files

Terraform doesn't support [Interpolated variables in terraform backend config](https://github.com/hashicorp/terraform/pull/12067), normally you write a seperate script to define s3 backend bucket name for different environments, but I recommend to hard code it directly as below. This way is called as [partial configuration](https://www.terraform.io/docs/backends/config.html#partial-configuration)

Add below code in terraform configuration files.

```bash
$ cat main.tf

terraform {
  backend "s3" {
    encrypt = true
  }
}
```

Define backend variables for particular environment

```bash
$ cat config/backend-dev.conf
bucket  = "<account_id>-terraform-states"
key     = "development/service-name.tfstate"
encrypt = true
region  = "ap-southeast-2"
#dynamodb_table = "terraform-lock"
```
- `bucket` - s3 bucket name, has to be globally unique.
- `key` - Set some meaningful names for different services and applications, such as vpc.tfstate, application_name.tfstate, etc
- `dynamodb_table` - optional when you want to enable [State Locking](https://www.terraform.io/docs/state/locking.html)

After you set `config/backend-dev.conf` and `config/dev.tfvars` properly (for each environment). You can easily run terraform as below:

```bash
env=dev
terraform get -update=true
terraform init -reconfigure -backend-config=config/backend-${env}.conf
terraform plan -var-file=config/${env}.tfvars
terraform apply -var-file=config/${env}.tfvars
```
If you encountered any unexpected issues, delete the cache folder, and try again.
```
rm -rf .terraform
```

## Retrieve state meta data from a remote backend

Normally we have several layers to manage terraform resources, such as network, database, application layers. After you create the basic network resources, such as vpc, security group, subnets, nat gateway in vpc stack. Your database layer and applications layer should always refer the resource from vpc layer directly via `terraform_remote_state` data srouce.

> Notes: in Terraform v0.12+, you need add extra `outputs` to reference the attributes, otherwise you will get error message of [Unsupported attribute](https://github.com/hashicorp/terraform/issues/21442)

```terraform
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = var.s3_terraform_bucket
    key    = "${var.environment}/vpc.tfstate"
    region = var.aws_region
  }
}

# Retrieves the vpc_id and subnet_ids directly from remote backend state files.
resource "aws_xx_xxxx" "main" {
  # ...
  subnet_ids = split(",", data.terraform_remote_state.vpc.data_subnets)
  vpc_id     = data.terraform_remote_state.vpc.outputs.vpc_id
}
```


## Use shared modules

Manage terraform resource with shared modules, this will save a lot of coding time. No need re-invent the wheel!

You can start from below links:

- [Terraform module usage](https://www.terraform.io/docs/modules/usage.html)

- [Terraform Module Registry](https://registry.terraform.io/)

- [Terraform aws modules](https://github.com/terraform-aws-modules)

> Up to Terraform 0.12, Terraform modules didn't support `count` parameter.
>
> From Terraform 0.13 on this feature is already available for your pleasure!


```terraform
variable "application" {
  description = "application name"
  default = "<replace_with_your_project_or_application_name, use short name if possible, because some resources have length limit on its name>"
}

variable "environment" {
  description = "environment name"
  default = "<replace_with_environment_name, such as dev, svt, prod,etc. Use short name if possible, because some resources have length limit on its name>"
}

locals {
  name_prefix    = "${var.application}-${var.environment}"
}

resource "<any_resource>" "custom_resource_name" {
  name = "${local.name_prefix}-<resource_name>"
  ...
}
```

With that, you will easily define the resource with a meaningful and unique name, and you can build more of the same application stack for different developers without change a lot. For example, you update the environment to dev, staging, uat, prod, etc.

> Tips: some aws resource names have length limits, such as less than 24 characters, so when you define variables of application and environment name, use short name.

## Use terraform import to include as many resources you can

Sometimes developers manually created resources. You need to mark these resource and use `terraform import` to include them in codes.

[terraform import](https://www.terraform.io/docs/import/usage.html)

## Avoid hard coding the resources

A sample:

```terraform
account_number=“123456789012"
account_alias="mycompany"
region="us-east-2"
```

The current aws account id, account alias and current region can be input directly via [data sources](https://www.terraform.io/docs/providers/aws/).

```terraform
# The attribute `${data.aws_caller_identity.current.account_id}` will be current account number.
data "aws_caller_identity" "current" {}

# The attribue `${data.aws_iam_account_alias.current.account_alias}` will be current account alias
data "aws_iam_account_alias" "current" {}

# The attribute `${data.aws_region.current.name}` will be current region
data "aws_region" "current" {}

# Set as [local values](https://www.terraform.io/docs/configuration/locals.html)
locals {
  account_id    = data.aws_caller_identity.current.account_id
  account_alias = data.aws_iam_account_alias.current.account_alias
  region        = data.aws_region.current.name
}
```

## validate and format terraform code

Always run `terraform fmt` to format terraform configuration files and make them neat.

I used below code in Travis CI pipeline (you can re-use it in any pipelines) to validate and format check the codes before you can merge it to master branch.

```yml
script:
  - terraform validate
  - terraform fmt -check=true -write=false -diff=true
  - <rest terraform commands>
```

One more check [tflint](https://github.com/wata727/tflint) you can add

```yml
- find . -type f -name "*.tf" -exec dirname {} \;|sort -u |while read line; do pushd $line; docker run --rm -v $(pwd):/data -t wata727/tflint; popd; done
```

## Generate README for each module with input and output variables

You needn't manually manage `USAGE` about input variables and outputs. A tool named `terraform-docs` can do the job for you.
```
-Windows :
If you are a Windows user:

Scoop
You can use Scoop.

scoop bucket add terraform-docs https://github.com/terraform-docs/scoop-bucket
scoop install terraform-docs
-Chocolatey
or you can use Chocolatey.

choco install terraform-docs
```

Now we have a work around.

```bash
# [Terraform >= 0.12]
docker run --rm \
  -v $(pwd):/data \
  cytopia/terraform-docs \
  terraform-docs-012 --sort-inputs-by-required --with-aggregate-type-defaults md . > README.md
```

For details on how to run `terraform-docs`, check this repository: <https://github.com/terraform-docs/terraform-docs>
 or follow : <https://terraform-docs.io/user-guide>

There is a simple sample for you to start [tf_aws_acme](https://github.com/BWITS/tf_aws_acme), the README is generatd by `terraform-docs`

## Update terraform version

Hashicorp doesn't have a good qa/build/release process for their software and does not follow semantic versioning rules.

For example, `terraform init` isn't compatible between 0.9 and 0.8. Now they are going to split providers and use "init" to install providers as plugin in coming version 0.10

So recommend to keep updating to latest terraform version



## Run terraform in docker container

Terraform releases official docker containers that you can easily control which version you can run.

Recommend to run terraform docker container, when you set your build job in CI/CD pipeline.

```terraform
TERRAFORM_IMAGE=hashicorp/terraform:0.12.3
TERRAFORM_CMD="docker run -ti --rm -w /app -v ${HOME}/.aws:/root/.aws -v ${HOME}/.ssh:/root/.ssh -v `pwd`:/app -w /app ${TERRAFORM_IMAGE}"
${TERRAFORM_CMD} init
${TERRAFORM_CMD} plan
```

## Usage of variable "self"

Quote from terraform documents:

```log
Attributes of your own resource

The syntax is self.ATTRIBUTE. For example \${self.private_ip} will interpolate that resource's private IP address.

Note: The self.ATTRIBUTE syntax is only allowed and valid within provisioners.
```

### One more use case

```terraform
resource "aws_ecr_repository" "jenkins" {
  name = var.image_name
  provisioner "local-exec" {
    command = "./deploy-image.sh ${self.repository_url} ${var.jenkins_image_name}"
  }
}

variable "jenkins_image_name" {
  default = "mycompany/jenkins"
  description = "Jenkins image name."
}
```

You can easily define ecr image url (`<account_id>.dkr.ecr.<aws_region>.amazonaws.com/<image_name>`) with \${self.repository_url}

Any attributes in this resource can be self referenced by this way.

Reference: <https://github.com/shuaibiyy/terraform-ecs-jenkins/blob/master/docker/main.tf>

## Terraform Compliance

- terraform-compliance is a lightweight, security and compliance focused test framework against terraform to enable negative testing capability for your infrastructure-as-code.

- compliance: Ensure the implemented code is following security standards, your own custom standards
- behaviour driven development: We have BDD for nearly everything, why not for IaC ?
- portable: just install it from pip or run it via docker. See Installation
- pre-deploy: it validates your code before it is deployed
- provider agnostic: it works with any provider
- easy to integrate: it can run in your pipeline (or in git hooks) to ensure all deployments are validated
- segregation of duty: you can keep your tests in a different repository where a separate team is responsible

Installation

- To install terraform-compliance it’s very simple, have 2 ways (docker or pip), in this tutorial we are using pip, just run
```
$ pip install terraform-compliance

```
-Installing/Using via Docker
Every new release of terraform-compliance is stored in Docker Hub.

You can either download this docker image directly and try to use it, or instead you can create a shell command to do this for you easily.

Here is the function for that :
```
[~] $ function terraform-compliance { docker run --rm -v $(pwd):/target -i -t eerkunt/terraform-compliance "$@"; }
Whenever there is a new release or if you want to upgrade it to the latest, you can run ;

[~] $ docker pull eerkunt/terraform-compliance
which will download the latest docker image.

```
- Reference doc <https://terraform-compliance.com/>





## TERRASPACE

- Install terraspace via RubyGems.

- First install gem
  $ gem install terraspace

- Configure AWS with Terraspace 
 
  set up the ~/.aws/config and ~/.aws/credentials files
  set up your AWS_PROFILE and AWS_REGION environment variables
 
- $ vim ~/.aws/config
[profile dev]
output = json
region = us-west-2
 
- $ vim ~/.aws/credentials
[dev]
aws_access_key_id = REPLACE_ME
aws_secret_access_key = REPLACE_ME
 
- In your ~/.bashrc or ~/.profile, use this line to set the AWS_PROFILE 

- $ vim ~/.bashrc 
  export AWS_PROFILE=dev
  export AWS_REGION=`aws configure get region` # to match what's in ~/.aws/config

- Here are some useful commands to test that the AWS CLI is working:

   $ aws sts get-caller-identity

- terraspace new project to generate a new terraspace project.
 
   $ terraspace new project hotplay-games --plugin aws
 
- The terraspace up command will build the files and then essentially run terraform apply.

   $ terraspace all up / $ terraspace up stack name (for run separate stack only)
 
   $ terraspace tree (it will show the folder structure)
 
   $ terraspace all down (deleting the infra)
 
- Cd  /project folder name 
- Then use  $ terraspace all up command (This command will work only inside the project folder )
 
Ref - https://terraspace.cloud/docs/learn/aws/configure/
 
- For creating tf file cmd is - $ terraspace seed network

- The terraspace seed command creates starter tfvars files for different environments. Examples

   TS_ENV=prod terraspace seed network
- Creates starter tfvars files for different regione :
   AWS_REGION=us-east-1 TS_ENV=prod "terraspace seed stack name" 
- For running the script use :
   AWS_REGION=us-east-1 TS_ENV=prod "terraspace up stack name"



