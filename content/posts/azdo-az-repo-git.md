---
title: "(Azure DevOps) How to commit/push to Azure Git Repository from Azure Pipeline"
date: 2021-03-14T00:28:21+04:00
draft: false
toc: false
images:
tags:
  - azure devops
  - pipelines
  - git repo
---
![azure](/azure-pipelines.png)
## Azure Git Repository
In one of our use-case, we had to create some terraform HCL files via Azure DevOps pipeline operations and then commit/push those newly generated files back to Azure Git Repos. This is generally not a very usual use case or operation. However, we were able to achieve it using the below pipeline.

Assuming, your pipeline has completed and has generated or modified the files, now it's time to push the changes to git - this surely will trigger another pipeline, however, this second pipeline will start the deployment and not the generation of terraform files as it did in the first pipeline trigger.

```
# get the branch name
GIT_BRANCH=$(git branch -q)

# set your git global variables
git config --global user.email "rizwan.khan@x-ops.com"
git config --global user.name "Rizwan Khan"

git add *.tf
if [ $? -ne 0 ]; then 
    echo "Nothing to git add, maybe path error?"
    exit 1
fi

git status

# perform git commit with generic message
git commit -m "[new generated env tf files via build] - commit via pipeline"
if [ $? -ne 0 ]; then 
    echo "Nothing to git commit, maybe path error?"
    exit 1
fi

git config --get remote.origin.url
# check if you're on the right branch
git branch

# push to the branch where you're working
git push --set-upstream origin HEAD:$GIT_BRANCH
if [ $? -ne 0 ]; then 
    echo "Not able to push the changes, maybe another change was made in the repo?"
    exit 1 
fi

git status
```

The commands are very self-explanatory. However, the important item I skipped mentioning is the authentication part, which happens using SSH keys, stored under Azure Pipelines -> Library -> Variable Group. You would also need to put your public SSH keys under your profile. Here is the [link from Microsoft which explains how the authentication setup is done](https://learn.microsoft.com/en-us/azure/devops/repos/git/use-ssh-keys-to-authenticate?view=azure-devops#set-up-ssh-key-authentication)

Hope you enjoy the reading!