---
title: "(Azure DevOps) Committing and Pushing to Azure Git Repository from Azure Pipeline"
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
## Azure Git Repository Workflow
In a unique use-case scenario, we encountered the need to dynamically generate Terraform HCL files during Azure DevOps pipeline operations and subsequently commit and push these files back to Azure Git Repos. While this isn't a conventional operation, we successfully achieved this using the pipeline configuration outlined below.

Assuming your pipeline has completed its tasks, including the generation or modification of files, the next step is to commit and push these changes to the Git repository. This action will inevitably trigger another pipeline, with the second pipeline focused on deployment rather than the initial Terraform file generation.

```shell
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

While the commands are self-explanatory, it's crucial to highlight the authentication aspect, which is facilitated through SSH keys stored in Azure Pipelines' Variable Group. Additionally, you need to place your public SSH keys in your profile. For detailed authentication setup, refer to [Microsoft's guide on SSH key authentication](https://learn.microsoft.com/en-us/azure/devops/repos/git/use-ssh-keys-to-authenticate?view=azure-devops#set-up-ssh-key-authentication).