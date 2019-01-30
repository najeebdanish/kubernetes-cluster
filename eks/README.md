# Description

This cloudformation template creates 2 public and 2 private subnets with NAT gateways in each of the public subnets. All the worker nodes are launched into private subnets.

## Pre-requisite

Before deploying the EKS stack please install
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [aws-cli](https://docs.aws.amazon.com/cli/latest/userguide/install-linux-al2017.html)
* [aws-iam-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)

## Deploy stack

Clone this repository and then go into that directory. Also, set proper AWS credentials to run the stack (make sure the credentials you are using have all the required permissions)

The command below will bring the stack up with default parameters. Please check the template to see if you need to change/pass any parameters.

Replace <MY_IP> with your local IP below

```sh
export cf_stack_name="nd-test-eks-stack"
aws cloudformation deploy --template-file cluster.yml --capabilities CAPABILITY_IAM --stack-name $cf_stack_name --parameter-overrides LocalIP=<MY_IP>
aws cloudformation describe-stacks --stack-name $cf_stack_name --query Stacks[0].Outputs
```

## Configure kubectl

Set `KUBECONFIG`. We get `$CLUSTER_NAME` below from the Cloudformation output value of `EKSClusterName`

```sh
export REPO_DIR=`pwd`
export KUBECONFIG=$REPO_DIR/k8s-config.yml
rm -f k8s-config.yml
touch k8s-config.yml
export CLUSTER_NAME=`aws cloudformation describe-stacks --stack-name $cf_stack_name --query Stacks[0].Outputs[1].OutputValue --output text`
aws eks update-kubeconfig --name $CLUSTER_NAME
```

## Join nodes to the cluster

We replace NODE_INSTANCE_ROLE in `k8s-configmap-aws.yml` and by replacing it with the Cloudformation output value of `NodeInstanceRole`

```sh
cp k8s-configmap-aws.yml.orig k8s-configmap-aws.yml
export NODE_INSTANCE_ROLE=`aws cloudformation describe-stacks --stack-name $cf_stack_name --query Stacks[0].Outputs[2].OutputValue --output text`
sed -i '' "s?NODE_INSTANCE_ROLE?${NODE_INSTANCE_ROLE}?g" k8s-configmap-aws.yml
kubectl apply -f k8s-configmap-aws.yml
```

## Define storage class

Before claiming a persistent volume, you need to define an EBS storage class and set it default.

```sh
kubectl apply -f k8s-storageclass-ebs-gp2.yml
kubectl get storageclass
NAME            PROVISIONER             AGE
gp2 (default)   kubernetes.io/aws-ebs   13s
```

[Storage Classes - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/storage-classes.html)

## Simple NGINX Ingress Controller

Setup an NGINX ingress controller

```sh
kubectl apply -f ingress-controller/nginx-ingress-controller/ingress-controller/common/namespace.yml
kubectl apply -f ingress-controller/nginx-ingress-controller/ingress-controller/common/service-account.yml
kubectl apply -f ingress-controller/nginx-ingress-controller/ingress-controller/common/default-secret.yml
kubectl apply -f ingress-controller/nginx-ingress-controller/ingress-controller/common/nginx-config.yml
kubectl apply -f ingress-controller/nginx-ingress-controller/ingress-controller/rbac/rbac.yml
kubectl apply -f ingress-controller/nginx-ingress-controller/ingress-controller/deployment/nginx-ingress.yml
kubectl apply -f ingress-controller/nginx-ingress-controller/ingress-controller/service/aws-elb.yml
```

## Deploy a sample application

We will be deploying four applications
* app1 and app2 in first sample
* app3 and app4 in second sample

### First set of applications

Deploy the first sample set of applications

```sh
kubectl create -f ingress-controller/nginx-ingress-controller/application-sample-1/application.yml
kubectl create -f ingress-controller/nginx-ingress-controller/application-sample-1/secret.yml
kubectl create -f ingress-controller/nginx-ingress-controller/application-sample-1/ingress.yml
```

Setup ELB public IP and HTTPS port to resolve the application domain

```sh
$ nslookup <ELB_Public_DNS_NAME>
$ IC_IP=<PUBLIC_IP_OF_LB>
$ IC_HTTPS_PORT=443
```

Test it

```sh
curl --resolve sample1.example.com:$IC_HTTPS_PORT:$IC_IP https://sample1.example.com:$IC_HTTPS_PORT/app1 --insecure
curl --resolve sample1.example.com:$IC_HTTPS_PORT:$IC_IP https://sample1.example.com:$IC_HTTPS_PORT/app2 --insecure
```

This works fine. Now test another app that does not exist in this set. It must give us a Not Found error.

```sh
curl --resolve sample1.example.com:$IC_HTTPS_PORT:$IC_IP https://sample1.example.com:$IC_HTTPS_PORT/app3 --insecure
```

### Second set of applications

Deploy the second sample set of applications

```sh
kubectl create -f ingress-controller/nginx-ingress-controller/application-sample-2/application.yml
kubectl create -f ingress-controller/nginx-ingress-controller/application-sample-2/ingress.yml
```

Test it

```sh
curl --resolve sample2.example.com:$IC_HTTPS_PORT:$IC_IP https://sample2.example.com:$IC_HTTPS_PORT/app3 --insecure
curl --resolve sample2.example.com:$IC_HTTPS_PORT:$IC_IP https://sample2.example.com:$IC_HTTPS_PORT/app4 --insecure
```

This works fine. Now test another app that does not exist in this set. It must give us a Not Found error.

```sh
curl --resolve sample1.example.com:$IC_HTTPS_PORT:$IC_IP https://sample2.example.com:$IC_HTTPS_PORT/app1 --insecure
```

## Clean up

Cleanup the Kubernetes cluster first

```sh
kubectl delete -f ingress-controller/nginx-ingress-controller/application-sample-2/application.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/application-sample-2/ingress.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/application-sample-1/application.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/application-sample-1/secret.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/application-sample-1/ingress.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/ingress-controller/default-backend.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/ingress-controller/service/aws-elb.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/ingress-controller/deployment/nginx-ingress.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/ingress-controller/rbac/rbac.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/ingress-controller/common/nginx-config.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/ingress-controller/common/default-secret.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/ingress-controller/common/service-account.yml
kubectl delete -f ingress-controller/nginx-ingress-controller/ingress-controller/common/namespace.yml
```

Now delete the cloudformation stack

```sh
aws cloudformation delete-stack --stack-name $cf_stack_name
```