---
description: 2021/07/18
---

# Distributed Training with TorchElastic on Kubernetes

Distributed training is useful for speeding up training of a model with large dataset by utilizing multiple nodes \(computers\). In the past few years, the technical difficulty of doing distributed training has lowered drastically that it is no longer reserved just for engineers working at a large AI institution.

At a quick glance, distributed training involves multiple nodes running a training. The nodes are assigned ranks from 0 to N, with 0 being the master. Typically:

* all nodes have a copy of the model
* each node will sample a partition of the full dataset \(e.g. by sampling data with `index % num_nodes == node_rank`\) to train its model copy
* the gradients are accumulated then synced to master for model update

Let's look at one way of doing this simply using 3 components:

* [PyTorch Lightning](https://github.com/PyTorchLightning/pytorch-lightning): handles all the engineering code such as device placement, so that there's little to no changes to the model code.
* [TorchElastic](https://pytorch.org/docs/stable/distributed.elastic.html): handles the distributed communication/interface in a fault-tolerant manner.
* [Kubernetes](https://kubernetes.io): compute cluster with multiple nodes \([see this guide](../engineering/setting-up-a-private-ml-kubernetes-cluster-with-k0s.md) to set up your own Kubernetes cluster\).

## PyTorch Lightning

If you're using PyTorch and manually doing a lot of "engineering chores" that are not directly related to your model/loss function, you should use [PyTorch Lightning](https://pytorchlightning.ai). Primarily, it takes away the need for boilerplate code by handling the training/evaluation loop, device placement, logging, checkpointing, etc. You only need to focus on the model architecture, loss function, and the dataset.

The device placement handling is very useful for running your model on different hardwares \(with or without GPUs\). It also supports distributed training with TorchElastic.

Suppose you already have a model written as a LightningModule that is not distributed yet. You only need to make a few modifications:

* [set the Trainer to use Distributed Data Parallel \(ddp\)](https://pytorch-lightning.readthedocs.io/en/latest/advanced/multi_gpu.html#distributed-data-parallel) by specifying `accelerator='ddp'`. This takes care of the distributed training logic, including the appropriate data sampling for each node by [automatically switching to DistributedSampler](https://pytorch-lightning.readthedocs.io/en/latest/advanced/multi_gpu.html#remove-samplers).
* in your LightningModule, assign a `self.is_dist` attributed based on whether `accelerator='ddp'` is specified.
* [synchronize your logging across multiple nodes](https://pytorch-lightning.readthedocs.io/en/latest/advanced/multi_gpu.html#synchronize-validation-and-test-logging) by specifying `sync_dist=self.is_dist`

  to prevent conflicts, e.g.`self.log('train/loss', loss, sync_dist=self.is_dist)`

* likewise, ensure that any processes that should be run once \(such as uploading artifacts/model file\) is called only on the master node by checking `os.environ('RANK', '0') == '0'`. The RANK environment variable will be set by TorchElastic when it runs.
* for more comprehensive tips on updating your LightningModule to support distributed training, check out [this page](https://pytorch-lightning.readthedocs.io/en/latest/advanced/multi_gpu.html).

Next, test your changes by [simulating distributed training on a single node](https://pytorch-lightning.readthedocs.io/en/latest/common/trainer.html#num-processes):

```python
# use num_processes to simulate 2 nodes on one machine
trainer = Trainer(accelerator="ddp", num_processes=2)
```

## TorchElastic

If you're using `PyTorch >= 1.9.0`, [TorchElastic is already included](https://pytorch.org/docs/stable/distributed.elastic.html) as `torch.distributed`. It will be used to [launch the distributed training process](https://pytorch.org/docs/stable/elastic/quickstart.html) on every node \(container\), which will be composed by our Kubernetes manifest file next.

TorchElastic works by using an [etcd](https://etcd.io) server for communication, so:

* in your Conda environment, install `python-etcd`.
* in your Dockerfile, install etcd to system. Use [this script](https://github.com/pytorch/elastic/blob/master/examples/bin/install_etcd).

PyTorchLightning works nicely with TorchElastic, so that's all.

## Kubernetes

No matter if you use a vendor or [set up your own](../engineering/setting-up-a-private-ml-kubernetes-cluster-with-k0s.md), a Kubernetes cluster with multiple nodes is ready to run distributed training. Here's the overview:

* spawn a etcd server on Kubernetes
* spawn multiple pods, each with a container that maximizes the amount of compute resources available on a node
* run the torchelastic command on each container with the etcd parameters

The logic for spawning pods - managing them elastically, inserting the right arguments such as RANK/WORLD\_SIZE, can be tricky. Luckily, TorchElastic has a ElasticJob Controller that can be installed on your cluster as Custom Resource Definition \(CRD\) to manage these pods elastically.

[Install the TorchElastic CRDs](https://github.com/pytorch/elastic/tree/master/kubernetes#install-elasticjob-controller-and-crd) - you will need cluster admin role to do so. By default this will create a elastic-job namespace to run the training in, but [you can customize it by modifying the config](https://github.com/pytorch/elastic/blob/master/kubernetes/config/default/kustomization.yaml#L2).

### Dockerfile

In your Dockerfile, as mentioned earlier, [install etcd](https://github.com/pytorch/elastic/blob/master/examples/bin/install_etcd).

Additionally, configure it to run torch distributed as entrypoint. The CRD will automatically append the relevant rank commands when creating pods.

```python
...

ENTRYPOINT ["python", "-m", "torch.distributed.run"]
CMD ["--help"]
```

### Manifest Files

We only need 2 Kubernetes manifest files - for etcd, and elasticjob.

Note that each etcd server is only reserved for running one distributed training session; suppose multiple engineers want to run different models with distributed training on the same cluster, they each need to spawn their own instance with a new pair of etcd server and elasticjob without conflict.

Here are the example manifest files modified from [the original TorchElastic examples](https://github.com/pytorch/elastic/tree/master/kubernetes/config/samples) to run simultaneously without conflict. Replace the example "MY-MODEL" with a different name for each instance.

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: elastic-job
  name: MY-MODEL-etcd-service
spec:
  ports:
    - name: etcd-client-port
      port: 2379
      protocol: TCP
      targetPort: 2379
  selector:
    app: MY-MODEL-etcd

---
apiVersion: v1
kind: Pod
metadata:
  namespace: elastic-job
  name: MY-MODEL-etcd
  labels:
    app: MY-MODEL-etcd
spec:
  containers:
    - name: etcd
      image: quay.io/coreos/etcd:latest
      command:
        - /usr/local/bin/etcd
        - --data-dir
        - /var/lib/etcd
        - --enable-v2
        - --listen-client-urls
        - http://0.0.0.0:2379
        - --advertise-client-urls
        - http://0.0.0.0:2379
        - --initial-cluster-state
        - new
      ports:
        - containerPort: 2379
          name: client
          protocol: TCP
        - containerPort: 2380
          name: server
          protocol: TCP
  restartPolicy: Always
```

```yaml
apiVersion: elastic.pytorch.org/v1alpha1
kind: ElasticJob
metadata:
  namespace: elastic-job
  name: MY-MODEL
spec:
  rdzvEndpoint: MY-MODEL-etcd-service:2379
  minReplicas: 1
  maxReplicas: 2
  replicaSpecs:
    Worker:
      replicas: 2
      restartPolicy: ExitCode
      template:
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: elasticjob-worker
            image: YOUR_DOCKER_IMAGE:0.0.1
            imagePullPolicy: Always
            args:
              - "--nproc_per_node=1"
              - "my_model/train.py"
              - "trainer.accelerator=ddp"
              # if you can pass it to argparse/Hydra config
            resources:
              limits:
                cpu: 4
                memory: 12Gi
                nvidia.com/gpu: 1
            volumeMounts:
            - name: dshm # increase shared memory for dataloader
              mountPath: /dev/shm
          volumes:
          - name: dshm
            emptyDir:
              medium: Memory
```

Then apply these manifest files and you'll have distributed training running, with the ElasticJob Controller managing pods elastically to ensure uptime.

You can also parametrize these manifest files with [Helm](https://helm.sh) to create multiple distributed training instances.

## Tying it all together

That's all you need to run distributed training. To summarize:

* use PyTorchLightning, specify `accelerator='ddp'` and some logging fixes
* install etcd on Conda environment as well as Docker image
* build your Docker image, specify the `ENDPOINT` with torch.distributed.run
* install the TorchElastic CRDs on your Kubernetes cluster
* create and apply the etcd and elasticjob manifest files on Kubernetes
* you have distributed training running!





