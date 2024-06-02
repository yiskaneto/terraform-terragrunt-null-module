# terraform-flux-github

Terraform module to bootstrap Flux with GitHub.

## How to use this module with Terragrunt

```hcl
terraform {
  source = "tfr:///yivolve/github/flux?version=<tag version>"
}

<optional terragrunt's configuration goes here>

inputs = {
  path = <path>
  ...rest of the inputs go here
}

```

### Provider Configuration

1. Ideally you would have the following configuration below on a file that can be passed down to multiple terragrunt child files:

  ```hcl
  generate "providers" {
    path      = "providers.tf"
    if_exists = "overwrite_terragrunt"
    contents  = <<EOF
  terraform {
    required_version = "<version>"
    required_providers {
      github = {
        source  = "integrations/github"
        version = "<version>"
      }
      github = {
        source  = "fluxcd/flux"
        version = "<version>"
      }
    }
  }

variable "GH_TOK" { // This is a PAT with write permissions to modify the contents of a repo.
  sensitive = true
  type      = string
}

provider "github" {
  owner = "<github owner>"
  token = var.github_token
}
EOF
}
```

1. Then the terragrunt project that calls this module would have this:

  > [!NOTE]
  > The inline conditionals statements are needed so "terragrunt plan" does not fail.

  ```hcl
  dependency "eks" {
    config_path = "../<path to eks dependency>/"

    mock_outputs_allowed_terraform_commands = ["validate", "plan", "init", "force-unlock"]
    mock_outputs = {
      cluster_name = "an-utterly-fake-k8s-cluster-name"
    }
  }

  dependency "private_key" {
    config_path = "../<path to private key dependency>/"

    mock_outputs_allowed_terraform_commands = ["validate", "plan", "init", "force-unlock"]
    mock_outputs = {
      secret_name = "a-super-fake-private-pem"
    }
  }

  generate "k8s-required-providers-configuration" {
    path      = "providers-configuration.tf"
    if_exists = "overwrite_terragrunt"
    contents  = <<EOF
  provider "kubernetes" {
      host                   = var.eks_cluster_name == "an-utterly-fake-k8s-cluster-name" ? "fake" : data.aws_eks_cluster.eks[0].endpoint
      cluster_ca_certificate = var.eks_cluster_name == "an-utterly-fake-k8s-cluster-name" ? "fake" : base64decode(data.aws_eks_cluster.eks[0].certificate_authority[0].data)
      token                  = var.eks_cluster_name == "an-utterly-fake-k8s-cluster-name" ? "fake" : data.aws_eks_cluster_auth.eks[0].token
  }

  provider "flux" {
    kubernetes = {
      host                   = var.eks_cluster_name == "an-utterly-fake-k8s-cluster-name" ? "fake" : data.aws_eks_cluster.eks[0].endpoint
      cluster_ca_certificate = var.eks_cluster_name == "an-utterly-fake-k8s-cluster-name" ? "fake" : base64decode(data.aws_eks_cluster.eks[0].certificate_authority[0].data)
      token                  = var.eks_cluster_name == "an-utterly-fake-k8s-cluster-name" ? "fake" : data.aws_eks_cluster_auth.eks[0].token
    }
    git = {
      url = "ssh://git@github.com/<github org>/<repo name>.git"
      ssh = {
        username    = "git"
        private_key = var.pem_secret_name == "a-super-fake-private-pem" ? "fake_string" : jsondecode(data.aws_secretsmanager_secret_version.pem_string[0].secret_string)
      }
    }
  }
  EOF
  }
  ```
