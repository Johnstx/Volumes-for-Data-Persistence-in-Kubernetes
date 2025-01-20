### Data Persistence in Kubernetes
 This lab provides a guide to the practice of achieving data persistence in pods.

Pod/s running in an EKS will be configured for data persistence.

#### Requirements

1. **An AWS account** - For provisioning an EKS cluster.
2. **Terraform** - To automate cluster provisioning. (optional in a testing case)

You can create  a test cluster quickly by using the block below - 

```
eksctl create cluster \
  --name staxx \
  --region us-east-2 \
  --nodegroup-name worker \
  --node-type t2.micro \
  --nodes 2
```

This will provision a cluster with the name ```staxx``` in the specified region and other specified settings as per above. It readies every other component required for a running kubernetes cluster.

```eksctl```must be installed in  your local device to enable these commands. For other pre-requisites, check [here](https://github.com/Johnstx/Project_22-Deploying-Applications-Into-Kubernetes-Cluster/blob/main/project22.md)