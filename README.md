CAPI and CAPZ demo
==================

Test environment used for management cluster and cli tools: Windows 11 + WSL2 + Ubuntu 20.04.2 LTS

Prerequisites
-------------

* Docker Desktop (or somewhere to run Kind)
* Install `kubectl` in your local environment
* Install `kind` in your Docker environment
* Bash-like shell environment to run the demo steps (e.g. WSL, Git Bash)

Management Cluster Setup (with Azure provider)
----------------------------------------------

```sh
# Install kind
GO111MODULE="on" go get sigs.k8s.io/kind@v0.11.1
kind version

# Install clusterctl
CLUSTERCTL_VERSION=v0.4.1
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/${CLUSTERCTL_VERSION}/clusterctl-linux-amd64 -o clusterctl
chmod +x ./clusterctl
sudo mv ./clusterctl /usr/local/bin/clusterctl
clusterctl version

# Create initial management cluster in kind
kind create cluster
```

You will see:

```sh
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind
```

Now check the cluster:

```sh
kubectl get nodes
kubectl cluster-info

# Azure provider initialization
export AZURE_SUBSCRIPTION_ID="$(az account show -o json | jq -r '.id')"

# Create an Azure Service Principal
az ad sp create-for-rbac --role contributor --name http://capz-demo > capz-demo-sp.json
export AZURE_TENANT_ID="$(az account show -o json | jq -r '.tenantId')"
export AZURE_CLIENT_ID="$(jq -r .appId < capz-demo-sp.json)"
export AZURE_CLIENT_SECRET="$(jq -r .password < capz-demo-sp.json)"

# Base64 encode the variables
export AZURE_SUBSCRIPTION_ID_B64="$(echo -n "$AZURE_SUBSCRIPTION_ID" | base64 | tr -d '\n')"
export AZURE_TENANT_ID_B64="$(echo -n "$AZURE_TENANT_ID" | base64 | tr -d '\n')"
export AZURE_CLIENT_ID_B64="$(echo -n "$AZURE_CLIENT_ID" | base64 | tr -d '\n')"
export AZURE_CLIENT_SECRET_B64="$(echo -n "$AZURE_CLIENT_SECRET" | base64 | tr -d '\n')"

# Settings needed for AzureClusterIdentity used by the AzureCluster
export AZURE_CLUSTER_IDENTITY_SECRET_NAME="cluster-identity-secret"
export CLUSTER_IDENTITY_NAME="cluster-identity"
export AZURE_CLUSTER_IDENTITY_SECRET_NAMESPACE="default"

# Create a secret to include the password of the Service Principal identity created in Azure
# This secret will be referenced by the AzureClusterIdentity used by the AzureCluster
kubectl create secret generic "${AZURE_CLUSTER_IDENTITY_SECRET_NAME}" --from-literal=clientSecret="${AZURE_CLIENT_SECRET}"

# Enable feature flags for AKS
export EXP_MACHINE_POOL=true
export EXP_AKS=true

# Finally, initialize the management cluster
clusterctl init --infrastructure azure
```

You will see:

```sh
Fetching providers
Installing cert-manager Version="v1.4.0"
Waiting for cert-manager to be available...
Installing Provider="cluster-api" Version="v0.4.2" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v0.4.2" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v0.4.2" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-azure" Version="v0.5.2" TargetNamespace="capz-system"

Your management cluster has been initialized successfully!

You can now create your first workload cluster by running the following:

  clusterctl generate cluster [name] --kubernetes-version [version] | kubectl apply -f -
```

Check the management cluster objects and state:

```sh
kubectl get pods,deployments -A
```

Wait until all pods are `Running`.

Workload Cluster Setup (IaaS on Azure VMs)
------------------------------------------

```sh
# Name of the Azure datacenter location. Change this value to your desired location.
export AZURE_LOCATION="australiaeast"

# Select VM types.
export AZURE_CONTROL_PLANE_MACHINE_TYPE="Standard_D2s_v3"
export AZURE_NODE_MACHINE_TYPE="Standard_D2s_v3"

clusterctl generate cluster capi-quickstart \
  --kubernetes-version v1.21.2 \
  --control-plane-machine-count=3 \
  --worker-machine-count=2 \
  > capi-quickstart.yaml

kubectl apply -f capi-quickstart.yaml
```

Check the status of the IaaS K8s cluster and wait until it is ready to access:

```sh
kubectl get cluster -A
clusterctl describe cluster capi-quickstart   # this shows a hierachical view of dependencies

kubectl get AzureMachineTemplate
kubectl get AzureMachines
kubectl describe AzureMachine <machine-name>

kubectl get kubeadmcontrolplane -A  # should show 3 replicas a READY after some time
kubectl describe kubeadmcontrolplane capi-quickstart
```

Get the Kube config for the workload cluster:

```sh
clusterctl get kubeconfig capi-quickstart > capi-quickstart.kubeconfig

# Install CNI provider (nodes won't be ready until we install a CNI plugin is deployed)
kubectl --kubeconfig=./capi-quickstart.kubeconfig \
  apply -f https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-azure/master/templates/addons/calico.yaml

kubectl --kubeconfig=./capi-quickstart.kubeconfig get nodes
# Wait until they are all Ready
```

Deploy a workload onto the workload cluster
-------------------------------------------

```sh
kubectl apply --kubeconfig=./capi-quickstart.kubeconfig \
    -f https://raw.githubusercontent.com/lastcoolnameleft/kubernetes-examples/master/service/podinfo-external-lb.yaml

kubectl --kubeconfig=./capi-quickstart.kubeconfig get svc
```

