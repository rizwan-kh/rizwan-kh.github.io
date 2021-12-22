---
title: "Integration of Azure AD as OIDC identity provider for AWS EKS"
date: 2021-12-22T23:40:21+04:00
draft: false
toc: false
images:
tags:
  - eks
  - azure AD
  - kubernetes
---

## Introduction
In my project we are using many flavours of Kubernetes viz. EKS, AKS, GKE, RKE, ACK. RBAC for all these cluster are managed via a central Active Directory as well as the user authentication, and this is achieved centrally by onboarding all the cluster on Rancher to manage all Kubernetes cluster.

I had a requirement where we couldn't onboard the users to our Active Directory, and the plan was to give them access to Amazon EKS via an Azure AD external users(or guest users).

Since now, [Amazon supports user authentication with OIDC compatible identity provider](https://aws.amazon.com/about-aws/whats-new/2021/02/amazon-eks-clusters-support-user-authentication-oidc-compatible-identity-providers/), I tried my hands at integrating AAD guest users to access this EKS.

## Setup

### Setup Azure AD App registration
- Click on New Registration
- Provide a name viz. `eks-auth-app`
- Select `Accounts in this organizational directory only (LubanDevOps only - Single tenant)`
- Click `Register`

After the app is created, there are couple of configuration that needs to be performed.
- Click on `Authentication` and under `Advance settings` and check the `Allow public client flows` and save it
---
- check if platform needs to be added and if yes, then add a platform of type Web with redirect URI as `http://localhost/red` and select `ID tokens (used for implicit and hybrid flows)`
---
- If you want group to be part of your OIDC token, under Token configuration click Add groups claim. Select Security groups and Group ID. Groups created in AAD can only be included by their ObjectID and not name.

- Copy the `Application (client) ID` and `Directory (tenant) ID` to be used later.

### Configure Amazon EKS
Amazon provides a way to configure OIDC compatible identity provider via the management console. Navigate to Authentication under Configuration in the EKS cluster panel when you select your cluster.

- Click on `Associate Identity Provider`
    - Issuer URL: `https://sts.windows.net/[Directory (tenant) ID]`
    - Client ID: `[Application (client) ID]`
    - Username claim: `email` #this will be `upn` if you want to authenticate direct member users of Azure AD and not guest users
    - Groups claim: `groups`
    - Username prefix: `aad:`
    - Groups prefix: `aad:`
- If you want to add tags to identify the service principal or any other detail, you can add tags and save.
- This will update your OIDC identity provider in the API server and this takes sometime.(For me it took almost 40-50 minutes on each trial)

Now the server configuration is completed. We will proceed with configuring clients

### Client configuration
Since mostly, we use `kubectl` to interact with Kubernetes, we will configure kubectl to use [kubelogin](https://github.com/Azure/kubelogin) which is a [client-go credential (exec) plugin](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins) implementing azure authentication. This plugin provides features that are not available in kubectl. It is supported on kubectl v1.11+

#### Install Azure/kubelogin
I followed the installation instructions from https://github.com/Azure/kubelogin:

Install using homebrew:
```
brew install Azure/kubelogin/kubelogin
```

Install directly from Github
```
wget https://github.com/Azure/kubelogin/releases/latest/download/kubelogin-linux-amd64.zip
unzip kubelogin-linux-amd64.zip -d kubelogin
mv kubelogin/bin/linux_amd64/kubelogin /usr/local/bin/
rm -r kubelogin*
```

#### Configure kubectl
Below config contains sample garbage value, please replace the below fields with proper value
- certificate-authority-data
- server
- value for server-id
- value for client-id
- value for tenant-id

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tC ...REDACTED STRING
    server: https://34DA2A37GFSDXY7GFYWGE7ABA34Q11R0.gr7.us-east-1.eks.amazonaws.com
  name: aws-eks-aad
contexts:
- context:
    cluster: aws-eks-aad
    user: azure-user
  name: aws-eks-azure-user
current-context: aws-eks-azure-user
kind: Config
preferences: {}
users:
- name: azure-user
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - get-token
      - --environment
      - AzurePublicCloud
      - --server-id
      - a3xxxx4fe-xxxx-xxxx-xxxx-dexxxxxx210
      - --client-id
      - a3xxxx4fe-xxxx-xxxx-xxxx-dexxxxxx210
      - --tenant-id
      - 67xxx5c-xxxx-xxxx-xxxx-254xxxxx9ccf
      command: kubelogin
      env: null
```

### Authentication
Post completion of this setup, issue `kubectl` command to get the instruction to authenticate yourself; Note this will only authenticate you, you would need to configure RBAC to allow the users to interact with cluster.

```
kubectl get pods
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code EJQH9Q8LS to authenticate.
```

### Authorization
We would need to setup RBAC for different users and group as per our need, for that I created 3 Azure AD groups and mapped then as below
1. eks-admins
2. eks-editors
3. eks-viewers
```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  annotations:
    for.ref.AAD.Group: https://portal.azure.com/#blade/Microsoft_AAD_IAM/GroupDetailsMenuBlade/Overview/groupId/abe95a71-b52a-43f2-9095-fecd4f6ef58d
  name: aad-cluster-admins
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: aad:abe95a71-b52a-43f2-9095-fecd4f6ef58d  # group ID setup
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: aad:rizwan.khan@mycontoso.com  # user setup
  
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    for.ref.AAD.Group: https://portal.azure.com/#blade/Microsoft_AAD_IAM/GroupDetailsMenuBlade/Overview/groupId/0bec3baa-511d-4816-ae62-2758d6023cf1
  name: aad-cluster-editors
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: aad:0bec3baa-511d-4816-ae62-2758d6023cf1

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  annotations:
    for.ref.AAD.Group: https://portal.azure.com/#blade/Microsoft_AAD_IAM/GroupDetailsMenuBlade/Overview/groupId/03e95a71-b92a-4cf2-9895-fecd3f6ef58d
  name: aad-cluster-viewers
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: Group
  name: aad:03e95a71-b92a-4cf2-9895-fecd3f6ef58d
  apiGroup: rbac.authorization.k8s.io

```

After these RBAC are set, I added my user and couple of other is those group and voila, we were all set!
