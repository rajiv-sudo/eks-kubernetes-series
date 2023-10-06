# Summary
In this example, we will use an EBS volume to mount inside our pods within the EKS cluster for storage persistance. We will use the [Amazon EBS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html). The steps used in this example is based on AWS documentation provided within [Amazon EBS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html).

# Pre-requisites
1. No addtional pre-requisites needed for this example. If you completed all the steps in [01 - Setting up and connecting to an Amazon EKS Cluster](https://github.com/rajiv-sudo/eks-kubernetes-series/tree/main/Examples/01-EKS-setup-and-connect), then all required pre-requisites for this example are already satisfied

# Architecture
The architecture diagram is showing our EKS managed kubernetes cluster in AWS. The user connects to the EKS cluster via the EKS managed control plane. The cluster is setup with 2 worker nodes within an auto scaling group spread across different availability zones.

There is an EBS volume in the one of the availability zones. We will use the Amazon EBS CSI driver to mount this EBS volume inside a pod of the kubernetes cluster for persistent storage.
***Note*** - EBS volume can be only attached to instances that are in the same Availability Zone

**Architecture - EKS Cluster with Amazon EBS CSI Driver**
<img src="https://github.com/rajiv-sudo/eks-kubernetes-series/blob/main/images/EBS-CSI-for-EKS.png?raw=true" alt="Link" width="600"/>

# Steps
Complete the pre-requisite steps mentioned in the Pre-requisites section. Once the pre-requisites steps are completed, follow the steps below.

### Step01:
We will use the Amazon EBS CSI plugin documentation from AWS to setup the necessary IAM roles and create the EKS addon for EBS that will manage attaching persistent EBS volumes to pods.

The Amazon EBS CSI plugin requires IAM permissions to make calls to AWS APIs on your behalf. We will follow the documentation [Creating the Amazon EBS CSI driver IAM role](https://docs.aws.amazon.com/eks/latest/userguide/csi-iam-role.html) to create the role and policy. Run the following commands.
```
aws eks describe-cluster --name ex-EKS-Managed-Node-Group --query "cluster.identity.oidc.issuer" --output text
```
**change the cluster name ```ex-EKS-Managed-Node-Group``` with your own cluster name**
The output looks like this.
```https://oidc.eks.us-east-1.amazonaws.com/id/0F752A3D6B506C15A9AC5B3B9B49258C```

Create the IAM role, granting the AssumeRoleWithWebIdentity action.
Copy the following contents to a file that's named ```aws-ebs-csi-driver-trust-policy.json```. Replace ```111122223333``` with your account ID. Replace ```EXAMPLED539D4633E53DE1B71EXAMPLE``` and ```region-code``` with the values returned in the previous step. If your cluster is in the AWS GovCloud (US-East) or AWS GovCloud (US-West) AWS Regions, then replace ```arn:aws:``` with ```arn:aws-us-gov:```.

~~~
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com",
          "oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
        }
      }
    }
  ]
}
~~~

Below screenshot shows the places highlighted where you need to update with your own values.
<img src="https://github.com/rajiv-sudo/eks-kubernetes-series/blob/main/images/example-02-iam-role.png?raw=true" alt="Link" width="600"/>


Create the role. Make sure the same role name does not exist. Run the command below.

```aws iam create-role --role-name AmazonEKS_EBS_CSI_DriverRole --assume-role-policy-document file://"aws-ebs-csi-driver-trust-policy.json"```

Attach the required AWS managed policy to the role with the following command. If your cluster is in the AWS GovCloud (US-East) or AWS GovCloud (US-West) AWS Regions, then replace ```arn:aws:``` with ```arn:aws-us-gov:```.

```aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy  --role-name AmazonEKS_EBS_CSI_DriverRole```

Now that we have created the Amazon EBS CSI driver IAM role, we can continue to Adding the Amazon EBS CSI driver add-on. When we deploy the plugin in that procedure, it creates and is configured to use a service account that's named ebs-csi-controller-sa. The service account is bound to a Kubernetes clusterrole that's assigned the required Kubernetes permissions.

### Step02:
In this step, we will add the Amazon EBS CSI driver as an Amazon EKS add-on. We will follow the documentation [Managing the Amazon EBS CSI driver as an Amazon EKS add-on](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html). You can explore additional options in this documentation.

