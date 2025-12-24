**MUST Requirement:** Provide a monitoring system capable of discovering and collecting metrics from workloads that expose them in a standard format (e.g. Prometheus exposition format). This ensures easy integration for collecting key metrics from common AI frameworks and servers.

#### Step 1. Prepare the test environment

Prepare the test environment, including:
- Creating a Kubernetes 1.34 cluster
- Adding a GPU node pool
- Install the latest CCE AI Suite (NVIDIA GPU) plugin
- Install the latest Cloud Native Cluster Monitoring plugin

#### Step 2. Prepare a local LLM

Download [ArtGet](https://mirrors.tools.huawei.com/mirrorDetail/huggingface?mirrorName=huggingface&catalog=llms) and upload the installation script to the selected node

```bash
source artget_install.sh
```

Pull a LLM

```bash
artget pull --repo-id "deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B" --local-dir "/data/huggingface-cache"
```

#### Step 3. Create a vllm deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deepseek-r1-qwen
  namespace: default
  labels:
    app: deepseek-r1-qwen
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deepseek-r1-qwen
  template:
    metadata:
      labels:
        app: deepseek-r1-qwen
    spec:
      volumes:
      - name: model-volume
        hostPath:
          path: /data/huggingface-cache/DeepSeek-R1-Distill-Qwen-1.5B
          type: Directory
      - name: shm
        emptyDir:
          medium: Memory
          sizeLimit: "2Gi"
      containers:
      - name: deepseek-r1-qwen
        image: swr.cn-north-7.myhuaweicloud.com/cnai/vllm/vllm-openai:nightly
        command: ["/bin/sh", "-c"]
        args: [
          "vllm serve /models/DeepSeek-R1-Distill-Qwen-1.5B --trust-remote-code --max-model-len 4096"
        ]
        ports:
        - containerPort: 8000
          name: http
        resources:
          limits:
            cpu: "8"
            memory: 12G
            nvidia.com/gpu: "1"
          requests:
            cpu: "2"
            memory: 4G
            nvidia.com/gpu: "1"
        volumeMounts:
        - mountPath: /models/DeepSeek-R1-Distill-Qwen-1.5B
          name: model-volume
          readOnly: true
        - name: shm
          mountPath: /dev/shm
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 120
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 120
          periodSeconds: 5
          timeoutSeconds: 5
          failureThreshold: 3
```

Observe that the deployment is running:

```bash
kubectl get pods -w
```

Output:

```output                                    
NAME                              READY   STATUS                     RESTARTS   AGE
deepseek-r1-qwen-84b95998-qtwgn   1/1     Running                    0          78m
```

Create an PodMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: vllm-deepseek-metrics
  namespace: default
  labels:
    app: deepseek-r1-qwen
spec:
  selector:
    matchLabels:
      app: deepseek-r1-qwen
  podMetricsEndpoints:
  - port: http
    path: /metrics
    interval: 30s
```

#### Step 4. Verify that the LLM is operational with a simple prompt

Port forward to test

```bash
kubectl port-forward deployment/deepseek-r1-qwen 8000:8000
```

Output:

```output
Forwarding from 127.0.0.1:8000 -> 8000
Forwarding from [::1]:8000 -> 8000
```

In a new terminal, send a test request

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/models/DeepSeek-R1-Distill-Qwen-1.5B",
    "messages": [{"role": "user", "content": "Hello, how are you?"}],
    "max_tokens": 50
  }'
```

Updated output from port forwarding

```output
Forwarding from 127.0.0.1:8000 -> 8000
Forwarding from [::1]:8000 -> 8000
Handling connection for 8000
```

Response from the LLM

```output
{"id":"chatcmpl-10321d932eaf4f8f820241c6308e9285","object":"chat.completion","created":1766490573,"model":"/models/DeepSeek-R1-Distill-Qwen-1.5B","choices":[{"index":0,"message":{"role":"assistant","content":"Alright, the user greeted me with \"Hello, how are you?\" which is a common and friendly way to start a conversation. I should respond in a similar tone to keep the conversation going smoothly.\n\nI want to make sure I acknowledge their greeting and","refusal":null,"annotations":null,"audio":null,"function_call":null,"tool_calls":[],"reasoning_content":null},"logprobs":null,"finish_reason":"length","stop_reason":null,"token_ids":null}],"service_tier":null,"system_fingerprint":null,"usage":{"prompt_tokens":11,"total_tokens":61,"completion_tokens":50,"prompt_tokens_details":null},"prompt_logprobs":null,"prompt_token_ids":null,"kv_transfer_params":null}
```

#### Step 5. Collect engine-level stats from the exposed metrics port

```bash
curl http://10.3.1.10:8000/metrics
```

Output (Redacted):

```output
vllm:request_success_total{finished_reason="length"} 1.0
vllm:prompt_tokens_total 11.0
vllm:generation_tokens_total 50.0
vllm:time_to_first_token_seconds_sum 1.21 seconds
vllm:e2e_request_latency_seconds_sum 13.58 seconds
vllm:inter_token_latency_seconds_sum 12.37 seconds
vllm:kv_cache_usage_perc 0.0
vllm:num_requests_running 0.0
vllm:num_requests_waiting 0.0
vllm:prefix_cache_queries_total 11.0
vllm:prefix_cache_hits_total 0.0
```
