# 07 - Monitoring

This module covers observability in AKS using Azure Monitor, Container Insights, and the Prometheus + Grafana stack.

---

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- Enable and use Azure Monitor Container Insights
- Query logs with Log Analytics and KQL
- Deploy Prometheus and Grafana using Helm
- Create custom dashboards for AKS workloads
- Set up alerts for cluster health

---

## 📚 Topics Covered

| File | Topic | Est. Time |
|------|-------|-----------|
| [01-azure-monitor.md](./01-azure-monitor.md) | Container Insights, Log Analytics, KQL queries | 35 min |
| [02-prometheus-grafana.md](./02-prometheus-grafana.md) | Prometheus scraping, Grafana dashboards, alerts | 45 min |

---

## Prerequisites

- Completed [06-security](../06-security/README.md)
- A running AKS cluster

---

## Monitoring Architecture

```
AKS Cluster
  ├── Container Insights (Azure native)
  │   └── Log Analytics Workspace → Azure Monitor → Alerts
  └── Prometheus + Grafana (open source)
      ├── Prometheus → scrapes metrics from pods
      └── Grafana → dashboards, alerting
```

## Key Metrics to Monitor

| Metric | Alert Threshold |
|--------|----------------|
| Node CPU utilization | > 80% sustained |
| Node memory utilization | > 80% sustained |
| Pod restart count | > 5 in 5 minutes |
| Pending pods | > 0 for > 5 minutes |
| PVC usage | > 80% of capacity |
| API server latency | > 1 second p99 |

---

## Next Steps

After completing this module, proceed to [08 - Scaling](../08-scaling/README.md).
