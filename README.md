# Setting Up Grafana, Prometheus, and ELK for Monitoring and Logging in AWS EKS

## Table of Contents

- [Introduction](#introduction)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Setting Up AWS EKS](#setting-up-aws-eks)
  - [Creating an EKS Cluster](#creating-an-eks-cluster)
  - [Installing Helm](#installing-helm)
- [Monitoring with Prometheus](#monitoring-with-prometheus)
  - [Setting Up Prometheus on EKS](#setting-up-prometheus-on-eks)
  - [Installing the CloudWatch Observability Add-on](#installing-the-cloudwatch-observability-add-on)
  - [Configuring Prometheus](#configuring-prometheus)
  - [Troubleshooting Prometheus on EKS](#troubleshooting-prometheus-on-eks)
- [Visualization with Grafana](#visualization-with-grafana)
  - [Setting Up Grafana for EKS](#setting-up-grafana-for-eks)
  - [Configuring AWS CloudWatch Data Source](#configuring-aws-cloudwatch-data-source)
  - [AWS Authentication in Grafana](#aws-authentication-in-grafana)
  - [Creating Dashboards for EKS Monitoring](#creating-dashboards-for-eks-monitoring)
- [Logging with ELK Stack](#logging-with-elk-stack)
  - [Setting Up the ELK Stack for EKS](#setting-up-the-elk-stack-for-eks)
  - [Log Collection and Configuration](#log-collection-and-configuration)
- [Alternative: Grafana Loki for Logging](#alternative-grafana-loki-for-logging)
  - [Installing Loki with Helm](#installing-loki-with-helm)
  - [Configuring Promtail for Log Collection](#configuring-promtail-for-log-collection)
- [Best Practices](#best-practices)
  - [Monitoring Best Practices](#monitoring-best-practices)
  - [Logging Best Practices](#logging-best-practices)
  - [Security Considerations](#security-considerations)
- [Summary](#summary)

## Introduction

This project provides a comprehensive guide for setting up monitoring and logging for Amazon Elastic Kubernetes Service (EKS) using industry-standard tools: Prometheus for metrics collection, Grafana for visualization, and the ELK stack (Elasticsearch, Logstash, Kibana) or Grafana Loki for log management.

Proper monitoring and logging are essential for maintaining healthy Kubernetes clusters, troubleshooting issues, and ensuring optimal performance. This setup will help you gain visibility into your EKS infrastructure, applications, and services.

## Architecture Overview

Below is a visual representation of the EKS monitoring pipeline:

![EKS Monitoring Flow Diagram](https://github.com/PraxisForge/AWS_EKS_Monitoring/blob/main/Flowdiagram_01.png)

The monitoring and logging architecture for EKS consists of:

1. **Metrics Collection**: Prometheus collects metrics from your Kubernetes cluster.
2. **Visualization**: Grafana dashboards visualize the collected metrics.
3. **Log Management**: Either the ELK Stack or Grafana Loki collects, processes, and stores logs.
4. **AWS Integration**: Amazon CloudWatch can be integrated for a unified monitoring approach.

## Prerequisites

Before you begin, ensure you have:

- An active AWS account with appropriate permissions
- AWS CLI installed and configured
- `kubectl` command-line tool installed
- `eksctl` command-line tool installed
- Helm package manager installed (v3+)
- Basic knowledge of Kubernetes and EKS

## Setting Up AWS EKS

### Creating an EKS Cluster

If you don't already have an EKS cluster, you can create one using the following command:

```bash
eksctl create cluster \
  --name my-monitoring-cluster \
  --version 1.27 \
  --region us-west-2 \
  --nodegroup-name standard-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed
```

After creating the cluster, configure `kubectl` to use your new cluster:

```bash
aws eks update-kubeconfig --name my-monitoring-cluster --region us-west-2
```

### Installing Helm

Helm is required to deploy most of the monitoring and logging components. Install it on your EKS cluster:

```bash
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

## Monitoring with Prometheus

### Setting Up Prometheus on EKS

You can set up Prometheus on EKS using Helm:

```bash
# Add Prometheus repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace \
  --set server.persistentVolume.storageClass=gp2 \
  --set server.persistentVolume.size=20Gi
```

### Installing the CloudWatch Observability Add-on

AWS provides a CloudWatch Observability add-on for EKS that integrates with Prometheus:

```bash
eks-ctl create addon \
  --name amazon-cloudwatch-observability \
  --cluster my-monitoring-cluster \
  --region us-west-2
```

Alternatively, you can enable it using the AWS Management Console:

1. Open the Amazon EKS console
2. Select your cluster
3. Choose the "Add-ons" tab
4. Click "Get more add-ons"
5. Select "Amazon CloudWatch Observability" and click "Next"
6. Complete the configuration and click "Create"

### Configuring Prometheus

Create a `prometheus.yaml` configuration file for your specific monitoring needs:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

  - job_name: 'kubernetes-nodes'
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
```

Apply this configuration to your Prometheus deployment:

```bash
kubectl create configmap prometheus-config --from-file=prometheus.yaml -n monitoring
kubectl rollout restart deployment/prometheus-server -n monitoring
```

### Troubleshooting Prometheus on EKS

Common issues and their solutions:

1. **No metrics showing up**:
   - Check Prometheus pod status: `kubectl get pods -n monitoring`
   - Check logs: `kubectl logs -n monitoring deployment/prometheus-server`
   - Verify service endpoints: `kubectl get endpoints -n monitoring`

2. **Permission issues**:
   - Ensure proper RBAC is configured for Prometheus to scrape metrics
   - Check if the service account has necessary permissions

## Visualization with Grafana

### Setting Up Grafana for EKS

Install Grafana using Helm:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set persistence.storageClassName=gp2 \
  --set service.type=LoadBalancer
```

Get the Grafana admin password:

```bash
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

### Configuring AWS CloudWatch Data Source

To use AWS CloudWatch as a data source in Grafana:

1. Log in to your Grafana dashboard
2. Go to Configuration > Data Sources
3. Click "Add data source"
4. Select "CloudWatch"
5. Configure the authentication (see next section)
6. Set the default region
7. Click "Save & Test"

### AWS Authentication in Grafana

For AWS authentication in Grafana, you have several options:

1. **IAM Roles for Service Accounts (IRSA)** - Recommended method:

   ```bash
   eksctl create iamserviceaccount \
     --name grafana \
     --namespace monitoring \
     --cluster my-monitoring-cluster \
     --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess \
     --approve
   ```

2. **Access & Secret Keys** - In the Grafana data source configuration:
   - Authentication Provider: Access & Secret Key
   - Access Key ID: Your AWS access key
   - Secret Access Key: Your AWS secret key

3. **AWS SDK Default** - Uses the default AWS SDK credential chain

### Creating Dashboards for EKS Monitoring

After setting up the data sources, import pre-built dashboards for Kubernetes monitoring:

1. Go to "+" > Import in the Grafana UI
2. Enter dashboard ID for Kubernetes (e.g., 3119 for Kubernetes cluster monitoring)
3. Select your Prometheus data source
4. Click Import

You can also create custom dashboards for EKS monitoring with panels for:
- Node CPU/Memory usage
- Pod metrics
- Deployment status
- Network traffic
- Custom application metrics

## Logging with ELK Stack

### Setting Up the ELK Stack for EKS

Install the ELK stack using Helm:

```bash
# Add Elastic Helm repository
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --create-namespace \
  --set replicas=3 \
  --set minimumMasterNodes=2 \
  --set volumeClaimTemplate.storageClassName=gp2 \
  --set volumeClaimTemplate.resources.requests.storage=100Gi

# Install Kibana
helm install kibana elastic/kibana \
  --namespace logging \
  --set service.type=LoadBalancer

# Install Filebeat for log collection
helm install filebeat elastic/filebeat \
  --namespace logging \
  --set daemonset.enabled=true
```

### Log Collection and Configuration

Configure Filebeat to collect container logs:

```yaml
# filebeat-values.yaml
daemonset:
  enabled: true
  
filebeat.inputs:
- type: container
  paths:
    - /var/log/containers/*.log

processors:
  - add_kubernetes_metadata:
      host: ${NODE_NAME}
      matchers:
        - logs_path:
            logs_path: "/var/log/containers/"

output.elasticsearch:
  host: '${ELASTICSEARCH_HOST:elasticsearch-master:9200}'
```

Apply the configuration:

```bash
helm upgrade filebeat elastic/filebeat \
  --namespace logging \
  --values filebeat-values.yaml
```

## Alternative: Grafana Loki for Logging

Grafana Loki offers a lightweight alternative to the ELK stack and integrates seamlessly with Grafana.

### Installing Loki with Helm

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=10Gi \
  --set loki.persistence.storageClassName=gp2
```

### Configuring Promtail for Log Collection

For EKS-specific log collection with Promtail:

```yaml
# promtail-values.yaml
config:
  clients:
    - url: http://loki:3100/loki/api/v1/push
  
  scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
        - role: pod
      pipeline_stages:
        - docker: {}
      relabel_configs:
        - source_labels: [__meta_kubernetes_pod_node_name]
          target_label: __host__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - action: replace
          replacement: $1
          separator: /
          source_labels:
            - __meta_kubernetes_namespace
            - __meta_kubernetes_pod_name
          target_label: job
```

Apply the Promtail configuration:

```bash
helm upgrade loki grafana/loki-stack \
  --namespace monitoring \
  --values promtail-values.yaml
```

Add Loki as a data source in Grafana:
1. Go to Configuration > Data Sources
2. Click "Add data source"
3. Select "Loki"
4. Set URL to "http://loki:3100"
5. Click "Save & Test"

## Best Practices

### Monitoring Best Practices

1. **Define meaningful alerts**: Focus on actionable metrics that indicate real problems.
2. **Use node and pod metrics**: Monitor both infrastructure and application-level metrics.
3. **Implement proper retention policies**: Define how long metrics should be stored based on your needs.
4. **Set up notification channels**: Configure alert notifications via email, Slack, or PagerDuty.
5. **Label your resources properly**: Use consistent labeling for better filtering and querying.

### Logging Best Practices

1. **Structured logging**: Use JSON or other structured formats for easier parsing and querying.
2. **Log levels**: Define and use appropriate log levels (INFO, WARN, ERROR).
3. **Retention policy**: Configure log retention based on compliance and debugging requirements.
4. **Include context**: Ensure logs contain enough context (request IDs, user IDs) for troubleshooting.
5. **Filter sensitive information**: Avoid logging sensitive data like passwords or tokens.

### Security Considerations

1. **RBAC**: Implement proper role-based access control for your monitoring tools.
2. **Network policies**: Restrict network access to monitoring and logging components.
3. **Data encryption**: Encrypt data at rest and in transit.
4. **Authentication**: Secure access to Grafana, Kibana, and other dashboards.
5. **API security**: Use secure methods for AWS authentication (IRSA preferred over access keys).

## Summary

This guide provides a comprehensive approach to monitoring and logging AWS EKS clusters using:

- **Prometheus** for metrics collection
- **Grafana** for visualization
- **ELK Stack** or **Grafana Loki** for log management

By following these steps and best practices, you'll have robust observability for your Kubernetes workloads, allowing you to detect issues early, troubleshoot efficiently, and ensure optimal performance of your applications.

Remember to regularly review and update your monitoring and logging setup as your infrastructure evolves and new best practices emerge.
