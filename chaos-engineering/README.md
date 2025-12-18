# SRE Philosophy Tool

A comprehensive toolkit for implementing Site Reliability Engineering (SRE) practices, including chaos engineering with VMware Mangle, SLI/SLO/SLA monitoring, and user metrics measurement.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Mangle VMware Chaos Engineering in Kubernetes](#mangle-vmware-chaos-engineering-in-kubernetes)
- [Creating Dashboards for SLI, SLO, and SLA](#creating-dashboards-for-sli-slo-and-sla)
- [User Metrics to Measure](#user-metrics-to-measure)
- [Best Practices](#best-practices)
- [References](#references)

## Overview

This toolkit helps organizations implement SRE principles by:
- Running controlled chaos experiments using VMware Mangle
- Monitoring Service Level Indicators (SLIs)
- Defining and tracking Service Level Objectives (SLOs)
- Measuring Service Level Agreements (SLAs)
- Collecting and analyzing user-centric metrics

## Prerequisites

- Kubernetes cluster (v1.19+)
- `kubectl` configured to access your cluster
- `helm` v3.x installed
- Wavefront account and access (VMware Tanzu Observability)
- Wavefront proxy installed in your cluster (for metrics collection)
- VMware vCenter access (for Mangle VMware-specific chaos experiments)
- Sufficient cluster resources (minimum 4 CPU, 8GB RAM recommended)

## Setup

### 1. Clone the Repository

```bash
git clone <repository-url>
cd sre-philosophy-tool
```

### 2. Install Wavefront Proxy

The Wavefront proxy collects metrics from your Kubernetes cluster and forwards them to Wavefront:

```bash
# Add Wavefront Helm repository
helm repo add wavefront https://wavefronthq.github.io/helm/
helm repo update

# Create namespace
kubectl create namespace wavefront

# Install Wavefront proxy
# Replace YOUR_CLUSTER_NAME and YOUR_WAVEFRONT_URL with your values
helm install wavefront wavefront/wavefront \
  --namespace wavefront \
  --set clusterName=YOUR_CLUSTER_NAME \
  --set wavefront.url=YOUR_WAVEFRONT_URL \
  --set wavefront.token=YOUR_WAVEFRONT_API_TOKEN
```

Alternatively, install using Kubernetes manifests:

```bash
# Create secret with Wavefront API token
kubectl create secret generic wavefront-secret \
  --from-literal=token=YOUR_WAVEFRONT_API_TOKEN \
  -n wavefront

# Apply Wavefront proxy deployment
kubectl apply -f https://raw.githubusercontent.com/wavefrontHQ/helm/master/wavefront/templates/proxy-deployment.yaml
```

### 3. Configure Wavefront Integration

#### Install Wavefront Kubernetes Integration

```bash
# Install Wavefront Kubernetes integration using Helm
helm install wavefront-k8s wavefront/wavefront-kubernetes \
  --namespace wavefront \
  --set wavefront.url=YOUR_WAVEFRONT_URL \
  --set wavefront.token=YOUR_WAVEFRONT_API_TOKEN \
  --set clusterName=YOUR_CLUSTER_NAME
```

#### Configure Wavefront Collector (Optional)

For advanced metrics collection, install the Wavefront collector:

```bash
helm install wavefront-collector wavefront/wavefront-collector \
  --namespace wavefront \
  --set wavefront.url=YOUR_WAVEFRONT_URL \
  --set wavefront.token=YOUR_WAVEFRONT_API_TOKEN
```

### 4. Access Wavefront

1. Log in to your Wavefront instance at `https://YOUR_INSTANCE.wavefront.com`
2. Navigate to **Dashboards** to create custom dashboards
3. Use **Alerts** to set up SLO-based alerting
4. Access **Metrics** browser to explore available metrics

### 5. Install Mangle

Mangle is VMware's chaos engineering tool. Install it in your Kubernetes cluster:

```bash
# Clone Mangle repository (reference: https://github.com/vmware-archive/mangle)
git clone https://github.com/vmware-archive/mangle.git
cd mangle

# Install Mangle using Helm or Kubernetes manifests
# Option 1: Using Helm (if available)
helm install mangle ./helm-chart --namespace mangle --create-namespace

# Option 2: Using Kubernetes manifests
kubectl create namespace mangle
kubectl apply -f deploy/kubernetes/ -n mangle
```

### 6. Verify Installation

```bash
# Check Mangle pods
kubectl get pods -n mangle

# Check Mangle services
kubectl get svc -n mangle

# Check Wavefront proxy
kubectl get pods -n wavefront

# Verify metrics are being sent to Wavefront
# Log into Wavefront UI and check Metrics browser for:
# - kubernetes.* metrics
# - wavefront.proxy.* metrics
# - Your application metrics (if configured)
```

## Mangle VMware Chaos Engineering in Kubernetes

### Overview

Mangle enables you to inject controlled failures into your Kubernetes infrastructure to test resilience and validate SLOs under adverse conditions.

### Configuration

#### 1. Configure Mangle for VMware vCenter

Create a ConfigMap with vCenter credentials:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mangle-vcenter-config
  namespace: mangle
data:
  vcenter.host: "vcenter.example.com"
  vcenter.username: "administrator@vsphere.local"
  vcenter.insecure: "true"
```

Create a Secret for sensitive data:

```bash
kubectl create secret generic mangle-vcenter-secret \
  --from-literal=password='your-vcenter-password' \
  -n mangle
```

#### 2. Access Mangle API

```bash
# Port forward to Mangle service
kubectl port-forward -n mangle svc/mangle-service 8080:8080

# Or expose via LoadBalancer/Ingress
kubectl patch svc mangle-service -n mangle -p '{"spec":{"type":"LoadBalancer"}}'
```

### Running Chaos Experiments

#### Example 1: CPU Stress on Kubernetes Pods

```bash
# Create a CPU stress fault
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -d '{
    "faultName": "cpu-stress",
    "faultSpec": {
      "fault": "CPU",
      "endpointName": "k8s-cluster",
      "cpuLoad": 80,
      "timeoutInMilliseconds": 300000
    }
  }'
```

#### Example 2: Memory Stress

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -d '{
    "faultName": "memory-stress",
    "faultSpec": {
      "fault": "MEMORY",
      "endpointName": "k8s-cluster",
      "memoryLoad": 70,
      "timeoutInMilliseconds": 300000
    }
  }'
```

#### Example 3: VMware VM Power Off

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -d '{
    "faultName": "vm-power-off",
    "faultSpec": {
      "fault": "VM_STATE",
      "endpointName": "vcenter-endpoint",
      "vmName": "k8s-worker-node-1",
      "vmState": "POWER_OFF",
      "timeoutInMilliseconds": 600000
    }
  }'
```

#### Example 4: Network Latency

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -d '{
    "faultName": "network-latency",
    "faultSpec": {
      "fault": "NETWORK",
      "endpointName": "k8s-cluster",
      "latency": 500,
      "timeoutInMilliseconds": 300000
    }
  }'
```

### Monitoring Chaos Experiments

While chaos experiments run, monitor your application metrics:

```bash
# Watch pod status
kubectl get pods -w

# Check application logs
kubectl logs -f deployment/your-app

# Monitor Wavefront metrics
# Access Wavefront UI and query relevant metrics
# Use Wavefront Query Language (WQL) to query metrics
```

### Remediation

Mangle automatically remediates faults after the timeout period, or you can manually remediate:

```bash
# List active faults
curl http://localhost:8080/mangle/api/rest/v1/faults

# Remediate a specific fault
curl -X DELETE http://localhost:8080/mangle/api/rest/v1/faults/{faultId}
```

## Creating Dashboards for SLI, SLO, and SLA

### Understanding SLI, SLO, and SLA

- **SLI (Service Level Indicator)**: A quantitative measure of some aspect of the level of service provided
- **SLO (Service Level Objective)**: A target value or range of values for an SLI
- **SLA (Service Level Agreement)**: A business agreement that includes consequences for missing SLOs

### Setting Up Wavefront Queries

#### Common SLI Metrics

Create these Wavefront Query Language (WQL) queries in Wavefront:

**Availability SLI:**
```wql
# Request success rate
sum(rate(ts(http.requests.total, status=~"2.."))) / sum(rate(ts(http.requests.total)))
```

**Latency SLI:**
```wql
# 95th percentile latency
percentile(95, ts(http.request.duration, source="*"))
```

**Throughput SLI:**
```wql
# Requests per second
sum(rate(ts(http.requests.total)))
```

**Error Rate SLI:**
```wql
# Error rate percentage
(sum(rate(ts(http.requests.total, status=~"5.."))) / sum(rate(ts(http.requests.total)))) * 100
```

### Creating Wavefront Dashboards

#### 1. SLI Dashboard

Create a new dashboard in Wavefront with the following charts:

**Chart 1: Availability**
- **Query:**
  ```wql
  (sum(rate(ts(http.requests.total, status=~"2.."))) / sum(rate(ts(http.requests.total)))) * 100
  ```
- **Chart Type:** Single Stat
- **Unit:** Percent (0-100)
- **Thresholds:** 
  - Green: >99.9%
  - Yellow: 99-99.9%
  - Red: <99%

**Chart 2: Latency (p50, p95, p99)**
- **Query:**
  ```wql
  percentile(50, ts(http.request.duration, source="*"))
  percentile(95, ts(http.request.duration, source="*"))
  percentile(99, ts(http.request.duration, source="*"))
  ```
- **Chart Type:** Line Chart
- **Unit:** Seconds
- **Y-axis:** Logarithmic scale recommended

**Chart 3: Error Rate**
- **Query:**
  ```wql
  (sum(rate(ts(http.requests.total, status=~"5.."))) / sum(rate(ts(http.requests.total)))) * 100
  ```
- **Chart Type:** Line Chart
- **Unit:** Percent
- **Y-axis:** 0-100%

**Chart 4: Request Rate**
- **Query:**
  ```wql
  sum(rate(ts(http.requests.total)))
  ```
- **Chart Type:** Line Chart
- **Unit:** Requests/sec

#### 2. SLO Dashboard

Create charts that compare current SLI values against SLO targets:

**Chart 1: Availability SLO Compliance**
- **Query:**
  ```wql
  if((sum(rate(ts(http.requests.total, status=~"2.."))) / sum(rate(ts(http.requests.total)))) >= 0.999, 1, 0)
  ```
- **Chart Type:** Single Stat
- **Description:** Shows 1 if meeting 99.9% availability SLO, 0 otherwise
- **Thresholds:** Green: 1, Red: 0

**Chart 2: Error Budget**
- **Query:**
  ```wql
  1 - (sum(rate(ts(http.requests.total, status=~"5.."))) / sum(rate(ts(http.requests.total))))
  ```
- **Chart Type:** Gauge
- **Min:** 0, **Max:** 1
- **Thresholds:** 
  - Green: >0.999
  - Yellow: 0.99-0.999
  - Red: <0.99

**Chart 3: SLO Burn Rate**
- **Query:**
  ```wql
  (sum(rate(ts(http.requests.total, status=~"5.."))) / sum(rate(ts(http.requests.total)))) / 0.001
  ```
- **Chart Type:** Line Chart
- **Description:** Shows how fast error budget is being consumed (1x = normal, >1x = burning fast)
- **Reference Line:** Add at y=1 to show normal burn rate

**Chart 4: 30-Day SLO Compliance**
- **Query:**
  ```wql
  avg(ts((sum(rate(ts(http.requests.total, status=~"2.."))) / sum(rate(ts(http.requests.total)))), source="*"), 30d) * 100
  ```
- **Chart Type:** Single Stat
- **Unit:** Percent

#### 3. SLA Dashboard

Create a business-focused dashboard showing SLA compliance:

**Chart 1: Monthly Uptime**
- **Query:**
  ```wql
  avg(ts((sum(rate(ts(http.requests.total, status=~"2.."))) / sum(rate(ts(http.requests.total)))), source="*"), 30d) * 100
  ```
- **Chart Type:** Single Stat
- **Unit:** Percent
- **Thresholds:** Based on your SLA (e.g., 99.9% = Green)

**Chart 2: SLA Violations This Month**
- **Query:**
  ```wql
  count(ts((sum(rate(ts(http.requests.total, status=~"2.."))) / sum(rate(ts(http.requests.total)))) < 0.999, source="*"), 30d)
  ```
- **Chart Type:** Single Stat
- **Description:** Count of times SLA was violated

**Chart 3: Service Health Score**
- **Query:**
  ```wql
  (
    (sum(rate(ts(http.requests.total, status=~"2.."))) / sum(rate(ts(http.requests.total)))) * 0.4 +
    (1 - percentile(95, ts(http.request.duration, source="*")) / 1.0) * 0.3 +
    (1 - sum(rate(ts(http.requests.total, status=~"5.."))) / sum(rate(ts(http.requests.total)))) * 0.3
  ) * 100
  ```
- **Chart Type:** Gauge
- **Min:** 0, **Max:** 100
- **Description:** Composite health score (0-100)

### Importing Pre-built Dashboards

Wavefront provides pre-built dashboards for Kubernetes and common integrations:

1. Go to **Dashboards** → **Browse**
2. Search for:
   - **Kubernetes Dashboard** - Comprehensive K8s monitoring
   - **Application Performance** - APM metrics
   - **Infrastructure Overview** - System metrics
3. Click **Clone** to customize for your environment

### Setting Up Wavefront Alerts

Create alerts in Wavefront for SLO violations:

#### Alert 1: High Error Rate

1. Go to **Alerts** → **Create Alert**
2. **Alert Name:** High Error Rate
3. **Query:**
   ```wql
   (sum(rate(ts(http.requests.total, status=~"5.."))) / sum(rate(ts(http.requests.total)))) * 100
   ```
4. **Condition:** Alert when the value is **> 1%** for **5 minutes**
5. **Severity:** Critical
6. **Notification Targets:** Configure email, PagerDuty, Slack, etc.

#### Alert 2: Availability Below SLO

1. **Alert Name:** Availability Below SLO
2. **Query:**
   ```wql
   (sum(rate(ts(http.requests.total, status=~"2.."))) / sum(rate(ts(http.requests.total)))) * 100
   ```
3. **Condition:** Alert when the value is **< 99.9%** for **5 minutes**
4. **Severity:** Critical
5. **Notification Targets:** Configure as needed

#### Alert 3: High Latency

1. **Alert Name:** High Latency
2. **Query:**
   ```wql
   percentile(95, ts(http.request.duration, source="*"))
   ```
3. **Condition:** Alert when the value is **> 1.0 seconds** for **5 minutes**
4. **Severity:** Warning
5. **Notification Targets:** Configure as needed

#### Alert 4: Error Budget Depletion

1. **Alert Name:** Error Budget Depletion
2. **Query:**
   ```wql
   1 - (sum(rate(ts(http.requests.total, status=~"5.."))) / sum(rate(ts(http.requests.total))))
   ```
3. **Condition:** Alert when the value is **< 0.99** for **10 minutes**
4. **Severity:** Critical
5. **Description:** Error budget is being consumed too quickly

### Wavefront Alert Best Practices

- Use **Alert Tags** to organize alerts by service, team, or environment
- Set up **Alert Routing** to send alerts to appropriate teams
- Configure **Alert Fatigue** settings to prevent notification spam
- Use **Alert States** to track alert lifecycle (Firing, Acknowledged, Resolved)
- Create **Alert Snoozing** rules for planned maintenance windows

## User Metrics to Measure

### RED Metrics (Rate, Errors, Duration)

These are the fundamental metrics for any service:

1. **Rate (Throughput)**
   - Requests per second
   - Transactions per second
   - Events per second
   - **Wavefront Query:**
     ```wql
     sum(rate(ts(http.requests.total)))
     ```

2. **Errors**
   - Error rate percentage
   - Error count by type
   - 4xx vs 5xx errors
   - **Wavefront Query:**
     ```wql
     (sum(rate(ts(http.requests.total, status=~"5.."))) / sum(rate(ts(http.requests.total)))) * 100
     ```

3. **Duration (Latency)**
   - Response time (p50, p95, p99)
   - Time to first byte
   - End-to-end latency
   - **Wavefront Query:**
     ```wql
     percentile(95, ts(http.request.duration, source="*"))
     ```

### USE Metrics (Utilization, Saturation, Errors)

For infrastructure components:

1. **Utilization**
   - CPU usage percentage
   - Memory usage percentage
   - Disk I/O utilization
   - Network bandwidth usage
   - **Wavefront Query Examples:**
     ```wql
     # CPU usage
     100 - avg(ts(cpu.usage.idle, source="*"))
     
     # Memory usage
     (1 - (ts(mem.available, source="*") / ts(mem.total, source="*"))) * 100
     
     # Disk usage
     (1 - (ts(disk.space.available, source="*") / ts(disk.space.total, source="*"))) * 100
     
     # Kubernetes CPU usage
     ts(kubernetes.cpu.usage.total, source="*")
     ```

2. **Saturation**
   - CPU load average
   - Queue depth
   - Thread pool utilization
   - **Wavefront Query Examples:**
     ```wql
     # Load average
     ts(load.avg.1m, source="*")
     
     # Disk I/O time
     rate(ts(disk.io.time, source="*"))
     
     # Kubernetes pod CPU throttling
     ts(kubernetes.cpu.throttled.time, source="*")
     ```

3. **Errors**
   - System errors
   - Failed operations
   - **Wavefront Query:**
     ```wql
     sum(rate(ts(disk.io.errors, source="*")))
     ```

### Business Metrics

1. **User Engagement**
   - Active users (DAU/MAU)
   - Session duration
   - Page views
   - Feature adoption rate

2. **User Experience**
   - Time to interactive
   - First contentful paint
   - Largest contentful paint
   - Cumulative layout shift

3. **Conversion Metrics**
   - Conversion rate
   - Checkout completion rate
   - Feature usage rate

4. **Revenue Metrics**
   - Revenue per user
   - Transaction success rate
   - Payment processing time

### Application-Specific Metrics

1. **Database Metrics**
   - Query latency
   - Connection pool usage
   - Slow queries
   - Replication lag
   - **Wavefront Query Examples:**
     ```wql
     # Database query latency
     percentile(95, ts(db.query.duration, source="*"))
     
     # Database connections
     ts(db.connections.active, source="*")
     
     # Database slow queries
     rate(ts(db.queries.slow, source="*"))
     ```

2. **Cache Metrics**
   - Hit rate
   - Miss rate
   - Eviction rate
   - Memory usage

3. **Message Queue Metrics**
   - Message rate
   - Queue depth
   - Consumer lag
   - Processing time

4. **API Metrics**
   - Endpoint-specific latency
   - Request size
   - Response size
   - Authentication failures

### Golden Signals

Monitor these four golden signals for every service:

1. **Latency** - Time to serve a request
2. **Traffic** - Demand on the system
3. **Errors** - Rate of failed requests
4. **Saturation** - How "full" the service is

### Example Metric Collection Setup

#### Sending Custom Metrics to Wavefront

Wavefront accepts metrics in multiple formats. Here's how to send custom metrics:

**Using Wavefront Proxy (Recommended):**

```bash
# Send metrics via Wavefront proxy on port 2878
echo "http.requests.total 1000 source=app1 method=GET status=200" | \
  nc wavefront-proxy.wavefront.svc.cluster.local 2878
```

**Using Wavefront SDK (Application Integration):**

```python
# Python example
from wavefront_sdk import WavefrontDirectClient
from wavefront_sdk.client import WavefrontClientFactory

# Create client
client_factory = WavefrontClientFactory()
client_factory.add_client("https://YOUR_INSTANCE.wavefront.com", "YOUR_API_TOKEN")
wavefront_sender = client_factory.get_client()

# Send metrics
wavefront_sender.send_metric(
    "http.requests.total",
    1000,
    None,
    {"source": "app1", "method": "GET", "status": "200"}
)
```

**Using Kubernetes Annotations:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    wavefront.com/metrics: "true"
spec:
  template:
    metadata:
      annotations:
        wavefront.com/metrics: "true"
    spec:
      containers:
      - name: app
        image: my-app:latest
```

#### Wavefront Metric Naming Conventions

- Use dots (.) to separate metric name components: `http.requests.total`
- Use lowercase with dots: `cpu.usage.percent`
- Include source tags: `source=app1`, `cluster=k8s-prod`
- Use consistent tag names across related metrics

## Best Practices

### 1. SLI Selection
- Choose SLIs that directly impact user experience
- Limit to 2-5 SLIs per service
- Focus on user-visible metrics
- Avoid internal implementation details

### 2. SLO Targets
- Start conservative and tighten over time
- Use error budgets to balance reliability and feature velocity
- Set different SLOs for different user journeys
- Review and adjust SLOs quarterly

### 3. Chaos Engineering
- Start with non-production environments
- Run experiments during low-traffic periods initially
- Have rollback plans ready
- Document all experiments and outcomes
- Gradually increase experiment scope

### 4. Monitoring
- Monitor at multiple levels (infrastructure, application, business)
- Set up alerting on SLO violations, not just outages
- Use dashboards for visualization, alerts for action
- Review and tune alerts regularly

### 5. Documentation
- Document all SLIs, SLOs, and SLAs
- Keep runbooks updated
- Document chaos experiment procedures
- Maintain metric definitions

## References

- [Mangle GitHub Repository](https://github.com/vmware-archive/mangle)
- [Google SRE Book](https://sre.google/books/)
- [Wavefront Documentation](https://docs.wavefront.com/)
- [Wavefront Query Language Guide](https://docs.wavefront.com/query_language_reference.html)
- [Wavefront Kubernetes Integration](https://docs.wavefront.com/kubernetes.html)
- [Wavefront Alerts Documentation](https://docs.wavefront.com/alerts.html)
- [SRE Workbook - Implementing SLOs](https://sre.google/workbook/slo-implementation/)
- [Wavefront Best Practices](https://docs.wavefront.com/best_practices.html)

## Contributing

Contributions are welcome! Please read our contributing guidelines and submit pull requests for any improvements.

## License

[Specify your license here]
