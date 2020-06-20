---
description: 2020/06/20
---

# Running WandB on Kubernetes

[Weights & Biases \(WandB\)](https://www.wandb.com) is currently my favorite ML tool because it does so many good things I wanted in my workflow. Among these are:

* experiment tracking
* useful builtin visualizations/dashboard
* great one-liner integration \(with PyTorch Lightning\)
* hyperparameter optimization \(WandB sweep\)

Since I'm running everything on Kubernetes \(k8s\) now, WandB **sweep** jobs fit perfectly into the k8s setup. Here's [how sweep works](https://docs.wandb.com/sweeps/quickstart):

1. declare the sweep config to define the search space
2. initialize a sweep, which will output an agent command
3. run agents using the command produced to take hyperparameter suggestions from the sweep controller

Step 2 and 3 are usually done manually as shown in the doc. When running on k8s, these should be executed in sequence.

This guide shows how to run a sweep on k8s. We assume basic familiarity with Docker and running things on k8s. First, let's ensure we have the prerequisites for running a sweep job:

* a sweep config YAML file `sweep.yaml`
* a python train script `train.py` for sweep agent to run
* a Dockerfile to build the image for running the sweep command and sweep agent\(s\)
* a k8s cluster

## Running Sweep as a Kubernetes Job

A sweep will run as a Job on k8s. Recall that we want to first run the sweep command, then use its output to run agent\(s\).

We will use the **initContainer** to run the sweep command, then save the output to a volume mount. When the job container spawns, we will mount it to the same volume mount to retrieve the agent command, then execute that command. An example k8s Job config `sweep-job.yaml` is shown below:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: dev-sweep
  labels:
    env: dev
spec:
  parallelism: 1
  backoffLimit: 0
  template:
    metadata:
      name: dev-sweep
      labels:
        env: dev
    spec:
      restartPolicy: Never
      containers:
      - name: dev-sweep
        image: 'your-docker-registry.com/foo/bar:0.0.1'
        imagePullPolicy: Always
        command: ['/bin/bash']
        args:
          - '--login'
          - '-c'
          - 'cmd=$(cat /tmp/sweep/sweep-agent.txt); for i in {1..2}; do eval $cmd & done; wait'
        resources:
          limits:
            cpu: 2
            memory: 10Gi
            nvidia.com/gpu: 1  # requesting 1 GPU
        volumeMounts:
        - name: dev-sweepcmd
          mountPath: /tmp/sweep
      initContainers:
      - name: dev-init-sweep
        image: 'your-docker-registry.com/foo/bar:0.0.1'
        imagePullPolicy: Always
        command: ['/bin/bash']
        args:
          - '--login'
          - '-c'
          - 'wandb sweep config/sweep/tst_bayes.yaml 2>&1 | tee /tmp/sweep/sweep-output.txt; echo `expr "$(cat /tmp/sweep/sweep-output.txt)" : ".*\(wandb agent.*\)"` > /tmp/sweep/sweep-agent.txt;'
        volumeMounts:
        - name: dev-sweepcmd
          mountPath: /tmp/sweep
      volumes:
      - name: dev-sweepcmd
        emptyDir: {}
```

Note that k8s containers cannot yet request fractional GPU. This means that we may want to run multiple agents in a single container. The example above runs 2 agents as shown in the container command `...for i in {1..2}` and the requested number of 2 cpus. Feel free to change those.

To deploy this, simply run:

```yaml
kubectl apply -f sweep-job.yaml
```

Users familiar with Helm may also templatize the job config, for instance the `env` variable `dev` or `live`, the number of agents, and the container resources. If you know Helm, this is straightforward to do.