Browse to: `http://<svc_ip>:9898`

Scaling workload nodes
----------------------

* Edit `capi-quickstart.yaml`
* Change `replicas: 3` to `replicas: 2` under `MachineDeployment`
* `kubectl apply -f capi-quickstart.yaml`
* `watch kubectl --kubeconfig=./capi-quickstart.kubeconfig get nodes`

Upgrading (Workload K8s version)
--------------------------------

With CAPZ, we use golden images that are created and pushed to the Azure Marketplace.
You can create and publish your own images.

Control Plane upgrade (keep within 2 minor versions of worker node versions):

* Edit `capi-quickstart.yaml`
* First upgrade control plane under `KubeadmControlPlane`
* Change `version: v1.21.2` to `version: v1.22.0`
* `kubectl apply -f capi-quickstart.yaml`
* `watch kubectl --kubeconfig=./capi-quickstart.kubeconfig get nodes`

Worker node upgrade (must be <= control plane version):

* Next upgrade the workload machines under `MachineDeployment`
* Change `version: v1.21.2` to `version: v1.22.0`
* `kubectl apply -f capi-quickstart.yaml`
* `watch kubectl --kubeconfig=./capi-quickstart.kubeconfig get nodes`

Workload Cluster Setup (Managed Kubernetes - AKS)
-------------------------------------------------

```sh
# Kubernetes values
export CLUSTER_NAME="capi-aks"
export WORKER_MACHINE_COUNT=2
export KUBERNETES_VERSION="v1.19.13"

# Azure values
export AZURE_LOCATION="australiaeast"
export AZURE_RESOURCE_GROUP="${CLUSTER_NAME}"
export AZURE_SUBSCRIPTION_ID="$(az account show -o json | jq -r '.id')"
export AZURE_NODE_MACHINE_TYPE="Standard_D2s_v3"

clusterctl generate cluster ${CLUSTER_NAME} --kubernetes-version ${KUBERNETES_VERSION} --flavor aks > capi-aks.yaml

# assumes an existing management cluster
kubectl apply -f capi-aks.yaml

# check status of created resources
kubectl get cluster-api -o wide

kubectl get cluster -A    # Wait for PHASE = Provisioned
clusterctl describe cluster ${CLUSTER_NAME}   # this shows a hierachical view of dependencies (wait for READY = True, takes 5 mins)

kubectl get azuremanagedcontrolplane -A
kubectl describe azuremanagedcontrolplane ${CLUSTER_NAME}

clusterctl get kubeconfig ${CLUSTER_NAME} > ${CLUSTER_NAME}.kubeconfig
kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig get nodes
# Wait until they are all Ready
```

Deploy a workload onto the AKS cluster
--------------------------------------

```sh
kubectl apply --kubeconfig=./${CLUSTER_NAME}.kubeconfig \
    -f https://raw.githubusercontent.com/lastcoolnameleft/kubernetes-examples/master/service/podinfo-external-lb.yaml

kubectl --kubeconfig=./${CLUSTER_NAME}.kubeconfig get svc
```

Browse to: `http://<svc_ip>:9898`

Checking all clusters
---------------------

```sh
kubectl get cluster -A

# IaaS clusters
clusterctl describe cluster capi-quickstart
kubectl get kubeadmcontrolplane -A

# Managed clusters (AKS)
clusterctl describe cluster capi-aks
az aks list -o table
```

Backup/restore management state
-----------------------

```sh
mkdir ./backups
clusterctl backup --directory ./backups
# clusterctl restore <cluster-name> --directory ./backups
```

Migrate management cluster from Kind to AKS (workload cluster)
--------------------------------------------------------------

```sh
clusterctl move --to-kubeconfig=capi-aks.yaml --dry-run
clusterctl move --to-kubeconfig=capi-aks.yaml

# Start use AKS as the management cluster
kubectl --kubeconfig=./capi-aks.yaml get pods,deployments -A
kuebctl --kubeconfig=./capi-aks.yaml get cluster -A

# Optional: merge context into your user's kubeconfig
KUBECONFIG=~/.kube/config:./capi-aks.yaml get
kubectl config view --flatten > ./merged.kubeconfig
cp ~/.kube/config ./backup.kubeconfig
mv ./merged.kubeconfig ~/.kube/config
kubectl config use-context capi-aks
kuebctl get cluster -A

# Decommission Kind cluster
kind delete cluster
```

Cleanup
-------

```sh
# Delete the workload clusters
kubectl delete cluster capi-quickstart
kubectl delete cluster capi-aks

# Optional, if not deleting the Kind cluster
kubectl delete secret generic "${AZURE_CLUSTER_IDENTITY_SECRET_NAME}"

# Delete the management cluster
kind delete cluster

# Delete the Azure Service Principal
az ad sp delete --id $AZURE_CLIENT_ID

# Clean up local files (gitignored)
rm capz-demo-sp.json
rm *.kubeconfig
rm *.yaml
rm -rf ./backups
```

Resources
---------

* https://kind.sigs.k8s.io/docs/user/quick-start/
* https://cluster-api.sigs.k8s.io/user/quick-start.html
* https://github.com/kubernetes-sigs/cluster-api-provider-azure/tree/master/templates
* https://github.com/orgs/kubernetes-sigs/projects/4
* https://azurearcjumpstart.io/azure_arc_jumpstart/azure_arc_k8s/cluster_api/capi_azure/
* https://capz.sigs.k8s.io/topics/managedcluster.html
