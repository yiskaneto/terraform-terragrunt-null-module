# terraform-terragrunt-null-module

The goal of this module is to allow [Terragrunt](https://terragrunt.gruntwork.io/) to place generated Terraform files from the Terragrunt's [generate](https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/#generate) configuration block into the [Terragrunt cache directory](https://terragrunt.gruntwork.io/docs/features/caching/#clearing-the-terragrunt-cache) instead of the same directory where the [terragrunt.cl](https://terragrunt.gruntwork.io/docs/getting-started/configuration/#terragrunt-configuration-file) file lives as it happens when we use Terragrunt only to generate Terraform resources or data sources on the fly instead of using a full-fleged Terraform module.

## How to use this module with Terragrunt

```hcl
terraform {
  source = "tfr:///yivolve/null-modue/terragrunt?version=<tag version>"
}

<optional terragrunt's configuration goes here>

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
