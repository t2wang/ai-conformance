**MUST Requirement:** Ensure that access to accelerators from within containers is properly isolated and mediated by the Kubernetes resource management framework (device plugin or DRA) and container runtime, preventing unauthorized access or interference between workloads.

## Tests Executed
### Test 1: Ensure that access to accelerators from within containers mediated by the Kubernetes resource management framework (device plugin or DRA) and container runtime.

#### Step 1. Prepare the test environment

Prepare the test environment, including:
- Creating a Kubernetes 1.34 cluster
- Adding a GPU node pool
- Install the NVIDIA Driver by installing the latest CCE AI Suite (NVIDIA GPU) plugin
- Uninstall the CCE AI Suite (NVIDIA GPU) plugin to remove device plugins (This does not uninstall the driver)

#### Step 2. Install the NVIDIA DRA driver

```bash
helm install nvidia-dra-driver-gpu ./nvidia-dra-driver-gpu-25.8.1.tgz \
    --create-namespace \
    --namespace nvidia-dra-driver-gpu \
    --set gpuResourcesEnabledOverride=true \
    --set nvidiaDriverRoot="/usr/local/nvidia" \
    --set resources.computeDomains.enabled=false \
    --set "kubeletPlugin.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[0].key=accelerator" \
    --set "kubeletPlugin.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[0].operator=Exists" \
    --set "kubeletPlugin.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[1].key=kubernetes.io/role" \
    --set "kubeletPlugin.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[1].operator=NotIn" \
    --set "kubeletPlugin.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[1].values[0]=virtual-kubelet" \
    --set "kubeletPlugin.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[1].values[1]=edge"
```

#### Step 3. Confirm that the DRA driver pods are running

```bash
kubectl get pods -n nvidia-dra-driver-gpu
```

Output:

```output
NAME                                         READY   STATUS    RESTARTS   AGE
nvidia-dra-driver-gpu-kubelet-plugin-vwkk2   1/1     Running   0          4h18m
nvidia-dra-driver-gpu-kubelet-plugin-qckbw   1/1     Running   0          4h18m
```

#### Step 4. Create a resource claim template and a deployment that requests the GPU resource

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-claim-template
spec:
  spec:
    devices:
      requests:
      - name: single-gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          allocationMode: ExactCount
          count: 1
```

Update the deployment to request a resource:

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
            claims:
              - name: single-gpu
      resourceClaims:
        - name: single-gpu
          resourceClaimTemplateName: gpu-claim-template
```

Observe that the deployment is running:

```bash
kubectl get pods -w
```

Output:

```output                                    
NAME                                  READY   STATUS    RESTARTS   AGE
pytorch-cuda-check-579b54bb75-j258b   1/1     Running   0          107s
```


#### Step 5. Update the Deployment to remove the resource claim request in the Pod spec, the command in the container should fail.

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
          #   claims:
          #     - name: single-gpu
      resourceClaims:
        - name: single-gpu
          resourceClaimTemplateName: gpu-claim-template
```

Retrieve the logs, they should show a failure:

```bash
kubectl logs pytorch-cuda-check-64f4757498-n82f6
```

Output:

```output
/usr/local/lib/python3.12/dist-packages/torch/cuda/__init__.py:63: FutureWarning: The pynvml package is deprecated. Please install nvidia-ml-py instead. If you did not install pynvml directly, please report this to the maintainers of the package that installed pynvml for you.
  import pynvml  # type: ignore[import]
1
/usr/local/lib/python3.12/dist-packages/torch/cuda/__init__.py:63: FutureWarning: The pynvml package is deprecated. Please install nvidia-ml-py instead. If you did not install pynvml directly, please report this to the maintainers of the package that installed pynvml for you.
  import pynvml  # type: ignore[import]
1
/usr/local/lib/python3.12/dist-packages/torch/cuda/__init__.py:63: FutureWarning: The pynvml package is deprecated. Please install nvidia-ml-py instead. If you did not install pynvml directly, please report this to the maintainers of the package that installed pynvml for you.
  import pynvml  # type: ignore[import]
1
/usr/local/lib/python3.12/dist-packages/torch/cuda/__init__.py:63: FutureWarning: The pynvml package is deprecated. Please install nvidia-ml-py instead. If you did not install pynvml directly, please report this to the maintainers of the package that installed pynvml for you.
  import pynvml  # type: ignore[import]
1
```

### Test 2: Ensure that access to accelerators from within containers is properly isolated.

#### Step 1. Create two Pods, each is allocated an accelerator resource. Execute a command in one Pod to attempt to access the other Pod’s accelerator, and should be denied

This can be verified by running this test [https://github.com/kubernetes/kubernetes/blob/v1.34.1/test/e2e/dra/dra.go#L180](https://github.com/kubernetes/kubernetes/blob/v1.34.1/test/e2e/dra/dra.go#L180) 

With a 1.34 AKS cluster:

```
% wget https://dl.k8s.io/$v1.34.1/kubernetes-test-linux-amd64.tar.gz
% tar -xzf kubernetes-test-linux-amd64.tar.gz
% cd kubernetes/test/bin
% ./e2e.test \
    -ginkgo.focus='must map configs and devices to the right containers' \
    --kubeconfig=/root/.kube/config \
    --provider=local \
    --report-dir=/root/test-results
==============================================================================
  I1218 20:15:54.604293  237215 e2e.go:109] Starting e2e run "d30d1905-d329-489c-86d7-0dc18dfb8ac4" on Ginkgo node 1
Running Suite: Kubernetes e2e suite - /root/kubernetes/test/bin
===============================================================
Random Seed: 1766060153 - will randomize all specs

Will run 1 of 7137 specs
•

Ran 1 of 7137 Specs in 13.073 seconds
SUCCESS! -- 1 Passed | 0 Failed | 0 Pending | 7136 Skipped
PASS
```
