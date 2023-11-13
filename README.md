# Amazon EKS-Kubernetes-Series
Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. It provides a flexible and highly scalable infrastructure for running and coordinating container workloads across a cluster of machines. Kubernetes simplifies application management, enhances resource utilization, and ensures high availability and fault tolerance in modern cloud-native environments.

Customers prefer using managed Kubernetes services because they offload the complex operational tasks of cluster management, including updates, scaling, and high availability, to cloud providers, allowing teams to focus on developing and deploying applications. Managed Kubernetes also offers enhanced security, reliability, and performance, with the added benefit of cost savings by eliminating the need for in-house Kubernetes expertise and infrastructure maintenance.

Amazon Elastic Kubernetes Service (Amazon EKS) is a managed Kubernetes service provided by Amazon Web Services (AWS). It simplifies the deployment, scaling, and management of containerized applications using Kubernetes, offering a highly available and secure platform for running container workloads on AWS infrastructure.

In this series, we will explore different setups, configurations and code modules that show how customers can get started with Amazon EKS and how they can build features and applications.

**Here are the modules available in the series:**

## [01 - Setting up and connecting to an Amazon EKS Cluster](https://github.com/rajiv-sudo/eks-kubernetes-series/tree/main/Examples/01-EKS-setup-and-connect)
In this example, we will create an Amazon EKS cluster using [Intel Optimized Cloud Modules for Terraform](https://github.com/intel/terraform-intel-aws-eks/tree/main/Examples/EKS_Managed_Node_Group). Then we will show how to connect to the kubernestes cluster from the user's computer using kubectl.

## [02 - Using Amazon EBS persistent storage inside Amazon EKS Kubernetes Cluster](https://github.com/rajiv-sudo/eks-kubernetes-series/tree/main/Examples/02-EBS-CSI-for-EKS)
In this example, we will use an EBS volume to mount inside our pods within the EKS cluster for storage persistance. We will use the [Amazon EBS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html). The steps used in this example is based on AWS documentation provided within [Amazon EBS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html).

## [03 - Using Intel's Node Feature Discovery for the right placement of Gen AI Fastchat Application](https://github.com/rajiv-sudo/eks-kubernetes-series/tree/main/Examples/03-Intel-Node-Feature-Discovery)
In this example, we will use a single EKS cluster with different types of worker nodes and place the right workload on the right node based on the node's hardware features.

[04 - Using Intel's 4th gen Xeon to run Fastchat using LLM and exposed with Web UI](https://github.com/rajiv-sudo/eks-kubernetes-series/tree/main/Examples/04-GenAI-Fastchat-WebUI)
In this example, we will demonstrate how you can use Intel's 4th generation Xeon scalable processors with AMX to run LLM model based chatbots, without having to use expensive accelerators like GPU.