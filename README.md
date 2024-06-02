# terraform-terragrunt-null-module

The goal of this module is to allow [Terragrunt](https://terragrunt.gruntwork.io/) to place generated Terraform files from the Terragrunt's [generate](https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/#generate) configuration block into the [Terragrunt cache directory](https://terragrunt.gruntwork.io/docs/features/caching/#clearing-the-terragrunt-cache), instead of the same directory where the [terragrunt.cl](https://terragrunt.gruntwork.io/docs/getting-started/configuration/#terragrunt-configuration-file) file lives as it happens when we use Terragrunt only to generate Terraform resources or data sources on the fly instead of using a fully-fleged Terraform module.

## Example on how to use this module with Terragrunt

1. Say you need to generate some Terraform resources or data sources on the fly without the need to call a fully-fleged Terraform module, for this example we will fetch 3 AWS AZs from the current AWS's region (not shown in the example as it's declared on the region_commons.hcl file) to be later feed on another Terraform module called by Terragrunt:

    ```hcl
    terraform {
      source = "tfr:///yiskaneto/null-module/terragrunt?version=<tag version>"
    }

    include "root" {
      path = find_in_parent_folders()
    }

    include "region_common" {
      path   = find_in_parent_folders("region_commons.hcl")
      expose = true
    }

    <optional Terragrunt's configuration goes here>

    ## Replace
    generate "aws_az" {
      path      = "aws_az.tf"
      if_exists = "overwrite_terragrunt"
      contents  = <<EOF
        data "aws_availability_zones" "available" {}

        locals {
          azs = slice(data.aws_availability_zones.available.names, 0, 3)
        }

        output "azs" {
          value = local.azs
        }

    EOF
    }
    ```

1. Now, we need to properly reference the "azs" output value  to the Terragrunt configuration file that calls the [Terraform VPC Module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest)

    ```hcl
    terraform {
      source = "tfr:///terraform-aws-modules/vpc/aws?version=5.8.1"
    }

    <<optional Terragrunt's configuration goes here>

    include "region_common" {
      path   = find_in_parent_folders("region_commons.hcl")
      expose = true
    }

    dependency "aws_az" {
      config_path = "<path to the module declared in the previous step>"

      mock_outputs_allowed_terraform_commands = ["validate", "plan", "init", "force-unlock"]
      mock_outputs = {
        azs = ["invalid-region-1", "invalid-region-2", "invalid-region-3"]
      }
    }

    inputs = {
      <inputs>

      azs              = dependency.aws_az.outputs.azs
      private_subnets  = [for k, v in dependency.aws_az.outputs.azs : cidrsubnet(include.region_common.locals.vpc_cidr, 4, k)]
      public_subnets   = [for k, v in dependency.aws_az.outputs.azs : cidrsubnet(include.region_common.locals.vpc_cidr, 4, k + 4)]
      database_subnets = [for k, v in dependency.aws_az.outputs.azs : cidrsubnet(include.region_common.locals.vpc_cidr, 4, k + 8)]

      <rest of inputs>
    }
```
