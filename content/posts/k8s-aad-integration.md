---
title: "Integration of Azure AD as OIDC identity provider for Kubernetes"
date: 2022-03-14T18:23:08+04:00
draft: false
toc: false
images:
tags:
  - kubernetes
  - azure AD
  - oidc
---

## Introduction
In my project, we are using many flavours of Kubernetes viz. EKS, AKS, GKE, RKE, ACK. RBAC for all these clusters is managed via a central Active Directory as well as the user authentication, and this is achieved centrally by onboarding all the clusters on Rancher to manage all Kubernetes clusters.

I had a requirement where we couldn't onboard the users to our Active Directory, and the plan was to give them access to the Kubernetes cluster via Azure AD external users(or guest users).

OIDC-based authentication is natively supported by Kubernetes and we will be taking advantage of this to set up authentication and authorization using Azure AD.

## Setup

### Setup Azure AD App registration
- Click on New Registration
- Provide a name viz. `k8s-auth-app`
- Select `Accounts in this organizational directory only (MyAccount only - Single tenant)`
- Click `Register`

After the app is created, there is a couple of configuration that needs to be performed.
- Click on `Authentication` and under `Advance settings` and check the `Allow public client flows` and save it
---
- check if platform needs to be added and if yes, then add a platform of type Web with redirect URI as `http://localhost/red` and select `ID tokens (used for implicit and hybrid flows)`
---
- If you want group to be part of your OIDC, under Token configuration click Add groups claim. Select Security groups and Group ID. Groups created in AAD can only be included by their ObjectID and not name.

- Copy the `Application (client) ID` and `Directory (tenant) ID` to be used later.

### Configure Kubernetes API Server
Kubernetes provides a way to configure OIDC-compatible identity providers via flags passed to the kube-apiserver component. We need to the below flags while starting the kube-apiserver

```
--oidc-client-id="spn:<application id>" \
--oidc-issuer-url="https://sts.windows.net/<azure AD tenant>/"
--oidc-username-claim="email" # this will be `upn` if you want to authenticate direct member users of Azure AD and not guest users
```

If you have created your cluster using [KOPS](https://kops.sigs.k8s.io/), you can add the below in the cluster configuration and perform an update and rolling update to re-create the master nodes to enable the authentication

```
spec:
    kubeAPIServer:
        oidcClientID: spn:<application ID>
        oidcIssuerURL: https://sts.windows.net/<azure AD tenant>/
        oidcUsernameClaim: upn
        oidcUsernamePrefix: 'aad:'
```

After the master nodes are up and running, the server configuration is completed. We will proceed with configuring clients

### Client configuration
Since mostly, we use `kubectl` to interact with Kubernetes, we will configure kubectl to use - [kubelogin](https://github.com/Azure/kubelogin) which is a [client-go credential (exec) plugin](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins) implementing azure authentication. This plugin provides features that are not available in kubectl. It is supported on kubectl v1.11+
- azure cli

We will explain below how to configure both

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
#### Install Azure cli
Install using homebrew
```
brew update && brew install azure-cli
```

Although not preferred by many, using the script we can install per below
```
curl -L https://aka.ms/InstallAzureCli | bash
```

#### Configure kubectl
The below kubeconfig contains sample garbage value, please replace the below fields with the proper value
- certificate-authority-data
- server
- value for server-id
- value for client-id
- value for tenant-id

```
# using kubelogin
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tC ...REDACTED STRING
    server: https://34DA2A37GFSDXY7GFYWGE7ABA34Q11R0.myk8s.com
  name: k8s-aad
contexts:
- context:
    cluster: k8s-aad
    user: azure-user
  name: k8s-azure-user
current-context: k8s-azure-user
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

```
# using azure cli
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tC ...REDACTED STRING
    server: https://34DA2A37GFSDXY7GFYWGE7ABA34Q11R0.myk8s.com
  name: k8s-aad
contexts:
- context:
    cluster: k8s-aad
    user: azure-user
  name: k8s-azure-user
current-context: k8s-azure-user
kind: Config
preferences: {}
users:
- name: azure-user
  user:
    auth-provider:
      config:
        apiserver-id: a3xxxx4fe-xxxx-xxxx-xxxx-dexxxxxx210
        client-id: a3xxxx4fe-xxxx-xxxx-xxxx-dexxxxxx210
        environment: AzurePublicCloud
        tenant-id: 67xxx5c-xxxx-xxxx-xxxx-254xxxxx9ccf
      name: azure
```

### Authentication
Post completion of this setup, issue `kubectl` command to get the instruction to authenticate yourself; Note this will only authenticate you, you would need to configure RBAC to allow the users to interact with the cluster.

```
kubectl get pods
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code EJQH9Q8LS to authenticate.
```

### Authorization
We would need to setup RBAC for different users and groups as per our need, for that I created 3 Azure AD groups and mapped them as below
1. k8s-admins
2. k8s-editors
3. k8s-viewers
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

After these RBAC are set, I added my user and a couple of others in those groups and voila, we were all set and good to go. We can also set up RoleBindings per namespace and access level as per requirement.
