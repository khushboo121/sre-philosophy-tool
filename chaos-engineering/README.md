# SRE Philosophy Tool

A comprehensive toolkit for implementing Site Reliability Engineering (SRE) practices, including chaos engineering with VMware Mangle, SLI/SLO/SLA monitoring with Wavefront, and user metrics measurement.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Mangle VMware Chaos Engineering in Kubernetes](#mangle-vmware-chaos-engineering-in-kubernetes)
- [Fault Injection Examples](#fault-injection-examples)
- [Resiliency Score with Mangle](#resiliency-score-with-mangle)
- [Creating Dashboards for SLI, SLO, and SLA](#creating-dashboards-for-sli-slo-and-sla)
- [User Metrics to Measure](#user-metrics-to-measure)
- [Best Practices](#best-practices)
- [References](#references)

## Overview

This toolkit helps organizations implement SRE principles by:
- Running controlled chaos experiments using VMware Mangle
- Calculating and tracking Resiliency Scores for services
- Monitoring Service Level Indicators (SLIs) with Wavefront
- Defining and tracking Service Level Objectives (SLOs)
- Measuring Service Level Agreements (SLAs)
- Collecting and analyzing user-centric metrics

## Prerequisites

- Kubernetes cluster (v1.19+)
- `kubectl` configured to access your cluster
- `helm` v3.x installed
- Wavefront account and access (VMware Tanzu Observability)
- Wavefront API token for metric collection
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

# Install Mangle using Kubernetes manifests
kubectl create namespace mangle
kubectl apply -f mangle-support/kubernetes/ -n mangle
```

For Docker-based deployment:

```bash
# Pull Mangle Docker image
docker pull vmware/mangle:latest

# Run Mangle container
docker run -d -p 8080:8080 \
  -e WAVEFRONT_URL=YOUR_WAVEFRONT_URL \
  -e WAVEFRONT_TOKEN=YOUR_WAVEFRONT_API_TOKEN \
  --name mangle vmware/mangle:latest
```

### 6. Verify Installation

```bash
# Check Mangle pods
kubectl get pods -n mangle

# Check Mangle services
kubectl get svc -n mangle

# Check Wavefront proxy
kubectl get pods -n wavefront

# Verify Mangle API is accessible
kubectl port-forward -n mangle svc/mangle-service 8080:8080
curl http://localhost:8080/mangle/api/rest/v1/health
```

## Mangle VMware Chaos Engineering in Kubernetes

### Overview

Mangle enables you to inject controlled failures into your Kubernetes infrastructure and applications to test resilience and validate SLOs under adverse conditions. Mangle supports various fault types including CPU, Memory, Disk, Network, and VM state faults.

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

#### 2. Configure Wavefront Integration in Mangle

Mangle needs Wavefront credentials to push Resiliency Score metrics:

```bash
# Create ConfigMap for Wavefront configuration
kubectl create configmap mangle-wavefront-config \
  --from-literal=wavefront.url=YOUR_WAVEFRONT_URL \
  -n mangle

# Create Secret for Wavefront token
kubectl create secret generic mangle-wavefront-secret \
  --from-literal=token=YOUR_WAVEFRONT_API_TOKEN \
  -n mangle
```

#### 3. Access Mangle API

```bash
# Port forward to Mangle service
kubectl port-forward -n mangle svc/mangle-service 8080:8080

# Or expose via LoadBalancer/Ingress
kubectl patch svc mangle-service -n mangle -p '{"spec":{"type":"LoadBalancer"}}'

# Access Mangle UI (if available)
# Navigate to http://localhost:8080/mangle in your browser
```

### Adding Endpoints

Before running faults, you need to add endpoints (targets) to Mangle:

```bash
# Add Kubernetes endpoint
curl -X POST http://localhost:8080/mangle/api/rest/v1/endpoints \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "endpointName": "k8s-cluster-1",
    "endpointType": "K8S_CLUSTER",
    "credentials": {
      "kubeConfig": "base64-encoded-kubeconfig"
    },
    "tags": {
      "environment": "production",
      "cluster": "primary"
    }
  }'

# Add Docker endpoint
curl -X POST http://localhost:8080/mangle/api/rest/v1/endpoints \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "endpointName": "docker-host-1",
    "endpointType": "DOCKER",
    "credentials": {
      "host": "docker.example.com",
      "port": 2376,
      "certFilePath": "/path/to/cert",
      "keyFilePath": "/path/to/key",
      "caFilePath": "/path/to/ca"
    }
  }'

# Add vCenter endpoint
curl -X POST http://localhost:8080/mangle/api/rest/v1/endpoints \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "endpointName": "vcenter-prod",
    "endpointType": "VCENTER",
    "credentials": {
      "hostname": "vcenter.example.com",
      "username": "administrator@vsphere.local",
      "password": "password",
      "port": 443
    }
  }'
```

## Fault Injection Examples

### CPU Faults

#### CPU Stress on Kubernetes Pods

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "faultName": "cpu-stress-k8s",
    "faultSpec": {
      "fault": "CPU",
      "endpointName": "k8s-cluster-1",
      "cpuLoad": 80,
      "timeoutInMilliseconds": 300000,
      "injectionHomeDir": "/tmp",
      "schedule": {
        "cronExpression": "0 0 2 * * ?"
      }
    }
  }'
```

#### CPU Stress on Docker Containers

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "faultName": "cpu-stress-docker",
    "faultSpec": {
      "fault": "CPU",
      "endpointName": "docker-host-1",
      "containerName": "my-app-container",
      "cpuLoad": 90,
      "timeoutInMilliseconds": 600000
    }
  }'
```

### Memory Faults

#### Memory Stress

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "faultName": "memory-stress",
    "faultSpec": {
      "fault": "MEMORY",
      "endpointName": "k8s-cluster-1",
      "memoryLoad": 70,
      "timeoutInMilliseconds": 300000
    }
  }'
```

### Disk Faults

#### Disk IO Stress

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "faultName": "disk-io-stress",
    "faultSpec": {
      "fault": "DISKIO",
      "endpointName": "k8s-cluster-1",
      "diskIO": {
        "blockSize": 1024,
        "writeBytesPerSec": 10485760,
        "readBytesPerSec": 10485760
      },
      "timeoutInMilliseconds": 300000
    }
  }'
```

#### Disk Space Fault

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "faultName": "disk-space-fault",
    "faultSpec": {
      "fault": "DISKSPACE",
      "endpointName": "k8s-cluster-1",
      "diskSpace": {
        "size": 1073741824,
        "path": "/tmp"
      },
      "timeoutInMilliseconds": 300000
    }
  }'
```

### Network Faults

#### Network Latency

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "faultName": "network-latency",
    "faultSpec": {
      "fault": "NETWORK",
      "endpointName": "k8s-cluster-1",
      "networkFault": {
        "latency": 500,
        "jitter": 50
      },
      "timeoutInMilliseconds": 300000
    }
  }'
```

#### Network Packet Loss

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "faultName": "network-packet-loss",
    "faultSpec": {
      "fault": "NETWORK",
      "endpointName": "k8s-cluster-1",
      "networkFault": {
        "packetLossPercentage": 10
      },
      "timeoutInMilliseconds": 300000
    }
  }'
```

### VMware VM Faults

#### VM Power Off

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "faultName": "vm-power-off",
    "faultSpec": {
      "fault": "VM_STATE",
      "endpointName": "vcenter-prod",
      "vmName": "k8s-worker-node-1",
      "vmState": "POWER_OFF",
      "timeoutInMilliseconds": 600000
    }
  }'
