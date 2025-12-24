**MUST Requirement:** Ensure that access to accelerators from within containers is properly isolated and mediated by the Kubernetes resource management framework (device plugin or DRA) and container runtime, preventing unauthorized access or interference between workloads.

## Tests Executed
### Test 1: Ensure that access to accelerators from within containers mediated by the Kubernetes resource management framework (device plugin or DRA) and container runtime.

#### Step 1. Prepare the test environment

Prepare the test environment, including:
- Creating a Kubernetes 1.34 cluster
- Adding a GPU node pool
- Install the NVIDIA Driver by installing the latest CCE AI Suite (NVIDIA GPU) plugin

#### Step 2. Confirm that the device plugin pods are running

```bash
kubectl get pods -n kube-system
```

Output:

```output
NAME                                  READY   STATUS    RESTARTS   AGE
nvidia-gpu-device-plugin-6zf4b        1/1     Running   0          32m
```

#### Step 3. Confirm that the device plugin is advertising the correct amounts of GPU resources to kubelet

Run nvidia-smi to check the number of GPUs

```bash
nvidia-smi
```

Output:

```output
Tue Dec 23 10:11:04 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.86.15              Driver Version: 570.86.15      CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA L2                      Off |   00000000:00:0D.0 Off |                    0 |
| N/A   49C    P8             16W /   72W |       1MiB /  23034MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA L2                      Off |   00000000:00:0E.0 Off |                    0 |
| N/A   50C    P8             16W /   72W |       1MiB /  23034MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA L2                      Off |   00000000:00:0F.0 Off |                    0 |
| N/A   42C    P8             12W /   72W |       1MiB /  23034MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA L2                      Off |   00000000:00:10.0 Off |                    0 |
| N/A   41C    P8             12W /   72W |       1MiB /  23034MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

Check node info to confirm available number of GPUs registered by the device plugin

```bash
kubectl describe node
```

Output (Redacted):

```output
Capacity:
  cpu:                96
  ephemeral-storage:  1055758772Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  localssd:           0
  localvolume:        0
  memory:             792305148Ki
  nvidia.com/gpu:     4
  pods:               110
Allocatable:
  cpu:                95690m
  ephemeral-storage:  972987282665
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  localssd:           0
  localvolume:        0
  memory:             771057148Ki
  nvidia.com/gpu:     4
  pods:               110
```

#### Step 4. Create a deployment that requests the GPU resource

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pytorch-cuda-check
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pytorch-cuda-check
  template:
    metadata:
      labels:
        app: pytorch-cuda-check
    spec:
      containers:
        - name: pytorch-cuda-check
          image: nvcr.io/nvidia/pytorch:25.09-py3
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                python3 -c "import torch; print(torch.cuda.device_count())"
                sleep 30
              done
          resources:
            limits:
              nvidia.com/gpu: 1
```

Observe that the deployment is running:

```bash
kubectl get pods -w
```

Output:

```output                                    
NAME                                  READY   STATUS    RESTARTS   AGE
pytorch-cuda-check-68bc4bf767-pdgw7   1/1     Running   0          15m
```


#### Step 5. Update the deployment to remove the resource claim request in the Pod spec, the command in the container should fail.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pytorch-cuda-check
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pytorch-cuda-check
  template:
    metadata:
      labels:
        app: pytorch-cuda-check
    spec:
      containers:
        - name: pytorch-cuda-check
          image: nvcr.io/nvidia/pytorch:25.09-py3
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                python3 -c "import torch; print(torch.cuda.device_count())"
                sleep 30
              done
          # resources:
          #   limits:
          #     nvidia.com/gpu: 1
```

Retrieve the logs, they should show a failure:

```bash
kubectl logs pytorch-cuda-check-68bc4bf767-2zm7s
```

Output:

```output
/usr/local/lib/python3.12/dist-packages/torch/cuda/__init__.py:63: FutureWarning: The pynvml package is deprecated. Please install nvidia-ml-py instead. If you did not install pynvml directly, please report this to the maintainers of the package that installed pynvml for you.
  import pynvml  # type: ignore[import]
0
```

### Test 2: Ensure that access to accelerators from within containers is properly isolated.

#### Step 1. Create two Pods, each is allocated an accelerator resource. Run nvidia-smi to ensure that each pod can only access the allocated accelerator.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pytorch-cuda-check-2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pytorch-cuda-check
  template:
    metadata:
      labels:
        app: pytorch-cuda-check
    spec:
      containers:
        - name: pytorch-cuda-check
          image: nvcr.io/nvidia/pytorch:25.09-py3
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                python3 -c "import torch; print(torch.cuda.device_count())"
                sleep 30
              done
          resources:
            limits:
              nvidia.com/gpu: 1
```

Observe that the pods are running:

```bash
kubectl get pods -w
```

Output:

```output                                    
NAME                                    READY   STATUS    RESTARTS   AGE
pytorch-cuda-check-2-68bc4bf767-h8b7q   1/1     Running   0          3m55s
pytorch-cuda-check-2-68bc4bf767-jb2kq   1/1     Running   0          3m55s
```

Verify that each pod has been assigned a different GPU:

```bash
kubectl exec -it pytorch-cuda-check-2-68bc4bf767-h8b7q -- nvidia-smi -L
kubectl exec -it pytorch-cuda-check-2-68bc4bf767-jb2kq -- nvidia-smi -L
```

```output                                    
GPU 0: NVIDIA L2 (UUID: GPU-1dda2a1d-a678-15e2-f2cc-9b0622d3d523)
GPU 0: NVIDIA L2 (UUID: GPU-f71e5af2-ca6e-ccc7-e612-c9d23092c9b4)
```
