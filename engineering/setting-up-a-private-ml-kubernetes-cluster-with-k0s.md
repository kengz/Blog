---
description: 2021/07/05
---

# Setting up a private ML Kubernetes cluster with K0s

In this guide, we will use [k0s](https://k0sproject.io) to set up a private multi-node GPU Kubernetes cluster with private Docker registry for ML applications.

## Previous guide

It's been over a year since I wrote about setting up a private Kubernetes \(k8s\) cluster with a private docker registry and GPU support:

{% page-ref page="setting-up-a-private-kubernetes-cluster.md" %}

That used Kubeadm to setup Kubernetes, which is quite an involved process. I wanted a faster way to do it, and luckily there's a new tool for that - [k0s](https://k0sproject.io). It fully sets up Kubernetes from just a few commands.

## Setup GPU nodes

We're still setting up our own GPU nodes \(machines\). See the following guide for instruction.

{% page-ref page="ubuntu-gpu-server-setup.md" %}

## Setup Kubernetes with k0s

This references the [official k0s guide](https://docs.k0sproject.io/latest/install/) and the [Mirantis guide](https://www.mirantis.com/blog/how-to-set-up-k0s-kubernetes-a-quick-and-dirty-guide/), with a few adjustments of my own. Run the following on a node designated as controller, which also doubles as a worker.

1. Download k0s:

```bash
curl -sSLf https://get.k0s.sh | sudo sh
```

2. On the controller node, install a k0s controller that also acts as a worker:

```bash
sudo k0s install controller --enable-worker
```

3. Start the installed k0scontroller service, and enable it \(so it auto-runs on node restart\). This will take a few minutes to run, so check the status in the mean time.

```bash
sudo systemctl start k0scontroller
sudo systemctl enable k0scontroller
sudo systemctl status k0scontroller
```

4. When it's ready, check the Kubernetes cluster. You should see the controller node with status "Ready":

```bash
sudo k0s kubectl get nodes
```

5. Next, export the kube config for Kubectl \([installation here](https://kubernetes.io/docs/tasks/tools/)\) to run without needing k0s.

```text
mkdir ~/.kube
sudo cp /var/lib/k0s/pki/admin.conf ~/.kube/config
```

To configure access to the cluster from another machine \(e.g. your laptop, or a different node\), simply copy the kube config above and replace the `server:` [`https://localhost:6443`](https://localhost:6443) part with the local IP of the controller node.

{% hint style="info" %}
Optional: use [Lens](https://k8slens.dev) to monitor your K8s cluster.
{% endhint %}

6. Now check that kubectl can run without k0s:

```
kubectl get nodes
```

### Setup worker nodes

Next, we set up additional worker nodes. We need to generate a token to join the Kubernetes server.

1. On the controller node, generate a worker token, which is just a base64-encoded kube config:

```text
k0s token create --role=worker
```

2. On a new worker node, install k0s, and run the join-command in a screen:

```bash
screen -S k0s
sudo k0s worker "THE_GENERATED_TOKEN"
```

3. Again, wait for a few minutes for the worker node to be set up. If you wish, set up the kube config on the worker node too as mentioned in the previous section. Check the cluster again:

```bash
kubectl get nodes
```

### Cluster setup for private Docker registry and GPU support

1 . Install a [private Docker registry from Helm](https://github.com/helm/charts/tree/master/stable/docker-registry). This uses `10.96.10.96` as the clusterIP for pushing and pulling images:

```bash
helm repo add stable https://charts.helm.sh/stable

k create namespace docker
helm install private stable/docker-registry --set persistence.enabled=true,persistence.storageClass=local-path,service.clusterIP=10.96.10.96 -n docker
```

{% hint style="info" %}
When building, pushing and pulling Docker images on any node in the cluster, use the clusterIP for the image name, i.e. image shall be tagged as `10.96.10.96:5000/ORG_NAME/IMAGE_NAME`
{% endhint %}

2. Install the NVIDIA device plugin on your cluster:

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta4/nvidia-device-plugin.yml
```

Now, we're nearly there. Before we can pull images and allocate GPU pods, we need to configure containerd to support the above.

## Configuring containerd

K0s uses [containerd](https://github.com/containerd/containerd) instead of Docker for its container runtime. In the [previous guide](setting-up-a-private-kubernetes-cluster.md#kubernetes-nvidia-gpu-device-plugin), we had to configure `/etc/docker/daemon.json` to:

* [add the IP for the private docker registry](setting-up-a-private-kubernetes-cluster.md#private-docker-registry-ip)
* [specify the default runtime to nvidia-container-runtime](setting-up-a-private-kubernetes-cluster.md#kubernetes-nvidia-gpu-device-plugin)

{% hint style="info" %}
Repeat the following for each node in the cluster.
{% endhint %}

Now, we need to do the equivalent for containerd, in its config file `/etc/k0s/containerd.toml`. First, get it ready:

1. Create a containerd config file, as per the [k0s guide](https://docs.k0sproject.io/v1.21.2+k0s.1/runtime/):

```bash
containerd config default > /etc/k0s/containerd.toml
```

2. Update the header of `/etc/k0s/containerd.toml` to match the k0s path:

```bash
version = 2
root = "/var/lib/k0s/containerd"
state = "/var/lib/k0s/run/containerd"
...

[grpc]
  address = "/var/lib/k0s/run/containerd.sock"
```

### Enable private Docker registry on k0s

#### Private Docker registry IP

Note that k8s pod image pull cannot use FQDN for in-cluster registry \(see [link](https://discuss.kubernetes.io/t/how-does-kubelet-dns-resolution-before-pod-creation-work/9489)\). Use a reserved clusterIP instead.

> Reserved clusterIP for registry: `10.96.10.96`

In the `/etc/k0s/containerd.toml`, we need to:

1. add a mirror for our clusterIP
2. allow for HTTP in pulling image from the clusterIP

Here's an example snippet added alongside the existing docker mirror:

```text
...
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."10.96.10.96:5000"]
          endpoint = ["http://10.96.10.96:5000"]
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."10.96.10.96:5000".tls]
          insecure_skip_verify = true
...
```

{% hint style="warning" %}
Different version of containerd has a different config key format in the toml file. So adopt accordingly to the toml you generated. For example, in [an older version](https://github.com/containerd/cri/blob/master/docs/registry.md) you have to replace `"io.containerd.grpc.v1.cri"` with `cri`.
{% endhint %}

```text
# older version
...
    [plugins.cri.registry]
      [plugins.cri.registry.mirrors]
        [plugins.cri.registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
        [plugins.cri.registry.mirrors."10.96.10.96:5000"]
          endpoint = ["http://10.96.10.96:5000"]
      [plugins.cri.registry.configs]
        [plugins.cri.registry.configs."10.96.10.96:5000".tls]
          insecure_skip_verify = true
...
```

### Enable GPU support on k0s

We have to configure the default runtime to `nvidia-container-runtime` to allow GPU container to be allocated in the Kubernetes cluster.

{% hint style="info" %}
Repeat the following for each node in the cluster.
{% endhint %}

#### Kubernetes NVIDIA GPU device plugin

1. In case needed, update your Nvidia driver first:

```text
sudo ubuntu-drivers autoinstall
```

2. As before, follow the [official NVIDIA GPU device plugin](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#official-nvidia-gpu-device-plugin) until the step to configure runtime

3. As explained in [this comment](https://github.com/NVIDIA/k8s-device-plugin/issues/168#issuecomment-625966602), k8s still needs `nvidia-container-runtime`; install it:

```bash
# install the old nvidia-container-runtime for k8s
curl -s -L https://nvidia.github.io/nvidia-container-runtime/gpgkey | \
  sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-runtime.list
sudo apt-get update
sudo apt-get install -y nvidia-container-runtime
```

4. In `/etc/k0s/containerd.toml` find and replace `plugins.linux` runtime value `runc` with `nvidia-container-runtime`, again per the [k0s guide](https://docs.k0sproject.io/v1.21.2+k0s.1/runtime/#using-nvidia-container-runtime):

```bash
...
[plugins."io.containerd.runtime.v1.linux"]
    shim = "containerd-shim"
    runtime = "nvidia-container-runtime"
    ...
```

{% hint style="warning" %}
Different version of containerd has a different config key format in the toml file. So adopt accordingly to the toml you generated. For example, an older variation is:
{% endhint %}

```bash
# older version
...
  [plugins.linux]
    shim = "containerd-shim"
    runtime = "nvidia-container-runtime"
    ...
```

5. Restart k0s, and describe Kubernetes nodes.

```bash
sudo systemctl restart k0scontroller
kubectl describe nodes
```

In the kubectl output, you should see `nvidia.com/gpu: 1` in the Allocatable section now.

```bash
...
Allocatable:
  cpu:                8
  ephemeral-storage:  884610528225
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             16245348Ki
  nvidia.com/gpu:     1
  pods:               110
```

### Restart

Finally, restart k0s for these changes to take effect:

```bash
sudo systemctl restart k0scontroller
```

{% hint style="success" %}
You now have a multi-node GPU Kubernetes cluster with a private Docker registry ready for use.
{% endhint %}

