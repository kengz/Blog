---
description: 2020/06/20
---

# Setting up a private Kubernetes cluster

Recently I set up a private Kubernetes \(k8s\) cluster on my home GPU servers. Since the cluster is private, it is not exposed to the internet.

This means that the docker-registry for hosting the container images is also private within the cluster. The private registry is not so straightforward to set up, but we will discuss it below.

First, we need one or more machines for the cluster. One will be the main \(master\) of the cluster, and the rest will be joining as nodes. If you have only one machine, no worries - the machine hosting the master can also run as a node.

I'm running Ubuntu Server 20.04 - this is much easier to set up than the Ubuntu Desktop with GUI. In fact, the whole process takes less than an hour, and even the notorious Nvidia driver installation is a breeze since we don't need to deal with GUI and X Server. See the linked post for guide.

{% page-ref page="ubuntu-gpu-server-setup.md" %}

## Setup nodes

Before installing kubernetes, setup the node machines as follows.

#### Private Docker registry IP

Note that k8s pod image pull cannot use FQDN for in-cluster registry \(see [link](https://discuss.kubernetes.io/t/how-does-kubelet-dns-resolution-before-pod-creation-work/9489)\). Use a reserved clusterIP instead.

> Reserved clusterIP for registry: `10.96.10.96`

Our private registry will run on the main. When setting up the k8s main node:

* append the following to `/etc/docker/daemon.json`:

  ```javascript
  {
      ...,
      "insecure-registries" : [
          "localhost:5000",
          "127.0.0.1:5000",
          "0.0.0.0:5000",
          "10.96.10.96:5000"
      ]
  }
  ```

* restart docker: `sudo systemctl restart docker`

#### Kubernetes NVIDIA GPU device plugin

* follow the [official NVIDIA GPU device plugin](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#official-nvidia-gpu-device-plugin) until the step to configure runtime
* as explained in [this comment](https://github.com/NVIDIA/k8s-device-plugin/issues/168#issuecomment-625966602), k8s still needs `nvidia-container-runtime`; install it:

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

* add the following /etc/docker/daemon.json as required by k8s

  ```javascript
  {
      "default-runtime": "nvidia",
      "runtimes": {
          "nvidia": {
              "path": "/usr/bin/nvidia-container-runtime",
              "runtimeArgs": []
          }
      }
  }
  ```

* restart docker and test:

  ```bash
  sudo systemctl restart docker
  # test that docker can run with GPU without the --gpus flag
  docker run nvidia/cuda:10.2-runtime-ubuntu18.04 nvidia-smi
  ```

## Install Kubernetes

Installs a self-hosted private Kubernetes.

> NOTE: I use the zsh kubectl shortcuts below, e.g. `k, kaf, keti`

* [install Kubernetes with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/). Then install a [local-path volume provisioner](https://github.com/rancher/local-path-provisioner#deployment), and the StorageClass shall be `local-path`.

  ```bash
  kubeadm init

  # logout from root and follow the kube config steps, then

  kubectl taint nodes --all node-role.kubernetes.io/master-
  # install pod network
  kaf "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
  # install local-path volume provisioner
  kaf https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
  ```

* the `kubeadm init` step will output a command for nodes to join the cluster. You can wait until the registry is set up below to join to ensure the registry in on your main.
* Kubernetes should auto-restart when host machine restarts. If not, check the status and possibly turn off the swap.
* ```bash
  # check the status
  systemctl status kubelet

  # you may get error like: failed to run Kubelet: Running with swap on is not supported
  sudo swapoff -a
  ```
* install a [private Docker registry from Helm Hub](https://github.com/helm/charts/tree/master/stable/docker-registry).

  ```bash
  helm repo add twuni https://helm.twun.io

  k create namespace docker
  helm install private twuni/docker-registry --set persistence.enabled=true,persistence.storageClass=local-path,service.clusterIP=10.96.10.96 -n docker
  ```

* install the NVIDIA device plugin on your cluster:

  ```bash
  kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta4/nvidia-device-plugin.yml
  ```

* install [Octant](https://github.com/vmware-tanzu/octant) on your local \(laptop\) for dashboard
* Generate service account kubeconfig for access: on the kubernetes master host machine, create a service account kubeconfig for your local machine, then copy the generated config file `sa-conf` to local, and update the local `~/.kube/config`:

  ```bash
  # on master node
  sudo bash bin/make_kube_config.sh sa default
  # copy this content for ~/.kube/config below
  cat /tmp/kube/sa-conf

  # on user local
  # backup your old config
  cp ~/.kube/config ~/.kube/config_backup
  # paste the content above into ~/.kube/config
  nano ~/.kube/config
  # check nodes
  k get nodes
  ```

### Install Prometheus and Grafana

```bash
k create namespace monitoring
helm install prometheus stable/prometheus-operator -n monitoring

# grafana
kpf svc/prometheus-grafana 3000:80 -n monitoring
# default creds: admin:prom-operator
# the choose some predefined k8s cluster dashboards
```

## How-tos

### Push image to registry

To push images from local to the registry, you can:

* on a k8s node: tag image with the clusterIP `10.96.10.96:5000` and push directly
* outside of node: you need to port-forward the service directly and push to `0.0.0.0:5000`

  ```bash
  # registry for direct docker push
  kpf service/private-docker-registry 5000 -n docker
  ```

* build your image with the either clusterIP or localhost tag and push:

  ```bash
  docker build -t 0.0.0.0:5000/foo/bar:0.0.1 .
  docker push 0.0.0.0:5000/foo/bar:0.0.1
  ```

### Debug DNS issues

* useful [DNS debugging guide](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

```bash
kaf https://raw.githubusercontent.com/kubernetes/kubernetes/master/hack/testdata/recursive/pod/pod/busybox.yaml
keti busybox -- nslookup private-docker-registry.docker
```

