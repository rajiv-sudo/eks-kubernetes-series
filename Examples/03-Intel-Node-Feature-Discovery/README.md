# Summary
In this example, we will show how you can use [Intel's Node Feature Discovery Kubernetes Add-on](https://kubernetes-sigs.github.io/node-feature-discovery/v0.14/get-started/index.html) to create a single EKS cluster with different worker node types and be able to use the same EKS cluster to run specific workloads on specific worker node based on the hardware features available on the node. This approach will further reduce and consolidate our EKS control plane cost by running multiple workloads in the same EKS cluster.

# Pre-requisites
1. No addtional pre-requisites needed for this example. Complete all the steps in [01 - Setting up and connecting to an Amazon EKS Cluster](https://github.com/rajiv-sudo/eks-kubernetes-series/tree/main/Examples/01-EKS-setup-and-connect)for setting up the EKS cluster and connect to the EKS cluster.

**One Caveat**
We have use two different node groups in the EKS cluster. One EKS node group will contain Intel 4th gen Xeon Sapphire Rapids based EC2 instance C7i.8xlarge that has the [Advanced Matrix Extension - AMX Accelerator.](https://www.intel.com/content/www/us/en/products/docs/accelerator-engines/advanced-matrix-extensions/overview.html) The other EKS node group will contain general purpose burstable EC2 instance T3.Medium. We will use the C7i.8xlarge worker node to run the Generative AI Fastchat application that will benefit from the Intel 4th gen Xeon Scalable processor with AMX. We will use the T3.Medium worker node to run basic NGINX web server.

The updated ```main.tf``` file under the [Intel® Cloud Optimization Modules for Terraform - Amazon EKS Module](https://github.com/intel/terraform-intel-aws-eks) repo in the ```./examples/EKS_Managed_Node_Group/``` folder is referenced [here](https://github.com/rajiv-sudo/eks-kubernetes-series/tree/main/Examples/03-Intel-Node-Feature-Discovery/main.tf).

The changes include the following:
1. Updating the instance types in the locals definition
```hcl
instance_types = ["c7i.8xlarge"]
```
2. Updating the auto-scaling group configuration in the ```default-node-group``` definition
```hcl
min_size     = 1
max_size     = 2
desired_size = 1
```
3. Adding the second EKS Managed Node Group called ```custom-node-group```
```hcl
custom-node-group = {
      instance_types = ["t3.medium"]
      use_custom_launch_template = false
      disk_size = 50
      min_size     = 1
      max_size     = 2
      desired_size = 1

      # Remote access cannot be specified with a launch template
      remote_access = {
        ec2_ssh_key               = module.key_pair.key_pair_name
        source_security_group_ids = [aws_security_group.remote_access.id]
      }
    }
```

# Architecture
The architecture diagram is showing our EKS managed kubernetes cluster in AWS. The user connects to the EKS cluster via the EKS managed control plane. The cluster is setup with 2 worker nodes within two different auto scaling groups spread across different availability zones.

The ```default-node-group``` has one worker node created on the ```C7i-8xlarge``` EC2 instance. This is the worker node that will be used to run the Generative AI Fastchat application based on matching labels identified by Intel's Node Feature Discovery Kubernetes Add-on.
The ```custom-node-group``` has one worker node created on ```T3.Medium``` EC2 instance. This worker node will be used to run the NGINX web service, which do not need special hardware features like AMX in Intel's 4th gen Xeon Scalable Processor

The ```Node Feature Discovery (NFD) Kubernetes Add-on``` is implemented as a daemonset object. Hence there is one pod within each of the worker nodes that is running NFD.

**Architecture - EKS Cluster with Intel's Node Feature Discovery Kubernetes Add-on**
<img src="https://github.com/rajiv-sudo/eks-kubernetes-series/blob/main/images/Intel-Node-Feature-Discovery.png?raw=true" alt="Link" width="600"/>

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
### Step02:
Install Node Feature Discover by running the command below. The isntructions are given in the [NFD Quick Start Installation](https://kubernetes-sigs.github.io/node-feature-discovery/v0.14/get-started/quick-start.html)
```
kubectl apply -k https://github.com/kubernetes-sigs/node-feature-discovery/deployment/overlays/default?ref=v0.14.3
```

### Step03:
Wait until NFD master and NFD worker are running. Run the command below to verify.
```
kubectl -n node-feature-discovery get ds,deploy
```

### Step04:
Check that NFD feature labels have been created for each of the worker nodes in the EKS cluster. Redirect the output to a file called node-labels.txt
```
kubectl get no -o json | jq '.items[].metadata.labels' > node-labels.txt
```
**The above command uses an utility called jq. If you do not have this utility installed on your system, please install the jq utility for your specific system.**

### Step05:
In the ```node-labels.txt``` file, we will look for the label that contains the ```cpu-cpuid.AMXBF16``` flag. The presence of this flag indicates that this worker node has the AMX accelerator. Run the command below.
```
cat node-labels.txt | grep "cpu-cpuid.AMXBF16"
```
You will see output like below.
```
"feature.node.kubernetes.io/cpu-cpuid.AMXBF16": "true",
```
In this case, it shows that the label is ```feature.node.kubernetes.io/cpu-cpuid.AMXBF16``` and the value for this label is ```true```.

Above command works on Linux. If you are using Windows machine, the equivalent to ```grep``` command is the Windows PowerShell command **```Select-String```**

### Step06:
Let us find the worker node name which contains this label. Run the command below.
```kubectl get nodes -l=feature.node.kubernetes.io/cpu-cpuid.AMXBF16 -o jsonpath='{.items[*].metadata.labels.kubernetes\.io/hostname}{"\n"}'```

You will see output like this, where the column ```NAME``` is the name of the worker node.
```
ip-172-31-40-249.ec2.internal
```
### Step07:
Verify that this node ```ip-172-31-40-249.ec2.internal``` is the EC2 instance that is created on ```C7i.8xlarge```. Run the below command.
```kubectl get nodes -l=kubernetes.io/hostname=ip-172-31-40-249.ec2.internal -o jsonpath='{.items[*].metadata.labels.node\.kubernetes\.io/instance-type}{"\n"}'```

The output will look like below.
```c7i.8xlarge```
This validates that the AMX feature is available on the C7i.8xlarge EC2 instance.

### Step08:
Here's some background on the Generative AI Fastchat application. The Fastchat application is based on the [Fastchat open source project](https://github.com/lm-sys/FastChat).

We have a Docker image that is created on Ubuntu-22.04. The Docker image has all the softwares and libraries installed for using the Fastchat application and is optimized on Intel's 4th gen Xeon scalable processor and AMX accelerator for BF16 (Brain Float 16 bit floating point precision) using Intel's extension of Pythorch. The docker image [mandalrajiv/intel-extension-pytorch-amx](https://hub.docker.com/repository/docker/mandalrajiv/intel-extension-pytorch-amx/general) is public and will be used in the creating the fastchat popd on the C7i.8xlarge EC2 instance based on the node label ```feature.node.kubernetes.io/cpu-cpuid.AMXBF16``` discovered by NFD.

Here's the **Dockerfile** for reference. This is the Dockerfile that has been used to create the docker image [mandalrajiv/intel-extension-pytorch-amx](https://hub.docker.com/repository/docker/mandalrajiv/intel-extension-pytorch-amx/general) on Docker Hub.
```
FROM ubuntu:22.04 as base

# Install prerequisities
RUN apt-get update
RUN apt-get -y install python3 python3-pip python-is-python3 net-tools

# Install Install Intel's Extension for PyTorch
RUN pip3 install intel_extension_for_pytorch

# Install Jinja2 version 3.1.2
RUN pip3 install jinja2==3.1.2

# Install FastChat - fschat[model_worker,webui]
RUN pip3 install "fschat[model_worker,webui]"

# Define environment variable CPU_ISA=amx for using Intel AI Accelerator AVX512_BF16/AMX to accelerate CPU inference.
ENV CPU_ISA=amx

# Set the working directory for logging into the container
WORKDIR /root

# Put the container to infinite sleep so the container does not exit
ENTRYPOINT [ "sleep" ]
CMD [ "infinity" ]
```
### Step09:
Run the Generative AI Fastchat application.
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: fastchat
  name: fastchat
spec:
  containers:
  - image: mandalrajiv/intel-extension-pytorch-amx:v1
    name: fastchat
    resources: {}
  nodeSelector:
    feature.node.kubernetes.io/cpu-cpuid.AMXBF16: "true"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
EOF
```
Check if the pod is created by running this command.
```kubectl get pods```

If the pod is created you will see output like below.
```
NAME       READY   STATUS    RESTARTS   AGE
fastchat   1/1     Running   0          64s
```
### Step10:
Now create the NGINX application. This application pod does not need specialized hardware and can be placed on any worker node in the cluster as deemed by Kubernetes scheduler. Run the below command. Please note that creating the container might take a few mins. Please be patient.
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: feature.node.kubernetes.io/cpu-cpuid.AMXBF16
            operator: DoesNotExist
  containers:
  - name: nginx
    image: nginx
EOF
```

Check if the pod is created by running this command.
```kubectl get pods```

If the pod is created you will see output like below.
```
NAME       READY   STATUS    RESTARTS   AGE
fastchat   1/1     Running   0          36m
nginx      1/1     Running   0          6s
```

### Step11:
Check the worker nodes on which the pods have been scheduled. Run the command below.

```kubectl get pods -o=wide```

You will see out like below.
```
NAME       READY   STATUS    RESTARTS   AGE   IP              NODE                            NOMINATED NODE   READINESS GATES
fastchat   1/1     Running   0          49m   172.31.35.115   ip-172-31-40-249.ec2.internal   <none>           <none>
nginx      1/1     Running   0          49s   172.31.46.58    ip-172-31-37-55.ec2.internal    <none>           <none>
```
As seen above, the fastchat pod is craeted on the C7i.8xlarge EC2 instance which has AMX accelerator. The NGINX pod is created on T3.Medium EC2 instance since it does not require special hardware feature like AMX. We use the ```Node Anti-Affinity``` Kubernetes feature to schedule the NGINX web server pod.

### Step12:
Test the Fastchat application. Log into the fastchat pod using the command below.

```kubectl exec -it fastchat -- /bin/bash```

Once you are at the command prompt within the fastchat container, run the following command to start chatting on the command line.

```python3 -m fastchat.serve.cli --model-path lmsys/vicuna-7b-v1.5 --device cpu```

Once you the chat is ready you should see like below, showing the ```USER``` prompt. The previous command might take a few mins.

```
Downloading (…)neration_config.json: 100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 162/162 [00:00<00:00, 1.14MB/s]
/usr/local/lib/python3.10/dist-packages/transformers/generation/configuration_utils.py:362: UserWarning: `do_sample` is set to `False`. However, `temperature` is set to `0.9` -- this flag is only used in sample-based generation modes. You should set `do_sample=True` or unset `temperature`. This was detected when initializing the generation config instance, which means the corresponding file may hold incorrect parameterization and should be fixed.
  warnings.warn(
/usr/local/lib/python3.10/dist-packages/transformers/generation/configuration_utils.py:367: UserWarning: `do_sample` is set to `False`. However, `top_p` is set to `0.6` -- this flag is only used in sample-based generation modes. You should set `do_sample=True` or unset `top_p`. This was detected when initializing the generation config instance, which means the corresponding file may hold incorrect parameterization and should be fixed.
  warnings.warn(
/usr/local/lib/python3.10/dist-packages/intel_extension_for_pytorch/frontend.py:462: UserWarning: Conv BatchNorm folding failed during the optimize process.
  warnings.warn(
/usr/local/lib/python3.10/dist-packages/intel_extension_for_pytorch/frontend.py:469: UserWarning: Linear BatchNorm folding failed during the optimize process.
  warnings.warn(
USER:
```
### Step13:
Give a prompt to chat. Let's use the prompt ```Write me a 100 word essay on why people should use electric vehicles```. See the prompt and chat output below.

```
USER: Write me a 100 word essay on why people should use electric vehicles
ASSISTANT: Electric vehicles (EVs) are a growing trend in the auto industry due to their numerous benefits. They are more environmentally friendly than traditional gasoline-powered vehicles, as they produce zero tailpipe emissions. This means that they help reduce air pollution, which is a significant contributor to climate change. Additionally, electric vehicles are quieter and smoother in operation, providing a more comfortable and enjoyable driving
```
Press ```Control + C``` to break out of the chat.
Type ```exit``` to break out of the container

### Step14:
This concludes the setup and demonstration of this use case. Please run ```terraform destroy``` to delete the resources and avoid incurring cloud costs.
```hcl
terraform destroy
```

# Summary
In this example, we saw how you can use the [Intel's Node Feature Discovery Kubernetes Add-on](https://kubernetes-sigs.github.io/node-feature-discovery/v0.14/get-started/index.html) in being able to use the single EKS cluster to place the right workload on the right worker node.

# References
1. Intel's Node Feature Discovery - a kubernetes add-on - https://kubernetes-sigs.github.io/node-feature-discovery/stable/get-started/index.html
2. Node Feature Discovery - Quick Start Installation - https://kubernetes-sigs.github.io/node-feature-discovery/v0.14/get-started/quick-start.html
3. Jsonpath in Kubernetes - https://kubernetes.io/docs/reference/kubectl/jsonpath/
4. Fastchat - FastChat is an open platform for training, serving, and evaluating large language model based chatbots - https://github.com/lm-sys/FastChat