```

#### VM Disk Removal

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "faultName": "vm-disk-remove",
    "faultSpec": {
      "fault": "VM_DISK",
      "endpointName": "vcenter-prod",
      "vmName": "k8s-worker-node-1",
      "diskId": "disk-1",
      "timeoutInMilliseconds": 300000
    }
  }'
```

### Kubernetes-Specific Faults

#### Pod Kill

```bash
curl -X POST http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "faultName": "pod-kill",
    "faultSpec": {
      "fault": "K8S_POD",
      "endpointName": "k8s-cluster-1",
      "podName": "my-app-pod",
      "namespace": "default",
      "timeoutInMilliseconds": 300000
    }
  }'
```

### Monitoring Chaos Experiments

While chaos experiments run, monitor your application metrics in Wavefront:

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
curl -X GET http://localhost:8080/mangle/api/rest/v1/faults \
  -H "Authorization: Bearer YOUR_TOKEN"

# Get fault details
curl -X GET http://localhost:8080/mangle/api/rest/v1/faults/{faultId} \
  -H "Authorization: Bearer YOUR_TOKEN"

# Remediate a specific fault
curl -X DELETE http://localhost:8080/mangle/api/rest/v1/faults/{faultId} \
  -H "Authorization: Bearer YOUR_TOKEN"
