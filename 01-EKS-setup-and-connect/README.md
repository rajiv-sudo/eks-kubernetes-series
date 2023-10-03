# Summary
In this example, we will create an Amazon EKS cluster using EKS managed node group. Then we will show how to connect to the kubernestes cluster from the user's computer using kubectl. 

We will use Hashicorp Terraform example from [Intel Cloud Optimization Modules for Terraform - Amazon EKS Module](https://github.com/intel/terraform-intel-aws-eks/tree/main/Examples/EKS_Managed_Node_Group) to create the cluster on Amazon EC2 instances. This is using Terraform which is an enterprise accepted Infrastructure as Code tool that works across multi cloud platforms.

**Note - ** You can use other methods to create your Amazon EKS cluster, following AWS Documentation - [Creating an Amazon EKS cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html) and [Creating a managed node group](https://docs.aws.amazon.com/eks/latest/userguide/create-managed-node-group.html). 

# Pre-requisites
1. Hashicorp Terraform installed. Documentation for [Hashicorp Terraform Install](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
2. AWS CLI installed and configured with required access for the user. We tested this setup with an AWS user having administrator access. but you can use fine grained IAM access and permissions for the user you will use. Documentastion to [Install or update the latest version of the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). Setup your AWS CLI environment for long term credentials - [Configuring using AWS CLI commands](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html). See the Lon-term credentials tab
3. Kubernetes kubectl tool installed. Documentation for [Installing kubectl Tool](https://kubernetes.io/docs/tasks/tools/)


# Architecture
The architecture diagram is showing our EKS managed kubernetes cluster in AWS. The user connects to the EKS cluster via the EKS managed control plane. The cluster is setup with 2 worker nodes within an auto scaling group spread across different availability zones.

**Architecture - Managed Kubernetes Cluster with EKS Managed Node Group**
<img src="https://github.com/rajivmandal123/eks-kubernetes-series/blob/main/images/EKS-Setup-and-Connect.png?raw=true" alt="Link" width="600"/>
# Steps

# Considerations

# References