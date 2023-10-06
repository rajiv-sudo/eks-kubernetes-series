# Summary
In this example, we will create an Amazon EKS cluster using EKS managed node group. Then we will show how to connect to the kubernestes cluster from the user's computer using kubectl.

We will use Hashicorp Terraform example from [Intel Cloud Optimization Modules for Terraform - Amazon EKS Module](https://github.com/intel/terraform-intel-aws-eks/tree/main/Examples/EKS_Managed_Node_Group) to create the cluster on Amazon EC2 instances. This is using Terraform which is an enterprise accepted Infrastructure as Code tool that works across multi cloud platforms.

**Note -** You can use other methods to create your Amazon EKS cluster, following AWS Documentation - [Creating an Amazon EKS cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html) and [Creating a managed node group](https://docs.aws.amazon.com/eks/latest/userguide/create-managed-node-group.html). 

# Pre-requisites
1. Hashicorp Terraform installed. Documentation for [Hashicorp Terraform Install](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
2. AWS CLI installed and configured with required access for the user. We tested this setup with an AWS user having administrator access. but you can use fine grained IAM access and permissions for the user you will use. Documentastion to [Install or update the latest version of the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). Setup your AWS CLI environment for long term credentials - [Configuring using AWS CLI commands](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html). See the Lon-term credentials tab
3. Kubernetes kubectl tool installed. Documentation for [Installing kubectl Tool](https://kubernetes.io/docs/tasks/tools/)
4. Git installed. Documentation for [Git Install](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)


# Architecture
The architecture diagram is showing our EKS managed kubernetes cluster in AWS. The user connects to the EKS cluster via the EKS managed control plane. The cluster is setup with 2 worker nodes within an auto scaling group spread across different availability zones.

**Architecture - Managed Kubernetes Cluster with EKS Managed Node Group**
<img src="https://github.com/rajiv-sudo/eks-kubernetes-series/blob/main/images/EKS-Setup-and-Connect.png?raw=true" alt="Link" width="600"/>

# Steps
Complete the pre-requisite steps mentioned in the Pre-requisites section. Once the pre-requisites steps are completed, follow the steps below.

### Step01:
Open your command prompt and run the below command.
```hcl
git clone https://github.com/intel/terraform-intel-aws-eks.git
```
```hcl
cd terraform-intel-aws-vm\examples\gen-ai-fastchat
```
###Step02:
Inside this folder, there is a README.md markdown file with the instructions to run the terraform code and what parameters need to be changed for your own AWS environment.

Review the README.md file, make necessary changes in the main.tf file in this folder and run the following terraform commands to create the EKS cluster.

```hcl
terraform init
terraform plan
terraform apply --auto-approve
```

### Step03:
Once the terraform commands from prior steps are completed successfully with no errors, you will have an EKS cluster created. Either from your AWS console or from the terraform output messages, copy the name of the cluster created.

The EKS cluster name created from this terraform code is called ```ex-EKS-Managed-Node-Group```

### Step04:
You we will need to configure the kubernetes configuration file with the cluster details. Use the following command.
```hcl
aws eks update-kubeconfig --region us-east-1 --name ex-EKS-Managed-Node-Group
```

In the above command, ```us-east-1``` is the AWS region where the EKS cluster is created. The us-east-1 AWS region is defaulted in the terraform code. If you want to change to a different region, please update the region in the appropriate terraform files before running the ```terraform apply --auto-approve``` command

More information about [Creating or updating a kubeconfig file for an Amazon EKS cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)

### Step05:
In this step, we will confirm we can run ```kubectl``` commands to interact with the EKS cluster. On your workstation/desktop/laptop, open a command prompt or terminal, then run the below command.
```hcl
kubectl get nodes
```
If your setup is successful, you will see sample output like below, showing the EKS managed nodes in your EKS managed cluster.

<img src="https://github.com/rajiv-sudo/eks-kubernetes-series/blob/main/images/kubectl-get-nodes-output.png?raw=true" alt="Link" width="600"/>

# Summary
In this example, we saw how you can use the ```Intel Cloud Optimization Modules for Terraform - Amazon EKS Module``` to create the EKS cluster and how you can setup your workstation or laptop to connect to this EKS cluster and run kubernetes kubectl commands.
