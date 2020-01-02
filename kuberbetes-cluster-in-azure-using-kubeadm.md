# Title

## Overview

By the end of this guide, you will have a Kubernetes cluster on Azure with one master and one worker node. You will know how to add more worker nodes to the cluster.

## Prerequisites

Before you begin, you'll need to have the following:

* an active [Azure account](https://azure.microsoft.com/en-us/free/)
* a MacOS or Linux computer.
* the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) installed on your machine.
* an [SSH key pair](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/mac-create-ssh-keys).

## Procedure

You will start from creating a base [image of a virtual machine](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/capture-image) that will contain [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/), [containerd](https://github.com/containerd/containerd), kubelet and kubectl.

The image will be used to create 2 virtual machines (one for Kubernetes master node and one for Kubernetes worker node).

After that, you will use kubeadm to [create a single control-plane Kubernetes cluster](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).

### Creating a virtual machine

You need to log in to Azure and create a new resource group that will be used through the rest of this guide.

```bash
az login
az group create --name KubernetesTestCluster --location eastus
```

The next command creates a [Standard_B2s](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes-general) virtual machine using the default UbuntuLTS image. You make check current pricing for this and other types of virtual machines at https://azureprice.net/

Make sure you have created your SSH key pair (see Prerequisites) otherwise this command will fail to find `~/.ssh/id_rsa.pub`.

```bash
az vm create \
  --resource-group KubernetesTestCluster \
  --name base-vm \
  --image UbuntuLTS \
  --size Standard_B2s \
  --admin-username azuser \
  --tags name=base-vm \
  --ssh-key-value ~/.ssh/id_rsa.pub
```

The output of `az vm create` contains `publicIpAddress`.
Use it to ssh to your new virtual machine and run `sudo` to become `root`.

```bash
ssh azuser@<publicIpAddress>
sudo su -
```

### Installing containerd to base-vm

These instructions are based on Kubernetes documentation. If you have any questions, please read [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).

In this guide, you will use `containerd` as a Kubernetes container runtime.
However there are several container runtimes listed in [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/), and you can use any of them if you want.

```bash
cat > /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay br_netfilter

# Setup required sysctl params, these persist across reboots.
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system

apt-get update && apt-get install -y apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"

apt-get update && apt-get install -y containerd.io

mkdir -p /etc/containerd && containerd config default > /etc/containerd/config.toml

# since UbuntuLTS uses systemd as the init system
sed -i 's/systemd_cgroup = false/systemd_cgroup = true/' /etc/containerd/config.toml
```

### Installing kubeadm to base-vm

These instructions are based on Kubernetes documentation. If you have any questions, please read [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update && apt-get install -y kubelet kubeadm kubectl && apt-mark hold kubelet kubeadm kubectl

# since containerd is configured to use the systemd cgroup driver
echo 'KUBELET_EXTRA_ARGS=--cgroup-driver=systemd' > /etc/default/kubelet
```

Make sure that swap is disabled in `/etc/fstab`.

### Creating a base VM image from base-vm

The following set of commands creates a custom image of the virtual machine you have just configured. Please read [Tutorial: Create a custom image of an Azure VM with the Azure CLI](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/tutorial-custom-images) if you want to know what each of the commands does.

Deprovision the virtual machine, exit from the `root` environment and close the SSH session.
```bash
waagent -deprovision+user
exit
exit
```

Execute the following commands in your local environment to create `base-vm-image`.

```bash
az vm deallocate \
   --resource-group KubernetesTestCluster \
   --name base-vm

az vm generalize \
   --resource-group KubernetesTestCluster \
   --name base-vm

az image create \
   --resource-group KubernetesTestCluster \
   --name base-vm-image --source base-vm
```

### Deleting unused resources

Now when you have the image, you no longer need the virtual machine you used to create it. You should delete the virtual machine to avoid being charged for resources you don't need.

Unfortunately at the moment of writing, deleting a virtual machine using CLI in Azure does not delete all dependent resources that were automatically created for the machine (for instance, disks and network interfaces).
* https://github.com/Azure/azure-cli/issues/4897
* https://github.com/Azure/azure-cli/issues/8532

Lucky for you, when you ran `az vm create` to create this virtual machine you applied a tag to every resource it created via `--tags name=base-vm`. Now you can use `az resource list --tag` to find all dependent resources that need to be deleted.

```bash
az vm delete \
   --resource-group KubernetesTestCluster \
   --name base-vm

az resource delete --ids $(az resource list --tag name=base-vm --query "[].id" -otsv)
```

### Initializing your control-plane node

You need to create a new virtual machine for your control-plane node using the prepared `base-vm-image`.

```bash
az vm create \
   --resource-group KubernetesTestCluster \
   --name master1 \
   --image base-vm-image \
   --size Standard_B2s \
   --admin-username azuser \
   --ssh-key-value ~/.ssh/id_rsa.pub
```

Use `publicIpAddress` from the output of the command above to ssh to your new virtual machine and run `sudo` to become `root`.

```bash
ssh azuser@<publicIpAddress>
sudo su -
```

Configure bash completion to make your life easier:

```bash
mkdir -p $HOME/.kube
kubectl completion bash > ~/.kube/completion.bash.inc

printf "
# Kubectl shell completion
source '$HOME/.kube/completion.bash.inc'
" >> $HOME/.bash_profile

source $HOME/.bash_profile
```

To initialize the control-plane node run:

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16

[init] Using Kubernetes version: v1.17.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <control-plane-host>:<control-plane-port> --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

If you have any questions about `kubeadm init`, please read [Creating a single control-plane cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).

Configure `kubectl` and run `kubectl cluster-info` to verify that Kubernetes master is running.

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
kubectl cluster-info
```

You should see:

```text
Kubernetes master is running at https://10.0.0.4:6443
KubeDNS is running at https://10.0.0.4:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Install pod network addon:

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### Adding a worker node to the cluster

You need to create a new virtual machine for a worker node using the prepared `base-vm-image`.

Open a new terminal windows and run:

```bash
az vm create \
   --resource-group KubernetesTestCluster \
   --name worker1 \
   --image base-vm-image \
   --size Standard_B2s \
   --admin-username azuser \
   --ssh-key-value ~/.ssh/id_rsa.pub
```

Use `publicIpAddress` from the output of the command above to ssh to your new virtual machine.

```bash
ssh azuser@<publicIpAddress>
```

Run the `kubeadm join` command that you saw in the output of `kubeadm init` above.

```bash
sudo kubeadm join <control-plane-host>:<control-plane-port> --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

> By default, tokens expire after 24 hours. If you are joining a node to the cluster after the current token has expired, you can create a new token by running `kubeadm token create` on the master node.

### Smoke testing

To verify that all nodes are `Ready` and all pods are `Running`, return to the terminal windows with the SSH session to the control-plane node (master1) and run:

```bash
kubectl get nodes -o wide
kubectl get pods --all-namespaces -o wide
```

The output should be similar to this:

```bash
$ kubectl get nodes -o wide
NAME      STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
master1   Ready    master   23m     v1.17.0   10.0.0.4      <none>        Ubuntu 18.04.3 LTS   5.0.0-1027-azure   containerd://1.2.10
worker1   Ready    <none>   2m20s   v1.17.0   10.0.0.5      <none>        Ubuntu 18.04.3 LTS   5.0.0-1027-azure   containerd://1.2.10

$ kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE    IP           NODE      NOMINATED NODE   READINESS GATES
kube-system   coredns-6955765f44-dj8kk          1/1     Running   0          23m    10.244.0.4   master1   <none>           <none>
kube-system   coredns-6955765f44-px77m          1/1     Running   0          23m    10.244.0.3   master1   <none>           <none>
kube-system   etcd-master1                      1/1     Running   0          23m    10.0.0.4     master1   <none>           <none>
kube-system   kube-apiserver-master1            1/1     Running   0          23m    10.0.0.4     master1   <none>           <none>
kube-system   kube-controller-manager-master1   1/1     Running   0          23m    10.0.0.4     master1   <none>           <none>
kube-system   kube-flannel-ds-amd64-mw84v       1/1     Running   0          14m    10.0.0.4     master1   <none>           <none>
kube-system   kube-flannel-ds-amd64-qrfn6       1/1     Running   0          3m3s   10.0.0.5     worker1   <none>           <none>
kube-system   kube-proxy-j7qns                  1/1     Running   0          3m3s   10.0.0.5     worker1   <none>           <none>
kube-system   kube-proxy-kdlgt                  1/1     Running   0          23m    10.0.0.4     master1   <none>           <none>
kube-system   kube-scheduler-master1            1/1     Running   0          23m    10.0.0.4     master1   <none>           <none>
```

## Cleaning up

When you no longer need your Kubernetes cluster, you should delete it to avoid being charged for the resource you don't use.

The fastest way to delete all created resources is to delete the resource group you created at the very beginning of this guide.

```bash
az group delete --name KubernetesTestCluster
```

That's it. Happy hacking!
