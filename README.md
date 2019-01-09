# Nvidia Docker Install Hook for KOPS


### Docker version
Docker, nvidia-docker and nvidia-container-runtime has to agree on a docker version.

The version of nvidia-docker and nvidia-container-runtime installed by
`nvidia-docker-installer.sh` is expecting the Kubernetes cluster to run docker
version 18.6.1.

Lets go ahead and change the docker version used by the cluster

```
kops edit cluster
```

You need to add the following under `spec`.

```
docker:
  logDriver: json-file
  version: 18.6.1
```

Perform a rolling update and verify that nodes are now running the correct
version of docker.

```
kops rolling-update cluster --yes
kubectl get nodes -o wide
```


```
kops create ig gpu-nodes
```

```
apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  creationTimestamp: null
  name: gpu-nodes
spec:
  image: kope.io/k8s-1.8-debian-stretch-amd64-hvm-ebs-2018-02-08
  machineType: p2.xlarge
  taints:
  - nvidia.com/gpu=:NoSchedule
  hooks:
  - before:
    - docker.service
    manifest: |
      Type=oneshot
      ExecStart=/bin/bash -c '/usr/bin/curl -L -S -f https://raw.githubusercontent.com/brunsgaard/kops-nvidia-docker-installer/master/nvidia-docker-installer.sh | /bin/bash'
    name: nvidia-docker-install.service
  maxSize: 2
  minSize: 2
  nodeLabels:
    kops.k8s.io/instancegroup: gpu-nodes
  role: Node
  subnets:
  - eu-central-1a
  - eu-central-1b
```

Update the cluster to start the nodes
```
kops update cluster --yes
```

Apply the nvidia device plugin

```
kubectl apply -f nvidia-device-plugin.yml
```

Run a testjob

```
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vector-add
spec:
  restartPolicy: OnFailure
  nodeSelector:
    beta.kubernetes.io/instance-type: p2.xlarge
  tolerations:
  - key: nvidia.com/gpu
    effect: NoSchedule
  containers:
    - name: cuda-vector-add
      # https://github.com/kubernetes/kubernetes/blob/v1.7.11/test/images/nvidia-cuda/Dockerfile
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
```
