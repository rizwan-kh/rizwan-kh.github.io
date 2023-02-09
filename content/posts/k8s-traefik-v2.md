---
title: "Traefik on Kubernetes with Let's Encrypt & Route53"
date: 2021-07-28T17:40:27+04:00
draft: false
tags:
- kubernetes
- traefik
---
![k8s-traefik-le-route53](/k8s-traefik-le-route53.svg)

## Why Traefik?
[**Traefik**](https://traefik.io/traefik/) is a modern dynamic load balancer and reverse proxy, it's easy to set up, and control and provides lots of options which sit right with our use cases. It integrates with Lets Encrypt to provide SSL termination along with support for service discovery, tracing, and metrics out of the box running on Kubernetes as a small pod.

### Since when
We have been using Traefik since early 2017 on 2 of our clusters as a daemonset Kubernetes object with external DNS to update the worker nodes IP with our DNS provider and used to manually generate and update Lets Encrypt certificate on a Kubernetes secret object.

Later, with the release of Traefik proxy v2, we started deploying Traefik as a deployment Kubernetes object and have been using it since then on more than 10 clusters with auto Lets Encrypt certificate generation and we use a small custom init-container to update the Route53 DNS entry.

## How is the entire setup done

It might feel a bit sketchy if you're doing this for the first time, but once set up - you're good to go for almost till you don't want to upgrade. :wink:

### Pre-requisite
- AWS IAM Keys with Route53 list and update access
- AWS Route53 Public Hosted Zone
- A working Kubernetes cluster

#### First thing first
Set up your Kubernetes cluster, doesn't matter if it's EKS, AKS, GKE, Custom kubeadm, RKE, K3S, KOPS, etc. Once the cluster is setup and ready; we would deploy the below Kubernetes manifest yaml (Now, Traefik support deployment via Helm), we use kustomize to deploy Traefik on each of our clusters from our GitLab CI CD pipeline (will not deep dive on that here, just will show you the manifest to start with)

#### Let's Start with Traefik Deployment
Deploy the below traefik-crd-sa-cr-crb.yaml file with `kubectl apply -f traefik-crd-sa-cr-crb.yaml` as-is.

[The below CRD's and supporting manifest are taken from this Traefik official documentation](https://doc.traefik.io/traefik/user-guides/crd-acme/#cluster-resources)
```
# traefik-crd-sa-cr-crb.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressrouteudps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteUDP
    plural: ingressrouteudps
    singular: ingressrouteudp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsstores.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSStore
    plural: tlsstores
    singular: tlsstore
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: serverstransports.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: ServersTransport
    plural: serverstransports
    singular: serverstransport
  scope: Namespaced

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
      - ingressroutes
      - traefikservices
      - ingressroutetcps
      - ingressrouteudps
      - tlsoptions
      - tlsstores
      - serverstransports
    verbs:
      - get
      - list
      - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: default
---
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: default
  name: traefik-ingress-controller

```

Now save the below manifest and deploy it on your cluster with a few amendments like replacing proper values for AWS Route53 hosted zone and IAM keys, storageClassName, acme email address, and your sub-domain in the initContainer args section

:information_source: - you can use the Instance profile/IAM Role as well if you don't want to use IAM keys 

```
# create PVC to store the Lets Encrypt cert to persist between Traefik restarts
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
  name: traefik-acme-storage-default
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: gp2
---
# replace the below values for data with the proper encoded value (these are random garbage values :stuck_out_tongue_closed_eyes: ) 
apiVersion: v1
data:
  access_key_id: S1FSMjI0NDRKNVNBSUtBRDRLTkgK  
  hosted_zone: WjM0RzY1QUVERURFCg==
  region: dXMtZWFzdC0x
  secret_key_id: SmszMmRUUUJWRlZTVVlCWVZESVVJRVZWSTZHVkhHMzJRVwo=
kind: Secret
metadata:
  labels:
  name: route53-secret
  namespace: default
type: Opaque
---
# create Traefik clusterIP service, note we won't be using the web as we avoid using port 80 for all Production purposes
apiVersion: v1
kind: Service
metadata:
  labels:
  name: traefik
  namespace: default
spec:
  ports:
  - name: web
    port: 8000
    protocol: TCP
    targetPort: 8000
  - name: admin
    port: 8080
    protocol: TCP
    targetPort: 8080
  - name: websecure
    port: 4443
    protocol: TCP
    targetPort: 4443
  selector:
    app: traefik
  type: ClusterIP
---
# Traefik deployment YAML, initContainers will be explained in detail below
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: traefik
  name: traefik
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: traefik
    spec:
      containers:
      - args:
        - --api.insecure
        - --accesslog=true
        - --entrypoints.websecure.Address=:4443
        - --providers.kubernetescrd
        - --providers.kubernetesingress
        - --entrypoints.websecure.http.tls.options=default
        - --entrypoints.websecure.http.tls.certResolver=default
        - --certificatesresolvers.default.acme.dnschallenge.provider=route53
        - --certificatesResolvers.default.acme.dnsChallenge.delayBeforeCheck=5
        - --certificatesresolvers.default.acme.email=my.email@xzy.com
        - --certificatesresolvers.default.acme.storage=/data/acme.json
        - --metrics.prometheus=true
        - --metrics.prometheus.entryPoint=metrics
        - --serverstransport.insecureskipverify
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              key: access_key_id
              name: route53-secret
        - name: AWS_HOSTED_ZONE_ID
          valueFrom:
            secretKeyRef:
              key: hosted_zone
              name: route53-secret
        - name: AWS_REGION
          valueFrom:
            secretKeyRef:
              key: region
              name: route53-secret
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              key: secret_key_id
              name: route53-secret
        image: traefik:v2.4
        imagePullPolicy: IfNotPresent
        name: traefik
        ports:
        - containerPort: 4443
          hostPort: 443
          name: websecure
          protocol: TCP
        - containerPort: 8080
          name: admin
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /data
          name: cert-storage-volume
      dnsPolicy: ClusterFirst
      initContainers:
      - args:
        - 'apt-get install awscli -y;HOSTED_ZONE_ID="Z34G65AEDEDE";INPUT="{\"ChangeBatch\": { \"Comment\":
          \"Update the A record set\", \"Changes\":[ {\"Action\": \"UPSERT\", \"ResourceRecordSet\":
          { \"Name\": \"*.k8s.mydomain.com\", \"Type\": \"A\", \"TTL\":
          300, \"ResourceRecords\": [ { \"Value\": \"HOSTIP\"}]}}]}}"; HOSTIP=`curl
          http://169.254.169.254/latest/meta-data/local-ipv4`;INPUT_JSON=`echo $INPUT
          | sed "s/HOSTIP/$HOSTIP/"`; echo $INPUT_JSON;aws route53 change-resource-record-sets
          --hosted-zone-id "$HOSTED_ZONE_ID" --cli-input-json "$INPUT_JSON"'
        command:
        - /bin/sh
        - -c
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              key: access_key_id
              name: route53-secret
        - name: AWS_REGION
          valueFrom:
            secretKeyRef:
              key: region
              name: route53-secret
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              key: secret_key_id
              name: route53-secret
        envFrom:
        - secretRef:
            name: route53-secret
        image: ubuntu:xenial    
        imagePullPolicy: IfNotPresent
        name: route53-changes
      serviceAccount: traefik-ingress-controller
      serviceAccountName: traefik-ingress-controller
      volumes:
      - name: cert-storage-volume
        persistentVolumeClaim:
          claimName: traefik-acme-storage-default
----
```

The manifest is pretty straightforward, with the only non-standard part being the initContainer, which is used here to get the IP address of the worker node on which Trafik is deployed and to update it in AWS Route53 with an A record.

Assuming the IP address is 10.7.60.70, an entry with record *.k8s.mydomain.com with an A record pointing to IP 10.7.60.70 will get inserted in Route53 in your public hosted zone. 

#### Is this even working?
Of course, if we have done all this, how do we validate, for that we will deploy a service and ingressRoute CRD with the manifest provided below

`kubectl apply -f traefik-ir.yaml`

```
# traefik-ir.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`traefik.k8s.mydomain.com`) && PathPrefix(`/`)
    kind: Rule
    services:
    - name: admin
      port: 8080
  tls:
    certResolver: default
```

You should now be able to browse to this URL https://traefik.k8s.mydomain.com with proper Lets Encrypt TLS certificates and would see the Treafik dashboard. 

![traefik-dashboard](/traefik-dashboard.webp)

### Final thoughts
This blog may not do justice in explaining how this tool exactly works, however, I intended to get you a working traefik deployment with AWS Route53, and I guess that would help someone in need.