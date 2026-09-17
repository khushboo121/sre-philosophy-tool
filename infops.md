### AI Inference Operations & SRE Playbook ###

AI Inference Operations & SRE PlaybookThis document serves as the core reference playbook for managing, scaling, and troubleshooting high-throughput Machine Learning and Large Language Model (LLM) inference workloads running on AWS EKS using vLLM and OpenTelemetry (OTel).1. Core Architecture Blueprint[ Incoming User Requests ]
            │
            ▼
[ AWS ALB (Application Load Balancer) ] ── (Injects W3C Trace Context headers)
            │
            ▼
[ Amazon EKS Cluster ] ◄─── (Managed by Karpenter for Dynamic Node Scaling)
            │
            ├──► [ Pod: vLLM Inference Engine ]
            │         ├── Runs LLM (e.g., Llama 3) on Dedicated GPU Nodes
            │         ├── Generates spans for Pre-fill, Decoding, and KV allocation
            │         └── Exposes OTel/Prometheus metrics & Traces
            │
            └──► [ Pod: OpenTelemetry Collector ]
                      ├── Scrapes metrics and collects trace spans from vLLM
                      └── Exports data to AWS X-Ray, Jaeger, or Honeycomb
2. Platform Commitments: SLAs vs. SLOs vs. SLIsTo maintain enterprise trust, the reliability stack must clearly distinguish between internal engineering targets and contractual business obligations.A. Service Level Agreements (SLA)The formal commitment made to end-users or customers. Breaches result in financial refunds or penalties.Enterprise Inference Availability SLA: 99.9% availability per calendar month. A request is considered unavailable if it returns an HTTP 5xx error or takes longer than 30 seconds to respond.B. Service Level Objectives (SLO)The internal target set by the SRE team to keep the system healthy before it breaches the SLA. Usually guarded by an Error Budget.Interactive Latency Target: 95% of successful requests must achieve a Time to First Token (TTFT) under 1.5 seconds over a rolling 30-day window.Streaming Smoothness Target: 99% of active inference sessions must maintain an Inter-Token Latency (ITL) under 50ms/token over a rolling 7-day window.C. Service Level Indicators (SLI)The actual quantifiable measurement used to see if the platform is meeting its SLOs.Metric ObjectiveSLI TypeProduction PromQL / OTel Metric FormulaInteractive LatencyLatency Histogramsum(rate(vllm:time_to_first_token_seconds_bucket{le="1.5"}[5m])) / sum(rate(vllm:time_to_first_token_seconds_count[5m]))Streaming ThroughputLatency Histogramsum(rate(vllm:time_per_output_token_seconds_bucket{le="0.05"}[5m])) / sum(rate(vllm:time_per_output_token_seconds_count[5m]))Platform AvailabilityRate Ratiosum(rate(http_requests_total{status=~"2.."}[5m])) / sum(rate(http_requests_total[5m]))3. Infrastructure & Auto-Scaling ArchitectureAI workloads cannot scale natively on standard CPU/Memory thresholds due to vLLM's static KV cache pre-allocation. Scaling must be driven by queue volume and immediately provisioned by Karpenter.Karpenter NodePool Configuration (nodepool.yaml)This configuration ensures Karpenter bypasses slow generic scaling and immediately provisions high-throughput, right-sized AWS GPU instances.yamlapiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: gpu-inference-pool
  namespace: kube-system
spec:
  template:
    spec:
      requirements:
        - key: "karpenter.k8s.aws/instance-category"
          operator: In
          values: ["g", "p"]           # Natively provisions G5 (A10G) or P4/P5 (A100/H100) instances
        - key: "karpenter.k8s.aws/instance-generation"
          operator: Gt
          values: ["4"]
        - key: "kubernetes.io/arch"
          operator: In
          values: ["amd64"]
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["on-demand"]        # Ensures stability; use "spot" only for non-critical workloads
      nodeClassRef:
        name: default-gpu-settings
  limits:
    cpu: 1000
    memory: 4000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h                   # Automatically recycles nodes after 30 days
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default-gpu-settings
  namespace: kube-system
