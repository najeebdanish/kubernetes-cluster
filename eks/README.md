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

```sh
$ export cf_stack_name="nd-test-eks-stack"
$ aws cloudformation deploy --template-file cluster.yml --capabilities CAPABILITY_IAM --stack-name $cf_stack_name
$ aws cloudformation describe-stacks --stack-name $cf_stack_name --query Stacks[0].Outputs
```

## Configure kubectl

Set `KUBECONFIG`. We get `$CLUSTER_NAME` below from the Cloudformation output value of `EKSClusterName`

```sh
$ export REPO_DIR=`pwd`
$ export KUBECONFIG=$REPO_DIR/k8s-config.yml
$ rm -f k8s-config.yml
$ touch k8s-config.yml
$ export CLUSTER_NAME=`aws cloudformation describe-stacks --stack-name $cf_stack_name --query Stacks[0].Outputs[1].OutputValue --output text`
$ aws eks update-kubeconfig --name $CLUSTER_NAME
```

## Join nodes to the cluster

We replace NODE_INSTANCE_ROLE in `k8s-configmap-aws.yml` and by replacing it with the Cloudformation output value of `NodeInstanceRole`

```sh
$ cp k8s-configmap-aws.yml.orig k8s-configmap-aws.yml
$ export NODE_INSTANCE_ROLE=`aws cloudformation describe-stacks --stack-name $cf_stack_name --query Stacks[0].Outputs[2].OutputValue --output text`
$ sed -i '' "s?NODE_INSTANCE_ROLE?${NODE_INSTANCE_ROLE}?g" k8s-configmap-aws.yml
$ kubectl apply -f k8s-configmap-aws.yml
```

### Define storage class

Before claiming a persistent volume, you need to define an EBS storage class and set it default.

```sh
$ ./kubectl apply -f k8s-storageclass-ebs-gp2.yml
$ ./kubectl get storageclass
NAME            PROVISIONER             AGE
gp2 (default)   kubernetes.io/aws-ebs   13s
```

[Storage Classes - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/storage-classes.html)

### Deploy a sample application


## Clean up

```sh
$ aws cloudformation delete-stack --stack-name $cf_stack_name
```