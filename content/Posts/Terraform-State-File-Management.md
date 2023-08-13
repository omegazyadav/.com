---
title: "Multi-region Terraform State Files"
date: "2021-09-09"
author: "Yadav Lamichhane"
keywords: "Terrafrom"
summary: "Article on how to manage multi-region deployment of the services with terraform"
---
---

On this blog post I will be discussing about the management of multi-region terraform state files in AWS Cloud which is desired approach for the design of fault tolerant infrastructure architecture.

---

## Backend Configuration

By default terraform manages the state file named `terraform.tfstate` on your local directory where the terraform resource files are present. Same state files can be stored remotely and considered as a good approach if we are  working in a team and managing state files through CI/CD pipelines.

This S3 backend supports the state locking and consistency checking with the help of DynamoDB table.

```
terraform {
  backend "s3" {
    bucket         = "test-bucket"
    key            = "omegazyadav"
    region         = "ap-south-1"
    dynamodb_table = "test-dynamodb-table"
  }
}
```

Human errors are inevitable, accidental deletion or corruption might occur so in order to recover the state files in such incidents, it is highly recommended to enable bucket versioning.

## Basic Setup

![Multi-region Terraform State File Architecture](https://i.imgur.com/DcChm1u.png)

The above mentioned backend configuration would only allows us to store the terraform state files within `ap-south-1` region. What if we want to store the state file independently across the multiple region like shown in the above diagram ?

In order to achieve this we need to restructure our existing terraform resources which would look something like this.

```
     ├── ap-south-1
     │   ├── backend.tf
     │   ├── output.tf
     │   ├── main.tf
     │   ├── providers.tf
     │   └── variables.tf
     ├── us-west-2
     │   ├── backend.tf
     │   ├── output.tf
     │   ├── main.tf
     │   ├── providers.tf
     │   └── variables.tf
     ├── modules
     │       └─ aws-s3-bucket
     │               ├── main.tf
     │               ├── outputs.tf
     │               ├── variables.tf
     │               └── www
     └── README.md
```
As in the above directory structure, we can separate out the terraform resources like backend configuration, environmental variables, provider, root modules, output and input variables based on the deployment region. Here the backend details of the `ap-south-1` is different than that of `eu-central-1` region. This will allows us to manage terraform state file across the deployment region independently which allows multiple team can work together concurrently with the same terraform resources.

## Let's get started!

In this example setup, I will be hosting a static website with the help of terraform. Here I have logically separated the terraform resources files based on the deployment region in which the terraform state files are also managed independently.

### Child Modules

Child modules are a kind of module which is being called by root or other module. These modules contains the resources files, input output variables, etc. which are required by the deployment region.

In this example we have aws-s3-bucket directory which holds the terraform resources for the aws s3 bucket to host a static website. The directory structure of the child module looks as follows.

```
modules
└── aws-s3-bucket
    ├── main.tf
    ├── outputs.tf
    ├── variables.tf
    └── www
```

### Deployment Region - `ap-south-1`

The `ap-south-1` directory contains the terraform resources required for the deployment of services in `ap-south-1` region.

- backend.tf

  `backend.tf` file contains the remote state configuration as described earlier.

  ```
  terraform {
    backend "s3" {
      region = "ap-south-1"
      bucket = "test-s3-bucket-1"
      key    = "omegazyadav"
      dynamodb_table = "test-dynamodb-table-1"
    }
  }
  ```
- provider.tf

  This resources interact with the AWS Cloud API in order to provision the requested resources based on the authentication provided.
  ```
  provider "aws" {
    region = "ap-south-1"
  }
  ```

- variables.tf

  These are the input variables which is referred by the other resources while defining it. This allows us to customize the behavior without editing the source.
  ```
    variable "s3_bucket_name" {
      default = "test-s3-bucket"
    }
  ```

- main.tf

  This is the root module for the `ap-south-1` region. All the resources which are defined in the child modules are referenced here.

  ```
  module "static_website" {
    source = "../modules/aws-s3-bucket"

    bucket_name = var.s3_bucket_name

    tags = {
      Terraform   = "true"
      Environment = "test"
    }
  }
  ```

- output.tf

  This resources returns the values for the terraform modules when provisioning of the defined resources are completed. Here buckets endpoint and url for the static website are configured as an output.
  ```
  output "website_bucket_arn" {
    description = "ARN of the bucket"
    value       = module.static_website.arn
  }

  output "website_bucket_name" {
    description = "Name (id) of the bucket"
    value       = module.static_website.name
  }

  output "website_endpoint" {
    description = "Domain name of the bucket"
    value       = module.static_website.website_endpoint
  }
  ```

Since all the resources are configured, now we can plan the root modules in order to visualize the resources about to be provisioned.

![Terraform Plan ap-south-1](https://i.imgur.com/CtRf3T1.png)

While executing the plan, terraform will
1. Read the current state of any already existing remote objects to make sure the Terraform state is up-to-date.
2. Comparing the current configuration to the prior state and noting any differences
3. Proposing a set of changes that should, if applied, make the remote objects match the configuration.

Now we can proceed to apply the proposal plan. We can simply execute `terraform apply` command and terraform will provision the requested resources as shown in the plan output.

![Terraform Apply ap-south-1](https://i.imgur.com/8joCiuD.png)


### Deployment Region - `us-west-2`

Similarly same configuration is replicated in the `us-west-2` region but the contents of `backend.tf` and `proivder.tf` file should be different as it holds the information of different remote state files and default region for the deployment respectively.

- backend.tf
  ```
  terraform {
    backend "s3" {
      region = "us-west-2"
      bucket = "test-s3-bucket-2"
      key    = "omegazyadav"
      dynamodb_table = "test-dynamodb_table"
    }
  }
  ```
- provider.tf
  ```
  provider "aws" {
    region = "us-west-2"
  }
  ```

## Conclusion

Our application should be scaling, so does our infrastructure. Working with the terrafrom can be stressful if we are depending on the single state file
which can be point of failure. Through this approach we can manage numerous terraform state files and which can be carried out for multi-region deployment process.

