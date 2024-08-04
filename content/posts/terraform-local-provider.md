---
title: "How to use local provider in Terraform"
date: 2024-08-04T11:50:27+04:00
draft: false
tags:
- terraform
- golang
---

# Enable local Terraform provider for Testing/Development

## Introduction

Recently, we faced an issue with databricks/databricks provider and we spend alomost 2 weeks figuring out an auth issue that we saw in the provider for storage_account for th Unity Catalog. And thus, we wanted to dive into the databricks provider golang code to see if we can add few debug statements and understand where it was failing. So we cloned the repo from GitHub, made the changes and used the `contributing.md` to learn how to build the package - it was as easy as running the command `make install` and the Makefile had all the magic command configured.

We got a binary locally available to use. Now we faced the challenge of how to use the local binary, as with each `terraform init` command, terraform was pulling the module from registry.terraform.io.

## Setup

Looking around the documents, we found a way and all it needed was to create a `~/.terraformrc` file with the below content

```t
provider_installation {
  dev_overrides {
    "registry.terraform.io/databricks/databricks" = "/Users/rizwan/GitHub/databricks-terraform/"
  }

  # For all other providers, install them directly from their origin provider
  # registries as normal. If you omit this, Terraform will _only_ use
  # the dev_overrides block, and so no other providers will be available.
  direct {}
}
```

The after "=" sign contains the value of the directory where the binary exist, ensure the binary has the version number. 

When you perform a new `terraform init` or `terraform apply`, you will be present with a warning, that it's using the local databricks provider and not the one available on the remote registry.terraform.io.