spec:
  amiFamily: Bottlerocket              # Optimized minimal Linux container OS with built-in NVIDIA drivers
  subnetSelectorTerms:
    - tags:
        alpha.eksctl.io/cluster-name: "ai-inference-cluster"
  securityGroupSelectorTerms:
    - tags:
        aws:eks:cluster-name: "ai-inference-cluster"
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        iops: 3000
        throughput: 125
Use code with caution.KEDA Queue-Based Autoscaler (keda-scaler.yaml)yamlapiVersion: keda.sh/v1alpha1
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
  cooldownPeriod: 300    
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://aps-workspace-service.monitoring.svc.cluster.local:8080
      metricName: vllm_num_requests_waiting
      query: sum(vllm:num_requests_waiting)
      threshold: '3'     # Trigger scale up if queue contains > 3 items
Use code with caution.4. Distributed Tracing for AI PipelinesTo track an inference request across your API Gateway, Guardrails, and vLLM engines, you must propagate standard W3C Trace Contexts. This lets you isolate whether latency is occurring inside the model execution loop or your application routing logic.vLLM Trace Instrumentation ConfigurationvLLM supports native OpenTelemetry tracing configuration. Pass the OTel exporter endpoints directly into the container variables.yamlapiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-llama3
  namespace: default
spec:
  template:
    spec:
      containers:
      - name: vllm-container
        image: vllm/vllm-openai:latest
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector.monitoring.svc.cluster.local:4317" # Internal OTel collector endpoint
        - name: OTEL_SERVICE_NAME
          value: "llama3-8b-inference-service"
        args: 
        - "--model"
        - "/data/hf-cache/hub/models--meta-llama--Meta-Llama-3-8B-Instruct"
        - "--port"
        - "8000"
        - "--otlp-traces-endpoint" # Activates native tracing hooks inside vLLM
        value: "otel-collector.monitoring.svc.cluster.local:4317"
Use code with caution.Anatomy of an AI Trace Span BreakdownWhen analyzing a trace trace in your UI (Jaeger, Honeycomb, or AWS X-Ray), look for these structured sub-spans:[Parent Span: HTTP POST /v1/chat/completions] ────────────────────────────────────────── (Total: 1800ms)
   ├── [Span: api-gateway-guardrail-check] ─────────────────────── (Latency: 150ms)
   └── [Span: vllm-inference-engine] ─────────────────────────────────────────────────── (Latency: 1650ms)
         ├── [Span: prompt-tokenization] ──────── (Latency: 10ms)
         ├── [Span: prefill-phase-execution] ─── (Latency: 240ms - First token calculated here)
         └── [Span: decoding-loop-tokens] ────────────────────────────────────────────── (Latency: 1400ms)
api-gateway-guardrail-check: Measures the time spent passing text through verification engines (like Llama Guard) before submitting it to the GPU.prefill-phase-execution: Measures the highly intensive phase where the model reads your entire input prompt block. If this is bloated, the prompt size is too massive for the current batch config.decoding-loop-tokens: The step-by-step sequential processing loop creating output text. High duration here points straight to high Inter-Token Latency constraints on hardware bandwidth.5. Troubleshooting Flow: High Latency (TTFT > 2s)                  [ Latency Alert: TTFT > 2s ]
                                │
            ┌───────────────────┴───────────────────┐
            ▼                                       ▼
  [ Are Request Queues Spiking? ]       [ Queue is Empty / Low ]
            │                                       │
     Yes ──► (Scale Issue)                   No ──► (Hardware/Memory Bottleneck)
            │                                       │
            ▼                                       ▼
     Verify Karpenter NodePool status.       Check `vllm:gpu_cache_usage_factor`.
     Look for `MaxLimitExceeded` errors     If > 0.95, text data is swapping to
     or AWS Compute Quota blocks.            slower system memory (DRAM).Out-Of-Memory (OOM) failures.
