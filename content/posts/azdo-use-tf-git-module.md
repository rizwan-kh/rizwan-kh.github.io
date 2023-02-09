---
title: "(Azure DevOps) Fetching Terraform (Git) Modules from Azure Git Repository via Azure Pipeline"
date: 2021-03-22T00:28:21+04:00
draft: true
toc: false
images:
tags:
  - azure devops
  - pipelines
  - git repo
  - terraform
---
![azure](/azure.png)
### Introduction
This small post gives us information on how to use git modules in Terraform. This is required as Azure Repos are private and we cannot call the private git modules with authentication.


### Setup

We need to set up SSH Keys using `InstallSSHKey` task, for this, we need to create a private key, public key and known host value of ssh.dev.azure.com host and add them as secure files and variables.

Keys can be created on Mac, Linux or Windows using OpenSSH.

`ssh-keygen -C "user-email-address"`

This will generate the below two files
- id_rsa - your private key
- id_rsa.pub - your public key. 

We need to add the `id_rsa.pub` file to SSH Public Keys under User Settings and upload `id_rsa` file under Pipelines -> Library -> Secure files

For the Known Host file, we need to run the below command and get the output and save it under known_host variable. `ssh-keyscan ssh.dev.azure.com`

Now, under Pipeline, we need to use the task below to use the SSH keys to fetch or perform git operations.

```
#azure-pipeline.yml
- task: InstallSSHKey@0
  inputs:
    knownHostsEntry: $(known_host)
    sshPublicKey: $(ssh_public_key)
    sshPassphrase: $(ssh_passphrase)
    sshKeySecureFile: id_rsa
```

Now to use the above set key, we need to call the git modules like below
```
#sample.tf
module "some_azure_repo_tf_module" {
  source = "git@ssh.dev.azure.com:v3/x-ops/devops/terraform-poc?ref=main"
}
```

Please note, you need to be part of **Contributor** role to have access to all repo that you intend to fetch/call/push.

For the first pipeline execution, we would need to allow the pipeline to read the id_rsa public key secure file