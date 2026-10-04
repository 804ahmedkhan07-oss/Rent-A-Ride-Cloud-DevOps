# Task 18 — Kubernetes Monitoring with CloudWatch, Prometheus & Grafana

## Objective

Implemented a monitoring and observability stack for the Rent-A-Ride Kubernetes application using:

* AWS CloudWatch
* CloudWatch Agent
* Amazon SNS
* Prometheus
* Grafana
* Kubernetes / Kind

The monitoring environment was deployed on a dedicated AWS EC2 instance without modifying the previously completed Rent-A-Ride infrastructure.

---

## Architecture

```text
                    AWS EC2
                       │
                 Kind Kubernetes
                       │
              ┌────────┼────────┐
              │        │        │
           Frontend  Backend  MongoDB
              │        │        │
              └────────┼────────┘
                       │
                  ┌────▼─────┐
                  │Prometheus│
                  └────┬─────┘
                       │
                  ┌────▼─────┐
                  │  Grafana │
                  └──────────┘

EC2/System Metrics
       │
       ▼
 CloudWatch
       │
     Alarm
       │
      SNS
       │
     Email
```

---

## Environment

| Component  | Configuration  |
| ---------- | -------------- |
| AWS        | EC2            |
| Instance   | m7i-flex.large |
| OS         | Ubuntu         |
| Kubernetes | Kind v1.34.0   |
| Kind       | v0.30.0        |
| Docker     | 29.1.3         |
| kubectl    | v1.36.4        |
| Helm       | v4.3.0         |

---

## Kubernetes Application

Rent-A-Ride was deployed through the existing Helm chart:

```bash
helm install rent-a-ride ./rent-a-ride-chart \
  -n rent-a-ride \
  --create-namespace
```

Deployment verification:

```text
backend      1/1 Running
frontend     2/2 Running
mongodb      1/1 Running
```

Services:

```text
backend-service     3000
frontend-service    8080
mongodb-service     27017
```

Backend HPA:

```text
Min replicas: 1
Max replicas: 6
CPU target: 50%
```

---

## CloudWatch Monitoring

Installed and configured the **Amazon CloudWatch Agent** with an IAM role using `CloudWatchAgentServerPolicy`.

Custom namespace:

```text
Task18/EC2
```

Collected metrics:

* CPU
* Memory
* Disk
* Disk I/O
* Network

CloudWatch Agent configuration was validated successfully and the service was verified as running.

---

## SNS Alerting

Created SNS topic:

```text
task18-monitoring-alerts
```

An email subscription was configured and confirmed.

Alert flow:

```text
CloudWatch Alarm → SNS → Email
```

---

## CloudWatch Alarms

### High CPU

```text
Alarm: task18-high-cpu
Metric: CPUUtilization
Threshold: >80%
Period: 1 minute
Action: SNS
State: OK
```

### High Memory

```text
Alarm: task18-high-memory
Metric: mem_used_percent
Threshold: >80%
Period: 1 minute
Action: SNS
State: OK
```

Both alarms were verified successfully.

---

## Prometheus

Prometheus was deployed using `kube-prometheus-stack`.

The stack provides:

* Prometheus
* Alertmanager
* Grafana
* Node Exporter
* kube-state-metrics
* Prometheus Operator

Verified Prometheus targets included Kubernetes API, kubelet, cAdvisor, node-exporter, kube-state-metrics, Grafana, Alertmanager and Prometheus.

Prometheus successfully collected:

* Container CPU metrics
* Container memory metrics
* Pod status
* Kubernetes/node metrics

> **Screenshot 01 — Prometheus Targets**
> *![Prometheus Targets](./https://github.com/804ahmedkhan07-oss/Rent-A-Ride-Cloud-DevOps/blob/feature/task18-monitoring/docs/Screenshot%202026-10-03%2010.24.07%20PM.png).*

---

## Application Metrics

The backend `/metrics` endpoint was tested.

Result:

```text
Cannot GET /metrics
```

Therefore, no fake application metrics were created.

The current application does not expose a Prometheus metrics endpoint. Application-level request rate, latency and error-rate metrics would require future application instrumentation.

---

## Grafana

Grafana was configured with Prometheus as its datasource.

A dedicated Rent-A-Ride dashboard was created with:

* **CPU Usage** — Time Series
* **Running Pods** — Stat
* **Memory Usage** — Gauge
* **Pod Memory Usage** — Gauges

Verified dashboard values included:

```text
Running Pods: 4
Memory Usage: ~266 MiB
```

> **Screenshot 02 — Grafana Rent-A-Ride Dashboard**
> *[Grafana Rent-A-Ride Dashboard](./docs/Screenshot 2026-10-03 10.23.17 PM.png).*

---

## Testing & Verification

### Completed

* EC2 monitoring environment
* Kubernetes cluster
* Helm deployment
* Application pods/services
* CloudWatch Agent
* CloudWatch metrics
* SNS email subscription
* CPU alarm
* Memory alarm
* Prometheus installation
* Prometheus targets
* Kubernetes/container metrics
* Grafana datasource
* Grafana dashboard

### Not Executed

Active fault/load testing was **not executed** in this phase:

* Artificial CPU stress
* Artificial memory stress
* Deliberate 5xx/error generation
* End-to-end alarm → SNS trigger test

These were not marked as successful because they were not actually performed.

---

## Troubleshooting

### Docker Permission

Resolved Docker access using the user Docker group.

### CloudWatch Agent

Initial Snap installation was unsuccessful; the official Ubuntu `.deb` package was used successfully.

### Application Deployment

The application was ultimately deployed through Helm after standalone manifest deployment exposed missing configuration dependencies.

### Prometheus Application Metrics

`/metrics` returned `Cannot GET /metrics`, confirming that the backend currently lacks Prometheus application instrumentation.

---

## Final Result

Task18 successfully established a dedicated Kubernetes observability environment:

```text
AWS EC2
   │
   ├── CloudWatch → Infrastructure Metrics → Alarms → SNS → Email
   │
   └── Kind Kubernetes
          │
          ├── Rent-A-Ride
          │
          ├── Prometheus → Kubernetes/Container Metrics
          │
          └── Grafana → Monitoring Dashboard
```

The setup provides working infrastructure, Kubernetes, container monitoring and visualization, while application-specific metrics remain a documented future enhancement.

**Task18: Completed and documented.**