```

## Resiliency Score with Mangle

### Overview

Mangle's Resiliency Score feature enables you to quantify the resiliency or fault tolerance capacity of your application or system under test. The score is calculated by retrieving application metrics from Wavefront during fault injection and is pushed back to Wavefront as a metric that can be monitored and tracked over time.

**Reference:** [Mangle Resiliency Score Documentation](https://github.com/vmware/mangle/blob/master/docs/sre-developers-and-users/resiliency-score.md)

### Prerequisites

Before using Resiliency Score, ensure:
1. Wavefront is configured in Mangle (see [Configuration](#configuration) section)
2. Wavefront integration is enabled in Mangle admin settings
3. You have queries configured in your monitoring system (Wavefront) that evaluate to Boolean values (0 or 1)

### Adding Queries

Queries define the metrics that will be evaluated to calculate the resiliency score:

```bash
# Add a query via Mangle UI or API
curl -X POST http://localhost:8080/mangle/api/rest/v1/resiliency-score/queries \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "queryName": "availability-check",
    "weightage": 40,
    "queryCondition": "ts(service.availability) >= 0.99"
  }'
```

**Query Requirements:**
- Query name must be unique
- Weightage is a percentage value (0-100) indicating the impact on overall resiliency score
- Query condition must evaluate to a Boolean value (0 or 1)
- It's recommended to use existing alert queries from Wavefront to avoid errors

### Adding Services

Services group queries together to calculate a holistic resiliency score:

```bash
# Add a service
curl -X POST http://localhost:8080/mangle/api/rest/v1/resiliency-score/services \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "serviceName": "payment-service",
    "queries": [
      "availability-check",
      "latency-check",
      "error-rate-check"
    ],
    "tags": {
      "service": "payment",
      "environment": "production"
    }
  }'
```

**Service Configuration:**
- Service name uniquely identifies the service
- Select queries that together measure service resilience
- Tags help Mangle filter fault events specific to the service

### Calculating Resiliency Score

```bash
# Calculate resiliency score for a service
curl -X POST http://localhost:8080/mangle/api/rest/v1/resiliency-score/calculate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "serviceName": "payment-service",
    "schedule": {
      "cronExpression": "0 */30 * * * ?"
    }
  }'
```

**Schedule Options:**
- Optional: Only required if score needs to be calculated at regular intervals
- Uses cron expression format
- If not provided, score is calculated immediately

### Viewing Resiliency Scores

Resiliency scores are pushed to Wavefront as metrics. Query them in Wavefront:

```wql
# Query resiliency score metric
ts(mangle.resiliency.score, service="payment-service")

# Query with tags
ts(mangle.resiliency.score, service="payment-service", environment="production")
```

### Resiliency Score Dashboard in Wavefront

Create a dashboard in Wavefront to track resiliency scores over time:

1. Go to **Dashboards** → **Create Dashboard**
2. Add a chart with query: `ts(mangle.resiliency.score, service="*")`
3. Group by service name to see scores for all services
4. Set up alerts when resiliency score drops below threshold

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
- Use Resiliency Score to track improvements over time

### 4. Monitoring
- Monitor at multiple levels (infrastructure, application, business)
- Set up alerting on SLO violations, not just outages
- Use dashboards for visualization, alerts for action
- Review and tune alerts regularly
- Integrate Mangle Resiliency Scores into your monitoring dashboards

### 5. Documentation
- Document all SLIs, SLOs, and SLAs
- Keep runbooks updated
- Document chaos experiment procedures
- Maintain metric definitions
- Document Resiliency Score queries and their purpose

## References

- [Mangle GitHub Repository](https://github.com/vmware-archive/mangle)
- [Mangle Users Guide](https://github.com/vmware-archive/mangle/tree/v3.0.0/docs/sre-developers-and-users)
- [Mangle Resiliency Score Documentation](https://github.com/vmware/mangle/blob/master/docs/sre-developers-and-users/resiliency-score.md)
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

