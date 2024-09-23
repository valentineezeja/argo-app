# Kubernetes Monitoring Stack with Loki, Promtail, Grafana, and Prometheus - Deployed and managed via ArgoCD

## Overview

This project demonstrates the setup of a full monitoring stack for a Kubernetes cluster using **Loki**, **Promtail**, **Grafana**, and **Prometheus**, all managed through **ArgoCD** for GitOps-based deployments. The setup collects and visualizes logs, metrics, and health information from applications running on the Kubernetes cluster.

### Main Components:
1. **Loki (v2.9.3)**: A horizontally scalable log aggregation system that works well with Grafana for querying logs.
2. **Promtail**: An agent that collects logs from Kubernetes pods and ships them to Loki.
3. **Grafana (v11.1.5)**: Provides dashboards and data visualizations, including a pre-configured Loki datasource for logs.
4. **Prometheus (v2.54.1)**: Collects metrics from the Kubernetes cluster and visualizes them in Grafana.

[Insert placeholder for demo video showing ArgoCD, Prometheus, and Grafana dashboards here]

---

## Repository Structure

The repository uses Helm charts to configure and deploy Loki, Promtail, Grafana, and Prometheus. These Helm charts are managed by ArgoCD, ensuring the desired state is always applied to the Kubernetes cluster.

```bash
├── monitoring
│   ├── charts
│   │   ├── loki-stack
│   │   │   ├── grafana
│   │   │   ├── loki
│   │   │   ├── promtail
│   │   ├── prometheus
│   ├── values.yaml
```

Each of the components has its Helm chart and values file to configure the deployment. Promtail, Loki, and Grafana are bundled into a `loki-stack` Helm chart, while Prometheus is managed separately.

---

## Prerequisites

### Kubernetes Setup

Ensure you have a Kubernetes cluster running version **>=1.20**. The setup uses **NodePorts** to expose services for Loki, Grafana, and Prometheus. 

### ArgoCD and GitOps

All configurations are designed to be deployed via **ArgoCD**, allowing for GitOps management of Kubernetes resources. Make sure ArgoCD is installed and configured to manage this repository.

### Helm

The project uses Helm charts to simplify the deployment of each component. Make sure **Helm** is installed locally to run any tests before pushing changes.

---

## Step-by-Step Components

### Loki Setup

**Version:** v2.9.3

Loki is configured to run with the following properties:
- **NodePort:** Exposed on port `31000`
- **Readiness and Liveness Probes:** Loki is monitored with probes to ensure its health.
- **Service Discovery:** Loki is available to other services in the monitoring namespace via the service `loki.monitoring.svc.cluster.local`.

Example Loki config:
```yaml
loki:
  enabled: true
  service:
    type: NodePort
    nodePort: 31000
  readinessProbe:
    httpGet:
      path: /ready
      port: http-metrics
    initialDelaySeconds: 45
  livenessProbe:
    httpGet:
      path: /ready
      port: http-metrics
    initialDelaySeconds: 45
```

### Promtail Configuration

Promtail acts as the log collector, sending log data to Loki.

- **Service Target:** Promtail forwards logs to Loki using the internal Kubernetes service `loki.monitoring.svc.cluster.local:3100`.
- **Log Level:** Configured for `info`.
- **Kubernetes Service Discovery:** Promtail uses Kubernetes annotations and labels to discover logs from pods.

Example Promtail config:
```yaml
promtail:
  enabled: true
  config:
    logLevel: info
    clients:
      - url: http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push
```

### Grafana Setup

Grafana is configured to automatically integrate with Loki for log visualisation.

- **Version:** v11.1.5
- **NodePort:** Exposed on port `30000`
- **Persistence:** Data is persisted to ensure dashboards remain available across restarts.
- **Loki Datasource:** Preconfigured Loki datasource for easy access to logs.

Example Grafana config:
```yaml
grafana:
  service:
    type: NodePort
    nodePort: 30000
  persistence:
    enabled: true
    size: 10Gi
  sidecar:
    datasources:
      enabled: true
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
      - name: Loki
        type: loki
        access: proxy
        url: http://loki.monitoring.svc.cluster.local:3100
```

### Prometheus Setup

Prometheus is deployed to collect and expose Kubernetes metrics for monitoring purposes.

- **Version:** v2.54.1
- **NodePort:** Exposed on port `32001`
- **Service Monitors:** Configured to scrape metrics from Prometheus' own services and Kubernetes nodes.

Example Prometheus config:
```yaml
prometheus:
  server:
    service:
      type: NodePort
      nodePort: 32001
  alertmanager:
    enabled: true
```

---

## Troubleshooting

1. **Grafana not connecting to Loki**: Ensure that Grafana is configured with the correct URL for Loki. You can test the connection by running:

    ```bash
    kubectl exec -it <grafana-pod> -n monitoring -- curl http://loki.monitoring.svc.cluster.local:3100/ready
    ```

2. **Promtail not pushing logs to Loki**: Check if Promtail can reach Loki by running:

    ```bash
    kubectl exec -it <promtail-pod> -n monitoring -- curl http://loki.monitoring.svc.cluster.local:3100/ready
    ```

3. **Prometheus not scraping metrics**: Verify Prometheus ServiceMonitors and PodMonitors are correctly configured to target the appropriate services.

---

## How to Access the Monitoring Dashboards

- **Grafana**: Access the Grafana dashboard at `http://<node-ip>:30000`. The default username is `admin`, and the password can be retrieved via:

    ```bash
    kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
    ```

- **Prometheus**: Access Prometheus metrics at `http://<node-ip>:32001`.

- **Loki**: Logs can be queried directly through Grafana's Explore tab by selecting the preconfigured Loki datasource.

---

## Conclusion

This setup provides a scalable, flexible monitoring and logging stack for your Kubernetes environment. With ArgoCD and Helm charts, infrastructure as code practices are followed to ensure consistent deployments and easy updates.

Feel free to clone, fork, and modify this project to suit your specific monitoring needs!

