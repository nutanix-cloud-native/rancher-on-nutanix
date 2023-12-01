# Rancher on Nutanix Demo Repo

- [Rancher on Nutanix Demo Repo](#rancher-on-nutanix-demo-repo)
  - [Overview](#overview)
  - [PreRequisites](#prerequisites)
  - [Staging Local Workstation and Rancher MCM Environment](#staging-local-workstation-and-rancher-mcm-environment)
    - [Clone Github repository locally and change directory](#clone-github-repository-locally-and-change-directory)
    - [Create Rancher Cloud Credentials Secret for Nutanix Prism Central](#create-rancher-cloud-credentials-secret-for-nutanix-prism-central)
    - [Optional - Upload Generic Cloud Image into Prism Central](#optional---upload-generic-cloud-image-into-prism-central)
  - [QuickStart Examples](#quickstart-examples)
    - [Provision RKE2 Cluster with Multiple Node Pools Helm Global Defaults](#provision-rke2-cluster-with-multiple-node-pools-helm-global-defaults)
    - [Provision K3s Cluster with Multiple Node Pools and Custom Configs](#provision-k3s-cluster-with-multiple-node-pools-and-custom-configs)
  - [Additional Scenarios](#additional-scenarios)
    - [Scaling Existing RKE2 Cluster Example](#scaling-existing-rke2-cluster-example)
    - [Upgrading RKE2 Kubernetes Cluster Version](#upgrading-rke2-kubernetes-cluster-version)
    - [Upgrading RKE2 Node OS Machine Image](#upgrading-rke2-node-os-machine-image)
    - [Provision RKE2 Cluster with 3 Linux VMs](#provision-rke2-cluster-with-3-linux-vms)
      - [Pre-requisites](#pre-requisites)
      - [Provision First RKE2 Control Plane Node](#provision-first-rke2-control-plane-node)
      - [Provision Remaining RKE2 Control Plane Nodes](#provision-remaining-rke2-control-plane-nodes)
    - [Deploy Highly Available Rancher MCM via Helm](#deploy-highly-available-rancher-mcm-via-helm)
    - [Deploy Nutanix CSI Driver via Helm](#deploy-nutanix-csi-driver-via-helm)
  - [Troubleshooting](#troubleshooting)
    - [Debugging Helm Deployments](#debugging-helm-deployments)
  - [Updating Cluster Template Helm Chart Package](#updating-cluster-template-helm-chart-package)

## Overview

The following repo is used to supplement the Rancher on Nutanix Tech Note.

This repo includes an example cluster template that could be used to deploy either RKE2 or K3S clusters with multiple nodepools across one or many Nutanix AHV Clusters.

## PreRequisites

- Existing Rancher MCM Cluster
- Nutanix AHV Node Driver Enabled and Activated within Rancher.
- Existing Nutanix AHV Cluster with IPAM Subnet Enabled.
- Existing Nutanix Prism Project with AHV Cluster and Subnet defined within Prism Central.
- Existing Linux OS Image with Cloud-Init Enabled
- Workstation with `kubectl`,`git` and `helm` cli available
- [Optional] Existing Nutanix Prism Central Category Key `AppTier` should be updated to include the following category values: `Rancher_Controlplane` `Rancher_Etcd` `Rancher_Worker`

Unless told otherwise, all commands listed below should be executed directly from the Kubernetes Cluster Context that is actively hosting the Rancher Multi-Cluster Manager.

## Staging Local Workstation and Rancher MCM Environment

The following deployment examples assume that each command is run from the respective Rancher MCM cluster’s kubectl context.

### Clone Github repository locally and change directory

Prior to proceeding with the next steps, make sure to clone the git repository locally and change directory to newly created repo location.  Upon completion, navigate to Rancher MCM cluster via kubectl and Create the Cloud Credentials needed for provisioning clusters within Nutanix Prism Central.

```bash
git clone https://github.com/jesse-gonzalez/rancher-on-nutanix
cd rancher-on-nutanix
```

### Create Rancher Cloud Credentials Secret for Nutanix Prism Central

Creating this secret is equivalent to providing cloud credentials in Rancher Cluster Management > Cloud Credentials section of UI.  However, when configuring from the UI, the Kubernetes secret defined within the cattle-global-data namespace generates a unique identifier for each secret, making it challenging to work within multiple environments.  By creating the secret via kubectl with a static name, you can reduce the overhead of having to find and override the cloudCredentialSecretName needed during the helm deployment. 

If creating the secret via the Rancher UI (Cluster Management > Cloud Credentials > Create Nutanix > Enter Prism Central Creds) is preferred, make sure to override the default value when setting helm values for cloudCredentialSecretName prior to deployment.

```bash
## set environment variables
export PRISM_CENTRAL_ENDPOINT="<pc_cluster_ip>"
export PRISM_CENTRAL_PASSWORD="<pc_password>"
export PRISM_CENTRAL_PORT="9440"
export PRISM_CENTRAL_USER="admin"

## create nutanix-pc-creds secret in cattle-global-data namespace
kubectl create secret generic nutanix-pc-creds -n cattle-global-data \
  --from-literal nutanixcredentialConfig-endpoint=${PRISM_CENTRAL_ENDPOINT} \
  --from-literal nutanixcredentialConfig-password=${PRISM_CENTRAL_PASSWORD} \
  --from-literal nutanixcredentialConfig-port=${PRISM_CENTRAL_PORT} \
  --from-literal nutanixcredentialConfig-username=${PRISM_CENTRAL_USER}

## get unique admin user id (user-xyz) and set annotation. This is used by Rancher and will generate errors in logs if not set.
export ADMIN_USER_ID=$(kubectl get users -o jsonpath='{.items[?(@.username=="admin")].metadata.name}')
kubectl annotate secret nutanix-pc-creds -n cattle-global-data field.cattle.io/creatorId=${ADMIN_USER_ID}
```

### Optional - Upload Generic Cloud Image into Prism Central

If you don't already have a working linux machine image with cloud-init enabled you can quickly upload the Generic Cloud Image into Prism Central to more easily follow examples below.

If leveraging other Linux images, make sure to update `vmImage` and `cloud-init` references when setting helm values.

- Navigate to Prism Central > Infrastructure > Compute & Storage > Images
- Click Add Image > Select URL
- Enter URL, For Example: `https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64.img`
- Click Upload File > Accept Remaining Defaults > Click Save

## QuickStart Examples

### Provision RKE2 Cluster with Multiple Node Pools Helm Global Defaults

In this example scenario, the goal would be to demonstrate how you can deploy an RKE2 cluster with multiple nodepools (1 x control-plane, 1 x etcd, 1 x worker) using global configs. This is ideal if you are looking to reduce the number of duplicate configs defined within your helm values by only leveraging global configs and whatever default values are leveraged throughout helm templates (e.g., vCPU, memory, disk, etc.).

Prior to running the command below, make sure to update the `./examples/multi-nodepool-global-values.yaml` file with the information related to your target environment.

```bash
## set environment variables
export KUBERNETES_VERSION=v1.24.17+rke2r1
export INSTANCE_PREFIX=rke2-on-nutanix

## run helm command to provision new rke2 cluster
helm upgrade --install ${INSTANCE_PREFIX} ./charts/rancher-nutanix-cluster-template \
  --namespace fleet-default \
  --values ./examples/multi-nodepool-global-values.yaml \
  --set kubernetesVersion=${KUBERNETES_VERSION} \
  --set cluster.name=${INSTANCE_PREFIX}
```

### Provision K3s Cluster with Multiple Node Pools and Custom Configs

In this example scenario, the goal would be to demonstrate how you can deploy an K3S cluster with multiple nodepools (1 x control-plane, 1 x etcd, 1 x worker) using a combination of global configs and local overrides for handling non-standard use-cases.

This scenario is ideal if you are looking to override the global/default values leveraged throughout helm templates with custom configurations per node pool (e.g., `vCPU`, `memory`, `disk`, `vmCategories`, etc.). Other example use cases could include placing each nodepool in alternate Nutanix AHV clusters, networks, projects, etc.

Prior to running the command below, make sure to update the `./examples/multi-nodepool-overrides-values.yaml` file with the information related to your target environment.

In addition, make sure that Prism Central Category Key `AppTier` should already be updated to include the following category values: `Rancher_Controlplane` `Rancher_Etcd` `Rancher_Worker`.

By leveraging custom categories per nodepool, you can setup dynamic policies throughout Prism Central, such as leveraging security policies / microsegmentation between the individual AppTiers.

```bash
## set environment variables
export KUBERNETES_VERSION=v1.24.17+k3s1
export INSTANCE_PREFIX=k3s-on-nutanix

## run helm command to provision new rke2 cluster
helm upgrade --install ${INSTANCE_PREFIX} ./charts/rancher-nutanix-cluster-template \
  --namespace fleet-default \
  --values ./examples/multi-nodepool-overrides-values.yaml \
  --set kubernetesVersion=${KUBERNETES_VERSION} \
  --set cluster.name=${INSTANCE_PREFIX}
```

## Additional Scenarios

### Scaling Existing RKE2 Cluster Example

```bash
### scale rke2 cluster nodepools by reusing existing helm values, but overriding via helm set command
export INSTANCE_PREFIX=rke2-on-nutanix

helm upgrade --install ${INSTANCE_PREFIX} ./charts/rancher-nutanix-cluster-template \
  --namespace fleet-default \
  --reuse-values \
  --set cluster.name=${INSTANCE_PREFIX} \
  --set nodepools[0].quantity=2 \
  --set nodepools[1].quantity=3 \
  --set nodepools[2].quantity=3
```

### Upgrading RKE2 Kubernetes Cluster Version

```bash
### upgrade rke2 cluster version to v1.25.15 but reuse values. make sure to take snapshot before doing.
export KUBERNETES_VERSION=v1.25.15+rke2r1
export INSTANCE_PREFIX=rke2-on-nutanix

helm upgrade --install ${INSTANCE_PREFIX} ./charts/rancher-nutanix-cluster-template \
  --namespace fleet-default \
  --reuse-values \
  --set kubernetesVersion=${KUBERNETES_VERSION} \
  --set cluster.name=${INSTANCE_PREFIX}
```

### Upgrading RKE2 Node OS Machine Image

```bash
### upgrade underlying node image to v2, simulating upgrade to new version of underlying machine image. make sure to take snapshot before doing.
export INSTANCE_PREFIX=rke2-on-nutanix

helm upgrade --install ${INSTANCE_PREFIX} ./charts/rancher-nutanix-cluster-template \
  --namespace fleet-default \
  --reuse-values \
  --set cluster.name=${INSTANCE_PREFIX} \
  --set global.nodepools.vmImage=ubuntu-22.04-server-cloudimg-amd64-v2.img
```

### Provision RKE2 Cluster with 3 Linux VMs

#### Pre-requisites

- Workstation with `kubectl` and helm cli `available`
- Nutanix AHV IPAM Enabled Subnet
- Static IP Addresses available for Load Balanced API Endpoint (using kube-vip)
- Existing 3 Linux VMs with SSH Access

#### Provision First RKE2 Control Plane Node

```bash
## set environment variables. Each node-ip is needed for backdoor if VIP becomes unavailable for whatever reason
export RKE2_API_VIP=10.38.16.140
export RKE2_NODE_0_IP=10.38.16.91
export RKE2_NODE_1_IP=10.38.16.99
export RKE2_NODE_2_IP=10.38.16.82

export NODE_JOIN_TOKEN=RancherOnNutanixRocks
export INTERFACE=enp0s3
export KUBE_VIP_VERSION=v0.4.2

## pre-create rke2 directories needed for rke2 config file and static pod manifests
mkdir -p /etc/rancher/rke2
mkdir -p /var/lib/rancher/rke2/server/manifests/

## create rke2 config yaml. disabling rke2 snapshot controller helm charts to avoid conflicts with nutanix csi and rke2 v1.25+
cat <<EOF | tee /etc/rancher/rke2/config.yaml
token: ${NODE_JOIN_TOKEN}
tls-san:
- ${HOSTNAME}
- ${RKE2_API_VIP}
- ${RKE2_NODE_0_IP}
- ${RKE2_NODE_1_IP}
- ${RKE2_NODE_2_IP}
write-kubeconfig-mode: 600
etcd-expose-metrics: true
cni:
- calico
disable:
- rke2-ingress-nginx
- rke2-snapshot-validation-webhook
- rke2-snapshot-controller
- rke2-snapshot-controller-crd
EOF

## install latest rke2 binaries, currently 1.26
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh - 

## enable and start rke2-server service
systemctl enable rke2-server.service
systemctl start rke2-server.service

## watch logs on rke2-server
journalctl -u rke2-server -f

## By default RKE2 deploys all the binaries in /var/lib/rancher/rke2/bin path. Containerd variables are leveraged by crictl/ctr
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock

## Optionally, you can append these lines into current user's .bashrc file and reload from source
cat <<EOF | tee -a $HOME/.bashrc
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock
EOF
source ~/.bashrc

## Validate via kubectl
kubectl get nodes -o wide

#### Deploy kube-vip

## Configure kube-vip RBAC 
curl https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/rke2/server/manifests/kube-vip-rbac.yaml

## Since rke2 server solely leverages containerd, and doesn't install docker, we'll be using crictl/ctr cli to generate kube-vip manifest
## fetch the kube-vip image
crictl pull docker.io/plndr/kube-vip:$KUBE_VIP_VERSION

## generate kube-vip daemonset as static pod manifest
alias kube-vip="ctr --namespace k8s.io run --rm --net-host docker.io/plndr/kube-vip:$KUBE_VIP_VERSION vip /kube-vip"
kube-vip manifest daemonset \
  --arp \
  --interface $INTERFACE \
  --address $RKE2_API_VIP \
  --controlplane \
  --leaderElection \
  --taint \
  --services \
  --inCluster | tee /var/lib/rancher/rke2/server/manifests/kube-vip.yaml

## Wait for the kube-vip to complete bootstrapping 
kubectl rollout status daemonset kube-vip-ds -n kube-system --timeout=300s

## Validate that kube-vip ds pods have been deployed
kubectl get ds -n kube-system kube-vip-ds

## ping VIP to ensure that it's up and running
ping $RKE2_API_VIP
```

#### Provision Remaining RKE2 Control Plane Nodes

The following commands should be run on the other 2 control plane nodes. They can be run simultaneously.

```bash
## set SAME environment variables on each node.
export RKE2_API_VIP=10.38.16.140
export RKE2_NODE_0_IP=10.38.16.91
export RKE2_NODE_1_IP=10.38.16.99
export RKE2_NODE_2_IP=10.38.16.82
export NODE_JOIN_TOKEN="RancherOnNutanixRocks"

## pre-create rke2 directories needed for rke2 config file and static pod manifests
mkdir -p /etc/rancher/rke2
mkdir -p /var/lib/rancher/rke2/server/manifests/

## set same environment variables
cat <<EOF | tee /etc/rancher/rke2/config.yaml
server: https://${RKE2_API_VIP}:9345
token: ${NODE_JOIN_TOKEN}
tls-san:
- ${HOSTNAME}
- ${RKE2_API_VIP}
- ${RKE2_NODE_0_IP}
- ${RKE2_NODE_1_IP}
- ${RKE2_NODE_2_IP}
write-kubeconfig-mode: 600
etcd-expose-metrics: true
cni:
- calico
disable:
- rke2-ingress-nginx
- rke2-snapshot-validation-webhook
- rke2-snapshot-controller
- rke2-snapshot-controller-crd
EOF

## install latest rke2 binaries, currently 1.26
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sh - 

## enable and start rke2-server service
systemctl enable rke2-server.service
systemctl start rke2-server.service

## watch logs on rke2-server
journalctl -u rke2-server -f

## By default RKE2 deploys all the binaries in /var/lib/rancher/rke2/bin path. Containerd variables are leveraged by crictl/ctr
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock

## Optionally, you can append these lines into current user's .bashrc file and reload from source
cat <<EOF | tee -a $HOME/.bashrc
export PATH=$PATH:/var/lib/rancher/rke2/bin
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock
EOF
source ~/.bashrc

## Validate via kubectl. After all nodes have been provisioned, validate that all nodes have been joined the domain
$ kubectl get nodes
NAME                 STATUS   ROLES                       AGE     VERSION
rke2-rancher-mcm-0   Ready    control-plane,etcd,master   37m   v1.26.9+rke2r1
rke2-rancher-mcm-1   Ready    control-plane,etcd,master   13m   v1.26.9+rke2r1
rke2-rancher-mcm-2   Ready    control-plane,etcd,master   13m   v1.26.9+rke2r1
```

### Deploy Highly Available Rancher MCM via Helm

```bash
## Deploy Certificate Manager via Helm
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0 \
  --set installCRDs=true \
  --wait

## Deploy Kube-VIP Daemonset to advertise VIPs for service load balancing scenario ONLY.
## IMPORTANT: This is NOT required for Running Rancher HA on RKE2 Cluster scenario and ONLY required for Running Rancher HA on NKE cluster scenario.
## as kube-vip daemonset has already been deployed and enabled for both control-plane and service load balancer use case.

helm repo add kube-vip https://kube-vip.github.io/helm-charts
helm repo update

helm upgrade --install kube-vip kube-vip/kube-vip \
  --namespace kube-system \
  --set image.repository="ghcr.io/kube-vip/kube-vip" \
  --set image.tag="v0.4.1" \
  --wait

helm upgrade --install kube-vip-cloud-provider \
  kube-vip/kube-vip-cloud-provider \
  --namespace kube-system \
  --set image.repository="ghcr.io/kube-vip/kube-vip-cloud-provider" \
  --set image.tag="v0.0.7" \
  --wait

## Create Kube-VIP ConfigMap for service loadbalancer ip address pool range to be used explicitly by ingress-nginx namespace. 
## This could be set to a range of ip addresses (i.e., 10.38.16.74-10.38.16.80) or just to network CIDR (i.e., 10.38.16.74/32).

export SERVICE_LB_IP_POOL="10.38.16.74/32"

kubectl create configmap -n kube-system kubevip \
  --from-literal cidr-ingress-nginx=${SERVICE_LB_IP_POOL}

## Deploy Ingress Nginx Controller via Helm with 3 replicas, and set to default ingress class.
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx \
  ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set rbac.create=true \
  --set controller.replicaCount=3 \
  --set ingressClassResource.default=true \
  --wait

## Validate that Ingress Nginx Controller has valid Service Load Balancer IP from Kube-VIP IP Range
export NGINX_LOADBALANCER_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath="{.status.loadBalancer.ingress[].ip}")
echo "Rancher Ingress NIP.IO Domain: rancher.${NGINX_LOADBALANCER_IP}.nip.io"

## Deploy Rancher via Helm

## set environment variables
export NGINX_LOADBALANCER_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath="{.status.loadBalancer.ingress[].ip}")
export RANCHER_DNS_DOMAIN="rancher.${NGINX_LOADBALANCER_IP}.nip.io"

helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=${RANCHER_DNS_DOMAIN} \
  --set replicas=3 \
  --set-string ingress.extraAnnotations."kubernetes\.io\/ingress\.class"=nginx \
  --wait
```

### Deploy Nutanix CSI Driver via Helm

```bash
## Create ntnx-system namespace
kubectl create ns ntnx-system

## Configure Nutanix Helm repo
helm repo add nutanix https://nutanix.github.io/helm/
helm repo update nutanix

## Install nutanix-csi-snapshot helm chart
helm upgrade --install -n ntnx-system \
  nutanix-csi-snapshot nutanix/nutanix-csi-snapshot \
  --set tls.renew=true \
  --wait

## Install nutanix-csi-storage helm chart and configure Nutanix Volumes and Nutanix Files Storage Classes

## set environment variables 
export FILE_SERVER_NAME="BootcampFS"
export PRISM_ELEMENT_VIP="10.38.20.7"
export PRISM_ELEMENT_USER="admin"
export PRISM_ELEMENT_PASS="password"
export STORAGE_CONTAINER_NAME="Default"

helm upgrade --install -n ntnx-system \
  nutanix-csi nutanix/nutanix-csi-storage \
  --set fileServerName=${FILE_SERVER_NAME} \
  --set prismEndPoint=${PRISM_ELEMENT_VIP} \
  --set username=${PRISM_ELEMENT_USER} \
  --set password=${PRISM_ELEMENT_PASS} \
  --set storageContainer=${STORAGE_CONTAINER_NAME} \
  --set volumeClass=true \
  --set volumeClassName=nutanix-volume \
  --set dynamicFileClass=true \
  --set dynamicFileClassName=nutanix-dynamicfile \
  --set defaultStorageClass=nutanix-volume \
  --set secretName=ntnx-secret \
  --set createSecret=true \
  --set kubeletDir=/var/lib/kubelet \
  --wait
```

## Troubleshooting

### Debugging Helm Deployments

If Cluster deployments are failing to deploy, but helm chart completed successfully, something may be wrong with underlying helm templates.  

Easiest way to debug is to add `--debug --dry-run` parameters to end of command. For Example:

```bash
## set environment variables
export KUBERNETES_VERSION=v1.23.17+rke2r1
export INSTANCE_PREFIX=rke2-on-ntnx-debug

## run helm command to provision new rke2 cluster
helm upgrade --install ${INSTANCE_PREFIX} ./charts/rancher-nutanix-cluster-template \
  --namespace fleet-default \
  --values ./examples/multi-nodepool-global-values.yaml \
  --set kubernetesVersion=${KUBERNETES_VERSION} \
  --set cluster.name=${INSTANCE_PREFIX} \
  --debug --dry-run
```

Alternatively, you can review the logs within the rancher pods to see if something is failing elsewhere

`kubectl logs -l app=rancher -n cattle-system`

## Updating Cluster Template Helm Chart Package

This step is needed to update the `cluster-template-<version>.tgz` and `index.yaml` file that is stored locally within this Github repository.
The file name and version are generated based on the `charts/rancher-nutanix-cluster-template/Chart.yaml`` file.

It should be run prior to committing changes to `main` branch so that the Rancher Repository synchronization can pick up changes
and respective custom cluster template UI udpates.

This file should be wrapped with CICD pipelines (i.e., github actions), properly version controlled using semantic versioning and
hosted within proper remote OCI compliant helm repository if being leveraged for production purposes.

```bash
helm package charts/rancher-nutanix-cluster-template -d .
helm repo index .
```
