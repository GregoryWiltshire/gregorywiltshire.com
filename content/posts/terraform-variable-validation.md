---
title: "Abusing Terraform Variable Validations for Science"
date: 2023-06-01T19:31:34-04:00
draft: true
author: "Gregory Wiltshire"
description: ""
tags: ["Terraform", "Testing"]
---

One task I've recently encountered was refactoring terraform configurations that had significantly drifted across environments. 

```shell
➜  diff ./prod ./dev --exclude terraform.tfvars --brief | wc -l
      58
```
_oh my..._

The prod environment naturally has a tendency to include more resources or policies than the dev environment. Luckily the mental overhead of having two unique configurations is unnecessary.

In principle, Terraform is a declarative configuration language. It employs several providers which abstract the more imperative elements such as error checking, validation, retries etc for creating your final requested configuration state.

In practice, there are many reasons why you may find yourself saying "this is not my beautiful declarative infra!".

Imperative constructs have crept in to the implemenation of Terraform over time with the addition of [dynamic blocks](https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks), or [meta-arguments](https://developer.hashicorp.com/terraform/language/meta-arguments/count) like count and for_each and while they may not be as expressive as a fully fledged programming language, they offer a reasonable amount of flexibility.

The reason I bring up these control flow constructs is that by using them our environments can have identical configuration where the only differences are their inputs: `terraform.tfvars`.

Consider the following s3 policy which includes a dynamic block that is only present in a "prod" environment.
```terraform
    dynamic "statement" {
        # ternary expression to conditionally include
        # the following statement if environment == "prod"
        for_each = var.environment == "prod" ? [1] : []
        content {
            sid = "allow access to prefix"
            principals {
                type        = "AWS"
                identifiers = [
                    "arn:aws:iam::012345678910:root",
                ]
            }
            effect = "Allow"
            actions = [
                "s3:GetObject",
            ]
            resources = [
                "${aws_s3_bucket.bucket.arn}/prefix/*"
            ]
        }
    }
```

Or the use of a boolean variable to accomplish toggling a resource/module.

```terraform
    module "service_user"{
        count = var.service_user ? 1 : 0
        source = "../../modules/service-role"
        role_name = local.service_user_role
        account = "012345678910"
        tags = var.tags
        s3_buckets = [
            aws_s3_bucket.bucket.arn
        ]
    }
``` 

Controlling these differences between environments with variables is very useful, it allows your terraform to be promotable across environments just by syncing two folders and viewing the resulting plan.

After much refactoring, and decomposing the control of my environments into their respective terraform.tfvars files I wanted to see if there was a way to discourage my IAC from further drifting.
Is there a way to test if dev and prod config files are the same?

That was definitely a leading question, the anser is Yes of course. One could certainly write tests and hook them up to pre-commit git hooks, but there is also native terraform construct to accomplish a validation like this.

Terraform supports [validation rules](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules) which are meant to validate input variables, however they can be used as a crude assert for any arbitrary expression.

So, what bag of expressive tricks does Terraform afford us?

Well, we're going to require listing files.
```shell
> fileset("../dev", "*.tf")
toset([
  "api-gateway.tf",
  "aurora-postgres.tf",
  "autoscale-perf.tf",
  "bastion.tf",
  "cert.tf",
  ...
  "waf.tf",
  "worker-queue-autoscaling.tf",
  "worker.tf",
])
```

To see the text content of files, and the hash of said text.
```shell
> file("../prod/hello.tf")
<<EOT
Hello!

EOT
> sha1(file("../prod/hello.tf"))
"a8d191538209e335154750d2df575b9ddfb16fc7"
```

If we recall set theory from way back in Discrete Math...
We can find the disjoint union of our filesets, or all the files that exist uniquely to each set.

```shell
➜  find . -type f
./prod/foo.tf
./prod/bizz.tf
./dev/buzz.tf
./dev/bar.tf
./dev/bizz.tf
```

This is equivalent to each set, minus their intersection.  
Here we can find the files missing from ../dev:
```terraform
> setsubtract(fileset("../dev", "*.tf"),setintersection(fileset("../dev", "*.tf"),fileset("../prod", "*.tf")))
toset([
  "bar.tf",
  "buzz.tf",
])
```
and ../prod:
```terraform
setsubtract(fileset("../prod", "*.tf"),setintersection(fileset("../prod", "*.tf"),fileset("../dev", "*.tf")))
toset([
  "foo.tf",
])
```

And this is usually where I would try to put a lot of these expressions into a [locals](https://developer.hashicorp.com/terraform/language/values/locals) block to simplify my expressions and make them easier on the eyes, but one caveat to this particular use-case is all of the validation and variable collection is done before any blocks are parsed. In addition, variable validations can only refer to their respective variable, and you would see an error like this if you tried to reference any other variables.
```shell
│ The error message for variable "test_iac_same_files" can only refer to the variable itself, using var.test_iac_same_files.
```

So, putting it all together: some set theory, a rats nest of expressions, and a dash of useful error messaging...

```shell
│ Error: Invalid value for variable
│ 
│   on tests.tf line 13:
│   13: variable "test_iac_same_files" {
│     ├────────────────
│     │ var.test_iac_same_files is true
│ 
│ missing from prod:
│ ["ecs-service.tf","mwaa.tf"]
│ 
│ missing from dev:
│ ["_ssm-inbound-aspera-on-cloud.tf","api-gateway-keys.tf","extractor-secrets.tf","role-dete.tf","role-inbound-aspera-on-cloud.tf","s3-hbo-max.tf","s3-inbound-aspera-on-cloud-logs.tf","s3-inbound-aspera-on-cloud.tf","veritone-postrelease.tf","veritone-prerelease.tf"]
│ 
│ files differing:
│ ["aurora-postgres.tf","cert.tf","cron-cnn.tf","data.tf","ecs-event-stream.tf","ecs.tf","main.tf","role.tf","s3-videos.tf","tests.tf"]
│ 
│ This was checked by the validation rule at tests.tf:16,3-13.
```
Now that's nice, I can see exactly which files are missing and which files have changes.
To keep my IAC exactly the same between environments I now have a basic validation which will run every time, that gives useful error messages and requires no extra hooks or tooling.

To wrap up this is what my configuration looks like.
```terraform
variable "test_iac_same_files" {
  description = "prod and dev IAC must have identical files"
  default = true
  validation {
    condition     =  !var.test_iac_same_files || (fileset("../dev", "*.tf") == fileset("../prod", "*.tf")) || length([for f in setintersection(fileset("../dev", "*.tf"),fileset("../prod", "*.tf")): f if sha1(file("../dev/${f}")) != sha1(file("../prod/${f}"))]) == 0
    error_message ="missing from prod:\n${jsonencode(setsubtract(fileset("../dev", "*.tf"),setintersection(fileset("../dev", "*.tf"),fileset("../prod", "*.tf"))))}\n\nmissing from dev:\n${jsonencode(setsubtract(fileset("../prod", "*.tf"),setintersection(fileset("../dev", "*.tf"),fileset("../prod", "*.tf"))))}\n\nfiles differing:\n${jsonencode([for f in setintersection(fileset("../dev", "*.tf"),fileset("../prod", "*.tf")): f if sha1(file("../dev/${f}")) != sha1(file("../prod/${f}"))])}"
  }
}
```

Finally, the validation can be entirely bypassed simply by overriding the variable itself to false:
```shell
tf plan -var 'test_iac_same_files=false'
```

# Citations/Recommended Reading
1. https://ordina-jworks.github.io/cloud/2023/06/05/back-to-terraform.html
2. https://blog.container-solutions.com/is-it-imperative-to-be-declarative
