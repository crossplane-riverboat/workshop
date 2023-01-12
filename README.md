# Crossplane workshop

## Requirements

- Personal laptop with docker installed according to the specific OS instructions
  - run `docker run --rm hello-world` from command line to confirm successful installation

- An AWS account

### AWS IAM User creation

In the AWS console create a new IAM user and grant it administrator access (you would normally grant only specific privileges but it's out of scope of this exercise)

<br />

# Getting started

## KIND (kubernetes in docker)

### Install KIND following your OS instructions from https://kind.sigs.k8s.io/docs/user/quick-start

```
kind create cluster --config kind-cluster.yaml
```
```
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
#
```

### Install kubectl 

https://kubernetes.io/docs/tasks/tools/

### Confirm you can connect and view cluster status

```
kubectl get no
```
```
NAME                 STATUS     ROLES           AGE   VERSION
kind-control-plane   Ready      control-plane   46s   v1.25.3
kind-worker          NotReady   <none>          15s   v1.25.3
kind-worker2         NotReady   <none>          15s   v1.25.3
```

Try again until the worker nodes are in a "Ready" state
```
kubectl get no
```
```
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   95s   v1.25.3
kind-worker          Ready    <none>          64s   v1.25.3
kind-worker2         Ready    <none>          64s   v1.25.3
```


### Install Helm

https://helm.sh/docs/intro/quickstart/

### Install Crossplane

```
kubectl create namespace crossplane-system
```
```
helm repo add crossplane-stable https://charts.crossplane.io/stable
```
```
helm repo update
```
```
helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

### (optional but recommended) set current context to crossplane-system

```
kubectl config set-context --current --namespace=crossplane-system
```

### Confirm crossplane is running

```
kubectl get po -n crossplane-system
```
```
NAME                                       READY   STATUS     RESTARTS   AGE
crossplane-6fd66789b9-ht79w                0/1     Init:0/1   0          9s
crossplane-rbac-manager-6f4889484b-vkw6c   0/1     Init:0/1   0          9s
```
```
kubectl get po -n crossplane-system
```
```
NAME                                       READY   STATUS    RESTARTS   AGE
crossplane-6fd66789b9-ht79w                1/1     Running   0          97s
crossplane-rbac-manager-6f4889484b-vkw6c   1/1     Running   0          97s
#
```

### Create the credentials file and save it as creds.conf

```
[default]
aws_access_key_id =
aws_secret_access_key =
```

### Create the kube secret

```
kubectl create secret generic aws-creds -n crossplane-system --from-file=creds=./creds.conf
```
> secret/aws-creds created

Note - if you already have the kube secret and you want to update it, you can either delete it and recreate or you can patch it with:
```
kubectl create secret generic aws-creds -n crossplane-system --from-file=creds=./creds.conf --save-config --dry-run=client -o yaml|kubectl apply -f -
```
> secret/aws-creds configured

### (Optional) Delete the creds.conf file

### Confirm the CRDs are available

```
kubectl get provider
```
> No resources found

"No resources found" and an exit code of zero indicates the CRDs are available. If the message is `error: the server doesn't have a resource type "provider"` you need to wait until the provider is available.

### Install the AWS Provider and enable debug mode

```
kubectl apply -f providers.yaml
```

### Confirm the provider is up

```
# kubectl get po
NAME                                         READY   STATUS    RESTARTS   AGE
crossplane-6fd66789b9-ht79w                  1/1     Running   0          27m
crossplane-rbac-manager-6f4889484b-vkw6c     1/1     Running   0          27m
provider-aws-245ce7fb587d-75b5b77dbd-l64fh   1/1     Running   0          2m8s
# kubectl logs provider-aws-245ce7fb587d-75b5b77dbd-l64fh 2>&1 | tail
1.667577908305575e+09   INFO    controller.managed/method.apigateway.aws.crossplane.io  Starting workers        {"reconciler group": "apigateway.aws.crossplane.io", "reconciler kind": "Method", "worker count": 10}
1.6675779083063836e+09  INFO    controller.managed/responseheaderspolicy.cloudfront.aws.crossplane.io   Starting workers        {"reconciler group": "cloudfront.aws.crossplane.io", "reconciler kind": "ResponseHeadersPolicy", "worker count": 10}
#
```

### Configure the AWS Provider

```
kubectl apply -f providerconfig.yaml
```
> providerconfig.aws.crossplane.io/default created

### Provision something

```
kubectl apply -f examples/vpc.yaml
```
> vpc.ec2.aws.crossplane.io/production-vpc created
```
kubectl get vpc
NAME             READY   SYNCED   ID                      CIDR          IPV6CIDR   AGE
production-vpc   True    True     vpc-08bca91979ce811e8   10.0.0.0/24              4s
#
```

If Synced is true then the new subnet should be visible in the AWS Console.

```
# kubectl logs provider-aws-245ce7fb587d-75b5b77dbd-l64fh 2>&1 | grep vpc|tail -3
1.6675786363062613e+09  DEBUG   provider-aws    External resource is up to date {"controller": "managed/vpc.ec2.aws.crossplane.io", "request": "/production-vpc", "uid": "54eb9996-90c7-4870-8d53-76354662c1ab", "version": "6745", "external-name": "vpc-08bca91979ce811e8", "requeue-after": 1667578696.306256}
1.6675786963316035e+09  DEBUG   provider-aws    Reconciling     {"controller": "managed/vpc.ec2.aws.crossplane.io", "request": "/production-vpc"}
1.667578696631119e+09   DEBUG   provider-aws    External resource is up to date {"controller": "managed/vpc.ec2.aws.crossplane.io", "request": "/production-vpc", "uid": "54eb9996-90c7-4870-8d53-76354662c1ab", "version": "6745", "external-name": "vpc-08bca91979ce811e8", "requeue-after": 1667578756.631114}
#
```

### Delete the VPC from the AWS console

Wait a few minutes and the VPC should show back. The provider logs should show:

```
1.6675787566551545e+09  DEBUG   provider-aws    Reconciling     {"controller": "managed/vpc.ec2.aws.crossplane.io", "request": "/production-vpc"}
1.667578757201898e+09   DEBUG   provider-aws    Successfully requested creation of external resource    {"controller": "managed/vpc.ec2.aws.crossplane.io", "request": "/production-vpc", "uid": "54eb9996-90c7-4870-8d53-76354662c1ab", "version": "6745", "external-name"
: "vpc-08bca91979ce811e8", "external-name": "vpc-0cd0dde5498154d51"}
1.6675787572036462e+09  DEBUG   events  Normal  {"object": {"kind":"VPC","name":"production-vpc","uid":"54eb9996-90c7-4870-8d53-76354662c1ab","apiVersion":"ec2.aws.crossplane.io/v1beta1","resourceVersion":"7172"}, "reason": "CreatedExternalResource", "message": "Succ
essfully requested creation of external resource"}
```

### List your managed resources

```
kubectl get managed
```
```
NAME                                       READY   SYNCED   ID                      CIDR          IPV6CIDR   AGE
vpc.ec2.aws.crossplane.io/production-vpc   True    True     vpc-0ea808fe13c599c03   10.0.0.0/24              21s
#
```

### Provision something else


```
kubectl apply -f examples/instance.yaml
```

```
kubectl apply -f examples/efs.yaml
```

Upstream provider documentation:

https://doc.crds.dev/github.com/crossplane/provider-aws@v0.32.0

### Create a composition

```
kubectl apply -f compositions/1-xrd.yaml 
```
> compositeresourcedefinition.apiextensions.crossplane.io/xcomputeinstances.example.com created
```
kubectl apply -f compositions/2-composition.yaml 
```
> composition.apiextensions.crossplane.io/computeinstance created
```
kubectl apply -f compositions/3-xr.yaml 
```
> computeinstance.example.com/test1 created

### Check status

```
# kubectl get computeinstance.example.com/test1
NAME    SYNCED   READY   CONNECTION-SECRET   AGE
test1   True     False                       42s
# kubectl get instance.ec2.aws.crossplane.io
NAME                READY   SYNCED   ID                    STATE     AGE
test1-cfjhd-x42vj   True    True     i-0bf72a9e83cdcae92   running   71s
# kubectl get filesystem.efs.aws.crossplane.io
NAME                READY   SYNCED   EXTERNAL-NAME
test1-cfjhd-k242v   True    True     fs-0b13b780db4ac71d2
# 
# kubectl get computeinstance.example.com/test1
NAME    SYNCED   READY   CONNECTION-SECRET   AGE
test1   True     True                        119s
# 
```

### Clean up

```
kind delete cluster
```
> Deleting cluster "kind" ...
