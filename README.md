# README file

This file documents how one could implement and use `Cluster API` in a PoC approach

## Implementation steps

### Prerequisites

* [docker](https://docs.docker.com/engine/install/ubuntu/)
* [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-on-linux)
* [clusterctl](https://release-0-3.cluster-api.sigs.k8s.io/user/quick-start#install-clusterctl)

### Installation of `Cluster API` components

* provision a `kind` cluster where `Cluster API` components will be deployed
```bash
# create the kind cluster using a custom config allowing access to the Docker socket
# (which is required for the CAPD provier to work properly)
kind create cluster --name 1cp1w-extramount-docker-socket --config ./manifests/1cp1w-extramount-docker-socket.yaml --kubeconfig $HOME/.kube/1cp1w-extramount-docker-socket.yaml
```

* deploy `Cluster API` components (within a docker infra context)
```bash
# Enable the experimental Cluster topology feature.
export CLUSTER_TOPOLOGY=true

# Enable preflight checks on MachineSets
export EXP_MACHINE_SET_PREFLIGHT_CHECKS=true

# Initialize the management cluster
clusterctl init --infrastructure docker
```

### Creating workload clusters

```bash
# generating the manifest file
clusterctl generate cluster test --flavor development \
  --kubernetes-version v1.31.2 \
  --control-plane-machine-count=1 \
  --worker-machine-count=1 \
> ./manifests/test.yaml

# creating the workload cluster
kubectl apply -f ./manifests/test.yaml

# monitoring the workload cluster provisioning
clusterctl describe cluster -n default test --echo --grouping=false --show-conditions=all

# when things stabilize and nodes are reported with 'Node condition Ready is False'
# deploy a CNI
# see here: https://cluster-api.sigs.k8s.io/user/quick-start#deploy-a-cni-solution
```

### Accessing the workload cluster

```bash
# get the kubeconfig file
clusterctl get kubeconfig -n default test > ~/.kube/test.yaml
```

### Deploying a CNI on the workload cluster

> see details [here](https://cluster-api.sigs.k8s.io/user/quick-start#deploy-a-cni-solution)

```bash
kubectl --kubeconfig=$HOME/.kube/test.yaml apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

After the CNI is deployed, the status of the nodes of the workload cluster will become `Ready`:  

```bash
kubectl --kubeconfig=$HOME/.kube/test.yaml get nodes
```

## Some useful commands about `Cluster API`

```bash
# list of available providers
clusterctl config repositories

# get the list of variables expected by a provider (and a pointer to github repo)
# clusterctl generate provider --<provider-type> <provider-name> --describe
clusterctl generate provider --infrastructure docker --describe

# get the list of variables that are required by a provider
# clusterctl generate cluster my-cluster --<provider-type> <provider-name> --list-variables
clusterctl generate cluster my-cluster --infrastructure vsphere:v1.10.2 --list-variables

# get the list of available providers
clusterctl config repositories
 
# get the list of variables exposed by a provider (and pointer to github repo)
# clusterctl generate provider --<provider-type> <provider-name> --describe
clusterctl generate provider --infrastructure openstack --describe
 
# get the list of variables that are required by a provider
clusterctl generate cluster my-cluster --infrastructure vsphere:v1.10.2 --list-variables
 
# generate a manifest file to build a workload cluster
# clusterctl generate cluster <cluster_name> \
#            --kubernetes-version v<major.minor.patch> \
#            --infrastructure aws:v0.4.1 \
#            --flavor development \
#            --control-plane-machine-count=<n> \
#            --worker-machine-count=<n> \
#            > manifest.yaml
clusterctl generate cluster my-cluster --kubernetes-version v1.28.0 --control-plane-machine-count=3 --worker-machine-count=3 > my-cluster.yaml
```