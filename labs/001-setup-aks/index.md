# Azure Console
## Spin up a Kubernetes cluster
In this section, you will be using Cloud Shell to create a Kubernetes cluster.
1. Click the shell icon on the top right of the browser.
2. In the new console create a new Resource Group

```console
az group create --name voting --location westus
```

3. Spin up Kubernetes cluster
```console
az aks create --resource-group voting \
   --name myAKSCluster \
   --node-count 1 \
   --enable-addons monitoring \
   --generate-ssh-keys \
   --node-vm-size Standard_D2s_v3
```

4. Get cluster credentials
```console
az aks get-credentials --resource-group voting \
   --name myAKSCluster
```

Confirm the cluster is running:
```console
kubectl get pods --all-namespaces
```

## Lab Complete!
