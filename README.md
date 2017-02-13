
# DEIS and MongoDB on K8s on Azure

I wanted to show everyone how to set up an ideal environment for running OSS workloads on Azure.  My favorite stack is the MEAN stack, so i am going to show how to set up an ACS cluster running Kubernetes with both DEIS Workflow and a MongoDB Replicaset installed.

First, you will need to deploy an ACS cluster running Kubernetes.  Since we are also going to install Deis Workflow, I am going to suggest we go through the install guide of K8s on Azure on Deis.com. 

Install DEIS Workflow on ACS with Kubernetes

<https://deis.com/docs/workflow/quickstart/provider/azure-acs/boot/>

If you prefer to set up a cluster without DEIS and Helm, you can go here.

<https://docs.microsoft.com/en-us/azure/container-service/container-service-kubernetes-walkthrough>

Once you are set up, you will want to create a premium storage account using the Azure CLI.  If you prefer to use the portal, make sure to select the premium sku.

```sh
az storage account create -l <location> -n <account_name> -g <resource_group> --sku Premium_LRS
```

Because of a bug in Kubernetes 1.5.1, we will need to create a container named 'vhds' in the storage account you just created.

```
az storage container create -n vhds
```

Azure Storage CLI Reference - <https://docs.microsoft.com/en-us/azure/storage/storage-azure-cli>


Now let's create a PVC on the Kubernetes cluster.  You will need to create a Presistant Volume and then make a claim for the volume to test.  Create a yaml file and name it premdisk.yaml or whatever you want and put this yaml contents into the file.  Make sure to put the correct storage account info below.  In mycase, I named my premium storage account premacctname in the EastUS 
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: fast
provisioner: kubernetes.io/azure-disk
parameters:
  skuName: Premium_LRS
  location: eastus
  storageAccount: premacctname
```

```
kubectl create -f premdisk.yaml
```

Then you need to create a claim.  In this example, I am going to create a 8GB data disk.  Notice the storage class, "fast" matches what I named the volume above.  Create a claim1.json file and insert the contents below.
```json
{
  "kind": "PersistentVolumeClaim",
  "apiVersion": "v1",
  "metadata": {
    "name": "claim1",
    "annotations": {
      "volume.beta.kubernetes.io/storage-class": "fast"
    }
  },
  "spec": {
    "accessModes": [
      "ReadWriteOnce"
    ],
    "resources": {
      "requests": {
        "storage": "8Gi"
      }
    }
  }
}

```

```
kubectl create -f claim1.json
```

While you are waiting for the disk to be formatted ( takes 6-7 minutes), you can monitor progress by running the following commands.

Check to see that the pvc has been created
```
kubectl get pvc
```

View all pods running on the K8s cluster
```
kubectl get pods --all-namespaces
```

If you are waiting a long time, there is probably some issue with creating the disk.  You can view the logs of the controller by calling the following command line.  your kube-controller-manager will have a different name, so be sure to replace your pod name over the one below.
```
kubectl logs kube-controller-manager-k8s-master-512bbf0f-0 -n kube-system
```

Now that we know we can create a disk, let's create a MongoDB Replicaset.  Make sure to update all the Helm charts in the cluster by running the following command.
```
helm update
```

We are going to use the Helm chart found in the incubator folder found in the charts area of Kubernetes.  This install uses StatefulSets which can be very useful when setting up Database clusters that need to be highly available.
https://github.com/kubernetes/charts/tree/master/incubator/mongodb-replicaset

Instead of using the default template, we will need to modiby the values.yaml file in the repo to look like this.  Note:  I changed the StorageClass from default to fast.

```yaml
# Default values for mongodb.
# This is a YAML-formatted file.
# Declare name/value pairs to be passed into your templates.
# name: value

Name: mongodb-replicaset
PeerPort: 27017
Component: "mongodb"
ReplicaSet: "rs0"
ClusterRole: "shardsvr"
Replicas: 3
Image: "mongo"
ImageTag: "3.2"
InstallImage: "gcr.io/google_containers/mongodb-install"
InstallImageTag: "0.3"
ImagePullPolicy: "Always"
Cpu: "100m"
Memory: "512Mi"
Storage: "10Gi"
StorageClass: fast
```

Then when creating the replicaset, use this command:
```
helm install --name my-release -f values.yaml incubator/mongodb-replicaset
```

As long as you have everything set up correctly with Kubernetes and PVCs, this should work straight away.  It will bring up each of the 3 pods in order and will make sure that there is a Pod always running on each of the 3 VMs.

If you want more info on StatefulSets (new in 1.5), that can be found here.
https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/

