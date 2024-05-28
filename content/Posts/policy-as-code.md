---
title: "Policy as Code: Cloudformation Guard"
date: 2024-05-28
author: Yadav Lamichhane
description: "Policy as Code: Cloudformation Guard"
tags:
- Linux
---

CloudFormation Guard (cfn-guard) is a tool developed by AWS to validate CloudFormation templates against a set of predefined rules. It ensures that your infrastructure as code (IaC) complies with organizational policies, security requirements, and best practices before deployment.

## Key Features

1. **Policy-as-Code**: Define reusable and version-controlled rules.
2. **Validation**: Check CloudFormation templates against these rules.
3. **Compliance**: Enforce security, cost management, and operational guidelines.
4. **Integration**: Seamlessly integrates with CI/CD pipelines for automated checks.

## Installing CloudFormation Guard

You can install CloudFormation Guard using AWS CLI or download it from the [releases page](https://github.com/aws-cloudformation/cloudformation-guard/releases).

### Using AWS CLI

```
pip install cfn-guard
```

### Downloading Binary

Download the appropriate binary for your operating system from the [releases page](https://github.com/aws-cloudformation/cloudformation-guard/releases) and add it to your PATH.

## Writing Guard Rules

Guard rules are written in a simple, declarative language. Here’s an example rule to ensure that S3 buckets have versioning enabled:

```
let s3_buckets = Resources.*[ Type == "AWS::S3::Bucket" ]

rule check_s3_versioning {
    s3_buckets.Properties.VersioningConfiguration.Status == "Enabled"
}
```

## Validating a Template

To validate a CloudFormation template, use the cfn-guard validate command with the rule set and template file.

```
cfn-guard validate --rules rules.guard --data bucket.yaml
```

## **Use Case**

## Object Encryption and Lifecycle policies for s3 bucket

```
Resources:
  DevOpsLearningBucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      LifecycleConfiguration:
        Rules:
          - Id: ExpirationDays
            Status: 'Enabled'
            ExpirationInDays: 30
      BucketName: devops-learning-101
      BucketEncryption:
        ServerSideEncryptionConfiguration:
        - ServerSideEncryptionByDefault:
            SSEAlgorithm: aws:kms
      PublicAccessBlockConfiguration:
        BlockPublicAcls: TRUE
        BlockPublicPolicy: TRUE
        IgnorePublicAcls: TRUE
        RestrictPublicBuckets: TRUE
      Tags:
        - Key: Name
          Value: !Join ['-', [ !Ref Environment, 's3', 'DevOpsLearning101' ]]
    DeletionPolicy: Delete
```

**Guard Rules**

```
let s3_buckets = Resources.*[ Type == 'AWS::S3::Bucket' ]
let allowed_algos = ["aws:kms", "AES256"]

rule s3_buckets_allowed_sse_algorithm when %s3_buckets !empty {
    let encryption = %s3_buckets.Properties.BucketEncryption
    %encryption exists
    %encryption.ServerSideEncryptionConfiguration[*].ServerSideEncryptionByDefault.SSEAlgorithm in %allowed_algos
}


let lifecycleRules = Resources.*[ Type == 'AWS::S3::Bucket' ].Properties.LifecycleConfiguration.Rules

rule check_lifecycle_status {
    # Ensure lifecycle rules are enabled
    %lifecycleRules.*.Status == 'Enabled'
}

rule check_lifecycle_expiration {
    %lifecycleRules.*.ExpirationInDays == 30
}
```

### Validating the Template

```
cfn-guard validate --rules rules.guard --data bucket.yaml -S pass
```

### Output

```
❯ cfn-guard validate --rules rules.guard --data bucket.yaml -S pass
/Users/yadavlamichhane/cf/repos/aggregated-data-subsystem/comp/buckets/snowflake-stage/bucket.yaml Status = PASS
PASS rules
rules.guard/s3_buckets_allowed_sse_algorithm    PASS
rules.guard/check_lifecycle_status              PASS
rules.guard/check_lifecycle_expiration          PASS

```

## Conclusion

CloudFormation Guard is a powerful tool to ensure that your CloudFormation templates meet the required policies and standards. By integrating it into your CI/CD pipelines, you can automate compliance checks and prevent non-compliant infrastructure from being deployed. For more information, refer to the [official documentation](https://github.com/aws-cloudformation/cloudformation-guard).
