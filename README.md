# Kubernetes Cluster Setup

This is a guide for setting up a Kubernetes cluster on [AWS EKS](https://aws.amazon.com/eks/) with 3 worker nodes on 3 different zones, autoscaling and `Cloudwatch` enabled. The cluster setup requires the use of the `eksctl` tool found [here](https://eksctl.io/introduction/installation/). Once the tool is downloaded you can create a new cluster using the `create cluster` command and by providing as values the cluster.yaml file found in this repository. 

## Prerequisites:

To summarise these are the prerequisites for creating a new cluster:

- AWS Account and AWS CLI installed
- The eksctl tool installed
- A service role ARN which you will need to replace in the `cluster.yaml`  
- Select an EC2 instance type and the number of instances to use

Once the service role ARN is added you can create the cluster by executing this:
```bash
eksctl create cluster -f cluster.yaml
```

It takes **15-20** minutes to create the new cluster using `eksctl`, which essentially deploys a new VPC and subnets, new nodegroups and service accounts for access to relevant AWS services, and of course the EKS cluster.

Once the cluster is ready and running you will be able to see all the nodes joining and at `<ready>` mode after a couple of minutes. The `eksctl` tool sets up automatically access to the cluster by modifying the `./kube/config` file so it is not required to do anything further for access.

To access the cluster nodes via `SSH` you will need to set up a keypair on AWS and store it securely. 

## Set up Storage Classes

Please run the command below to add the new Storage class:

```
kubectl apply -f gp2-encrypted.yaml
```

## Set up Helm

The Helm package manager is used for deployments on Kubernetes. To setup Helm you need to first create a service account by applying the following:

```bash
kubectl create clusterrolebinding add-on-cluster-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:default
```

With Helm version 3 you do not need to have the Tiller server running on Kubernetes so the only thing left is to check things are running OK by executing:
```bash
helm ls
```

## Set up the Kubernetes Dashboard

To install the Kubernetes dashboard, you can simply follow the official guide provided by AWS [here](https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html). Otherwise simply run the commands below, that install the metrics server and the dashboard.

First execute:
```bash
kubectl apply -f metrics-components.yaml
```
And then the command below to install the dashboard:
```bash
kubectl apply -f kubernetes-dashboard.yaml
```

You will also need a new service account by executing this:
```bash
kubectl apply -f eks-admin-service-account.yaml
```

## Set up NGINX Controller

To expose any Restful API running inside Kubernetes you will need to set up an `NGINX Ingress Controller`. The official helm chart resides in the repo `https://kubernetes-charts.storage.googleapis.com/` so you will need to add that first.

```bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
```
The ingress values found in this repository have been customised.
```bash
helm install nginx-ingress stable/nginx-ingress -f nginx-ingress-values.yaml -n kube-system 
```

## Cluster Teardown

If you wish to remove this set up and destroy the Kubernetes cluster, simply run:

```bash
eksctl delete cluster -f cluster.yaml
```

It takes a few minutes to complete since the `eksctl` tool goes through each Cloudformation stack. Once the command finishes successfully, ensure that all resources are deleted by checking Cloudformation and the EC2 Dashboard. Often the VPC configuration can stay behind and is necessary to force delete it. 