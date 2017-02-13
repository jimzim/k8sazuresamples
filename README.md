
# OSS App on K8s on azure

First, you will need to deploy an ACS cluster running Kubernetes.  Since we are also going to install Deis Workflow, I am going to suggest we go through the install guide of K8s on Azure on Deis.com. 

Install DEIS Workflow on ACS with Kubernetes - <https://deis.com/docs/workflow/quickstart/provider/azure-acs/boot/>

' create premium storage account
https://docs.microsoft.com/en-us/azure/storage/storage-premium-storage

az storage account create -l <location> -n <account_name> -g <resource_group> --sku <account_sku>


' make sure vhds container is in storage account - bug

' create pvc for fast storage class

' how to debug logs - kk logs kube-controller-manager-k8s-master-512bbf0f-0 -n kube-system
' helm update - helm search stable to make sure it is there

' install mongodb replicaset using StatefulSets
https://github.com/kubernetes/charts/tree/master/incubator/mongodb-replicaset

More on StatefulSets
https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/

