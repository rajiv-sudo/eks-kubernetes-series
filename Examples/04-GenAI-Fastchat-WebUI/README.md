# Summary
In this example, we will show how you can implement [Fastchat](https://github.com/lm-sys/FastChat) on an Amazon EKS Kubernetes cluster. FastChat is an open platform for training, serving, and evaluating large language model based chatbots. We will implement the vicuna-7b-v1.5 seven billion parameter model from [Hugging Face Repo](https://huggingface.co/lmsys/vicuna-7b-v1.5). The chat experience will be surfaced with an url that users can oput on the browser and beable to interact with the chatbot. This will be built on an EKS cluster running on Intel's 4th generation of Xeon scalable processors called Sappire Rapids and Intel's Extension of PyTorch optimized to use the Advanced Matrix Extension (AMX) hardware accelerator.

# Pre-requisites
1. No addtional pre-requisites needed for this example. Complete all the steps in [01 - Setting up and connecting to an Amazon EKS Cluster](https://github.com/rajiv-sudo/eks-kubernetes-series/tree/main/Examples/01-EKS-setup-and-connect)for setting up the EKS cluster and connect to the EKS cluster.

**Please Note**
We have used one node groups in the EKS cluster.This node group is using the C7i.8xlarge EC2 instance type. Our EKS cluster has a minimum size of 2 nodes, desired size of 2 nodes, and a maximum size of 3 nodes. When the cluster gets created, it will be created with 2 nodes of C7i.8xlarge instance type. The C7i.8xlarge instance type is based on Intel's 4th generation of Xeon scalable processors called Sappire Rapids and has AMX hardware accelerator.

The changes include the following:
1. Updating the instance types in the locals definition
```hcl
instance_types = ["c7i.8xlarge"]
```
2. Updating the auto-scaling group configuration in the ```default-node-group``` definition
```hcl
min_size     = 2
max_size     = 3
desired_size = 2
```

# Architecture
The architecture diagram is showing our EKS managed kubernetes cluster in AWS. The user connects to the EKS cluster via the EKS managed control plane. The cluster is setup with 2 worker nodes with the EC2 instance type being C7i.8xlarge spread across two different availability zones.

The Fastchat controller, Fastchat worker, and the Gradio web server are deployed using YAML manifest files as kubernetes deployment objects. Each of these deployments have one replica pod deployed for demo purposes.

The deployments for Fastchat controller and Fastchat worker are exposed using Kubernetes ClusterIP services. The Gradio web server deployment is exposed using a NodePort kubernetes service. The user can put the url on their browser and be able to interact with the fastchat application from the browser.

**Architecture - EKS Cluster deployed with Generative AI Fastchat application**
<img src="https://github.com/rajiv-sudo/eks-kubernetes-series/blob/main/images/GenAI-Fastchat-WebUI.png?raw=true" alt="Link" width="600"/>

# Steps
Complete the pre-requisite steps mentioned in the Pre-requisites section. Once the pre-requisites steps are completed, follow the steps below.

### Step01:
Let's make sure we are able to connect to the EKS cluster and run kubectl commands.

```kubectl get node -o=wide```

The output will look similar to below.
```
NAME                            STATUS   ROLES    AGE   VERSION                INTERNAL-IP     EXTERNAL-IP    OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-172-31-37-55.ec2.internal    Ready    <none>   18h   v1.24.17-eks-43840fb   172.31.37.55    3.80.238.20    Amazon Linux 2   5.10.192-183.736.amzn2.x86_64   containerd://1.6.19
ip-172-31-40-249.ec2.internal   Ready    <none>   18h   v1.24.17-eks-43840fb   172.31.40.249   52.91.29.125   Amazon Linux 2   5.10.192-183.736.amzn2.x86_64   containerd://1.6.19
```
### Step02
Deploy Fastchat controller
```
kubectl apply -f fastchat-controller-deployment.yaml
```
The file ```fastchat-controller-deployment.yaml``` is available as part of this Github repo under examples folder called ```04-GenAI-Fastchat-WebUI```. Copy this file to the folder location from where you are running the ```kubectl``` commands.

***Note***
It might take a few minutes to download the container image from Docker Hub. Run the following command to watch the status of the Fastchat controller pod. Wait and do not proceed to next step until the status changes to ```Running```
```
kubectl get pods --watch
```

### Step03
Deploy Fastchat controller ClusterIP service
```
kubectl apply -f fastchat-controller-service.yaml
```
The file ```fastchat-controller-service.yaml``` is available as part of this Github repo under examples folder called ```04-GenAI-Fastchat-WebUI```. Copy this file to the folder location from where you are running the ```kubectl``` commands.

### Step04
Deploy Fastchat model worker
```
kubectl apply -f fastchat-model-worker-deployment.yaml
```
The file ```fastchat-model-worker-deployment.yaml``` is available as part of this Github repo under examples folder called ```04-GenAI-Fastchat-WebUI```. Copy this file to the folder location from where you are running the ```kubectl``` commands.

Run the below command to check that the Fastchat model worker pod is in ```Running``` status. Wait until the status changes to ```Running```.

```
kubectl get pods --watch
```

### Step05
Deploy Fastchat model worker ClusterIP service
```
kubectl apply -f fastchat-model-worker-service.yaml
```
The file ```fastchat-model-worker-service.yaml``` is available as part of this Github repo under examples folder called ```04-GenAI-Fastchat-WebUI```. Copy this file to the folder location from where you are running the ```kubectl``` commands.

### Step06
Deploy Gradio Web Server
```
kubectl apply -f gradio-web-server-deployment.yaml
```
The file ```gradio-web-server-deployment.yaml``` is available as part of this Github repo under examples folder called ```04-GenAI-Fastchat-WebUI```. Copy this file to the folder location from where you are running the ```kubectl``` commands.

Run the below command to check that the Fastchat model worker pod is in ```Running``` status. Wait until the status changes to ```Running```.

```
kubectl get pods --watch
```

### Step07
Deploy Gradio Web Server Nodeport service
```
kubectl apply -f gradio-service.yaml
```
The file ```gradio-service.yaml``` is available as part of this Github repo under examples folder called ```04-GenAI-Fastchat-WebUI```. Copy this file to the folder location from where you are running the ```kubectl``` commands.

### Step08:
Here's some background on the Generative AI Fastchat application. The Fastchat application is based on the [Fastchat open source project](https://github.com/lm-sys/FastChat).

We have a Docker image that is created on Ubuntu-22.04. The Docker image has all the softwares and libraries installed for using the Fastchat application and is optimized on Intel's 4th gen Xeon scalable processor and AMX accelerator for BF16 (Brain Float 16 bit floating point precision) using Intel's extension of Pythorch. The docker image [mandalrajiv/fastchat-intel-extension-pytorch](https://hub.docker.com/repository/docker/mandalrajiv/fastchat-intel-extension-pytorch/general) is public and will be used in the creating the Fastchat controller, Fastchat model worker, and Gradion Webserver pods on the C7i.8xlarge EC2 instance/s.

Here's the **Dockerfile** for reference. This is the Dockerfile that has been used to create the docker image [mandalrajiv/fastchat-intel-extension-pytorch](https://hub.docker.com/repository/docker/mandalrajiv/fastchat-intel-extension-pytorch/general) on Docker Hub.
```
FROM ubuntu:22.04 as base

# Install prerequisities
RUN apt-get update
RUN apt-get -y install python3 python3-pip python-is-python3 net-tools

# Install Install Intel's Extension for PyTorch
RUN pip3 install intel_extension_for_pytorch

# Install Jinja2 version 3.1.2
RUN pip3 install jinja2==3.1.2

# Install Fastchat
RUN pip3 install fschat

# Install FastChat - fschat[model_worker,webui]
RUN pip3 install "fschat[model_worker,webui]==0.2.30"

# Install pydantic==1.10.13
RUN pip3 install pydantic==1.10.13

# Define environment variable CPU_ISA=amx for using Intel AI Accelerator AVX512_BF16/AMX to accelerate CPU inference.
ENV CPU_ISA=amx
```

### Step09:
Let's check that the Gradio webserver service is running as a NodePort service. Run the below command.
```
kubectl get svc
```

The outpot will look something like this.
```
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
fastchat-controller     ClusterIP   10.100.81.198    <none>        21001/TCP        10m
fastchat-model-worker   ClusterIP   10.100.251.229   <none>        21002/TCP        108s
gradio-service          NodePort    10.100.73.91     <none>        7860:30010/TCP   28s
kubernetes              ClusterIP   10.100.0.1       <none>        443/TCP          159m
```

As you can see, the ```gradio-service``` is running as  aNodePort service on port ```30010```

### Step10:
Get the External Public IP addresses for the EKS cluster worker nodes. Run the command below.
```
kubectl get nodes -o=wide
```

You will see the output like this.
```
NAME                            STATUS   ROLES    AGE   VERSION                INTERNAL-IP     EXTERNAL-IP    OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-172-31-34-168.ec2.internal   Ready    <none>   89m   v1.24.17-eks-4f4795d   172.31.34.168   54.90.123.76   Amazon Linux 2   5.10.198-187.748.amzn2.x86_64   containerd://1.6.19
ip-172-31-37-162.ec2.internal   Ready    <none>   89m   v1.24.17-eks-4f4795d   172.31.37.162   3.81.124.65    Amazon Linux 2   5.10.198-187.748.amzn2.x86_64   containerd://1.6.19
```

The public IP address are shown in the ```EXTERNAL-IP``` column. Te test the demo, you can seklect any of the IP addresses. For our demo, we will select the first External IP ```54.90.123.76```.

Please note, in your EKS cluster you IP address will be different from the setup in this demo.

### Step11:
You will need to add a rule in the AWS EC2 instance security group of the worker nodes that will allow TCP traffic from your computer or laptop to the worker node's port 30010. If this step is not done, then the Fastchat web ui will not work.

Go to the AWS console, navigate to the security group of your worker nodes and add the rule. The rule will look like this. The redacted portion of the screen show is the public IP address of your computer or laptop.

<img src="https://github.com/rajiv-sudo/eks-kubernetes-series/blob/main/images/security-group-nodeport.png?raw=true" alt="Link" width="600"/>

### Step12:
Access the chatbot on the browser. Open your favorite browser and type in the url in the form of ```<Public-IP:30010```. In our case, the url will be ```54.90.123.76:30010```

The chatbot web UI should open up on the browser.

It should look like this.

<img src="https://github.com/rajiv-sudo/eks-kubernetes-series/blob/main/images/Gradio-Web-UI.png?raw=true" alt="Link" width="600"/>

### Step13:
Let's interact with the chatbot web UI.
Enter a prompt, for example, in our demo we used a prompt ```write me a 100 word paragraph to describe what residential customers should consider solar power```. After entering the prompt, click the ```Send``` button.

In our case the output is like this.
```Residential customers should consider solar power as a reliable and sustainable energy solution for their homes. With solar power, customers can generate their own electricity, reducing their dependence on the grid and lowering their energy bills. Solar power is also a renewable energy source, which means it is environmentally friendly and helps to reduce greenhouse gas emissions. Additionally, solar power systems can be customized to meet the specific energy needs of a home, and they can be installed with little to no upfront cost through a solar panel leasing or power purchase agreement (PPA) program. Overall, solar power is a smart investment for residential customers who want to save money, reduce their carbon footprint, and increase the energy independence of their homes.```

### Step14:
This concludes the setup and demonstration of this use case. Please run ```terraform destroy``` to delete the resources and avoid incurring cloud costs.
```hcl
terraform destroy
```

# Summary
In this example, we saw how you can use Intel's 4th generation Xeon scalable processors with AMX to run LLM model based chatbots, without having to use expensive accelerators like GPU.

# References
1. Fastchat - FastChat is an open platform for training, serving, and evaluating large language model based chatbots - https://github.com/lm-sys/FastChat