**To add the Amazon EBS CSI add-on using the AWS CLI**.
Run the following command. Replace ```my-cluster``` with the name of your cluster, ```111122223333``` with your account ID, and ```AmazonEKS_EBS_CSI_DriverRole``` with the name of the role that was created earlier. If your cluster is in the AWS GovCloud (US-East) or AWS GovCloud (US-West) AWS Regions, then replace ```arn:aws:``` with ```arn:aws-us-gov:```.

```aws eks create-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver  --service-account-role-arn arn:aws:iam::111122223333:role/AmazonEKS_EBS_CSI_DriverRole```

### Step03:
In this step, we will deploy a sample application and verify that the CSI driver is working. Detailed steps are in AWS documentation for [Deploy a sample application](https://docs.aws.amazon.com/eks/latest/userguide/ebs-sample-app.html)

This procedure uses the Dynamic volume provisioning example from the Amazon EBS Container Storage Interface (CSI) driver GitHub repository to consume a dynamically provisioned Amazon EBS volume.

##### Step03-a
Clone the Amazon EBS Container Storage Interface (CSI) driver GitHub repository to your local system.
```git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git```

Navigate to the dynamic-provisioning example directory.
```cd aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/```

##### Step03-b
Deploy the ebs-sc storage class, ebs-claim persistent volume claim, and app sample application from the manifests directory.
```kubectl apply -f manifests/```

##### Step03-c
Describe the ebs-sc storage class.
```kubectl describe storageclass ebs-sc```
An example output is as follows.
~~~
Name:            ebs-sc
IsDefaultClass:  No
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"ebs-sc"},"provisioner":"ebs.csi.aws.com","volumeBindingMode":"WaitForFirstConsumer"}

Provisioner:           ebs.csi.aws.com
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
~~~

##### Step03-d
Watch the Pods in the default namespace. After a few minutes, the app Pod's status changes to Running.
```kubectl get pods --watch```

##### Step03-e
Enter Ctrl+C to return to a shell prompt. List the persistent volumes in the default namespace. Look for a persistent volume with the ```default/ebs-claim``` claim.
```kubectl get pv```
An example output is as follows.
~~~
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
pvc-37717cd6-d0dc-11e9-b17f-06fad4858a5a   4Gi        RWO            Delete           Bound    default/ebs-claim   ebs-sc                  30s
~~~

##### Step03-f
Describe the persistent volume. Replace ```pvc-37717cd6-d0dc-11e9-b17f-06fad4858a5a``` with the value from the output in the previous step.
```kubectl describe pv pvc-37717cd6-d0dc-11e9-b17f-06fad4858a5a```

An example output is as follows.
~~~
Name:              pvc-37717cd6-d0dc-11e9-b17f-06fad4858a5a
Labels:            <none>
Annotations:       pv.kubernetes.io/provisioned-by: ebs.csi.aws.com
Finalizers:        [kubernetes.io/pv-protection external-attacher/ebs-csi-aws-com]
StorageClass:      ebs-sc
Status:            Bound
Claim:             default/ebs-claim
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          4Gi
Node Affinity:
  Required Terms:
    Term 0:        topology.ebs.csi.aws.com/zone in [region-code]
Message:
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            ebs.csi.aws.com
    VolumeHandle:      vol-0d651e157c6d93445
    ReadOnly:          false
    VolumeAttributes:      storage.kubernetes.io/csiProvisionerIdentity=1567792483192-8081-ebs.csi.aws.com
Events:                <none>
~~~
The Amazon EBS volume ID is the value for VolumeHandle in the previous output.

##### Step03-g
Verify that the Pod is writing data to the volume.
```kubectl exec -it app -- cat /data/out.txt```

An example output is as follows.
~~~
Wed May 5 16:17:03 UTC 2021
Wed May 5 16:17:08 UTC 2021
Wed May 5 16:17:13 UTC 2021
Wed May 5 16:17:18 UTC 2021
[...]
~~~

##### Step03-h
After you're done, delete the resources for this sample application.
```kubectl delete -f manifests/```

# Summary
In this example, we saw how you can use the [Amazon EBS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) to attach EBS persistent volumes to your pods in an EKS cluster.
