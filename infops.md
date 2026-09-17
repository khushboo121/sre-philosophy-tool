### AI Inference Operations & SRE Playbook

This document serves as the core reference playbook for managing, scaling, and troubleshooting high-throughput Machine Learning and Large Language Model (LLM) inference workloads running on **AWS EKS** using **vLLM** and **OpenTelemetry (OTel)**. 

### 1. Core Architecture Blueprint

[ Incoming User Requests ]
            │
            ▼
[ AWS ALB (Application Load Balancer) ]
            │
            ▼
[ Amazon EKS Cluster ] ◄─── (Managed by Karpenter for Dynamic Node Scaling)
            │
            ├──► [ Pod: vLLM Inference Engine ]
            │         ├── Runs LLM (e.g., Llama 3) on Dedicated GPU Nodes
            │         └── Exposes OTel/Prometheus metrics (Latency, KV Cache, Queue)
            │
            └──► [ Pod: OpenTelemetry Collector ]
                      ├── Scrapes metrics from vLLM via OTel Operator
                      └── Exports data to Central Observability Backend

### 2. Observability Strategy & AI SLOs

Traditional SRE Golden Signals must shift from tracking basic infrastructure saturation to model execution characteristics. AI workloads are highly compute-bound and multi-modal. 

### The AI Golden Signals

* **Time to First Token (TTFT):** The duration between the user submitting a prompt and receiving the very first character of the output. Drives user-perceived responsiveness.
* **Inter-Token Latency (ITL):** The average time between subsequent tokens during text streaming. Determines if text streams naturally or lags.
* **KV Cache Saturation:** The volume of GPU VRAM consumed by conversation context. Spikes here cause severe resource bottlenecks.
* **Model Quality Drift:** Quantitative variation in output accuracy, safety boundaries, or performance drift measured over time.

### Target Service Level Objectives (SLOs)

MetricSLO TargetSLI Metric Formula (Prometheus/OTel)
****Interactive Responsiveness****
**95%** of user requests must achieve a TTFT under **1.5 seconds**.sum(rate(vllm:time_to_first_token_seconds_bucket{le="1.5"}[5m])) / sum(rate(vllm:time_to_first_token_seconds_count[5m]))
****Streaming Throughput****
**99%** of streaming sessions must maintain an ITL under **50ms/token**.sum(rate(vllm:time_per_output_token_seconds_bucket{le="0.05"}[5m])) / sum(rate(vllm:time_per_output_token_seconds_count[5m]))
****Platform Availability****
**99.9%** of inference requests must return an HTTP 200 without OOM failures.sum(rate(http_requests_total{status=~"2.."}[5m])) / sum(rate(http_requests_total[5m]))

### 3. Infrastructure & Auto-Scaling Architecture

AI workloads cannot scale natively on standard CPU/Memory thresholds. The playbook mandates **Event-Driven Scaling via KEDA** based on the vLLM waiting request queue. 

### KEDA Queue-Based Autoscaler

This deployment targets the waiting queue vllm_num_requests_waiting instead of memory tracking, because vLLM pre-allocates 90% of memory on boot to use as a KV cache buffer. 

yaml

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm-autoscaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm-llama3
  minReplicaCount: 1     
  maxReplicaCount: 10    
  cooldownPeriod: 300    # Prevent scaling "flapping" due to GPU boot times
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://aps-workspace-service.monitoring.svc.cluster.local:8080
      metricName: vllm_num_requests_waiting
      query: sum(vllm:num_requests_waiting)
      threshold: '3'     # Scale out immediately if more than 3 requests are queued

Use code with caution.

### Eliminating Cold Starts: EBS CSI Snapshot Cloning

To avoid downloading large model weights (e.g., 15GB+) over the internet during a traffic surge, nodes clone an optimized AWS EBS Snapshot instantly upon launch. 

yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vllm-model-cache-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: llama3-weights-snapshot  # Pre-baked AWS EBS Snapshot containing model weights
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io

Use code with caution.

### 4. GPU Resource Management Standards

* **Multi-Instance GPU (MIG):** Mandated for NVIDIA A100/H100 nodes. Physically slices hardware into independent, secure execution blocks to ensure dedicated memory and processing boundaries for micro-models.
* **Fractional/Time-Slicing:** Reserved for lower tier hardware (like AWS G5 / A10G instances) running inside dev, testing, or lower-priority staging clusters.

### 5. Troubleshooting Flow: High Latency (TTFT > 2s)

When an OpenTelemetry alert marks an SLO violation, track down the root cause in this order: 

### Phase 1: Capacity Assessment

* **Metric:** Check vllm:num_requests_waiting.
* **Diagnosis:** If this metric is elevated, the infrastructure is capacity-constrained.
* **Remediation:** Check Karpenter provisioning sheets. Verify if the cluster has breached AWS EC2 Service Quotas for GPU instance families (e.g., P or G families).

### Phase 2: Memory & Cache Saturation

* **Metric:** Check vllm:gpu_cache_usage_factor.
* **Diagnosis:** If the factor approaches 1.0, your GPU VRAM is completely saturated by active request contexts.
* **Remediation:** vLLM will begin swapping conversation history chunks to CPU system RAM. This severely breaks latency standards. Increase your node count or optimize your model parameters using quantization techniques (e.g., AWQ/GPTQ).

### Phase 3: Hardware Profiling

* **Metric:** Check NVML data (nvidia_smi_power_readings_watts).
* **Diagnosis:** If power draws or temperature metrics are high, the hardware may be thermal-throttling.
* **Remediation:** Evacuate tasks from the node to trigger Karpenter replacement, or implement smaller batch configurations to ease execution load.

### 6. AI Platform Reliability Patterns

* **Fallback Redundancy:** Configure API Gateways (e.g., Envoy) to intercept prolonged HTTP 503 or 429 errors from the EKS cluster and automatically forward requests to a serverless model backup (e.g., Amazon Bedrock) until Karpenter adds compute capacity.
* **Context-Based Intelligent Routing:** Route simple requests (short prompts) to small, highly efficient 8B model clusters on cheaper hardware, reserving resource-intensive 70B model clusters for multi-turn conversations or massive context sizes.
* **Upstream Ingress Rate Limiting:** Implement strict token bucket filters at the API gateway level to drop excessive bursts of input before they hit the underlying EKS workloads, avoiding cascading GPU Out-Of-Memory (OOM) failures.
