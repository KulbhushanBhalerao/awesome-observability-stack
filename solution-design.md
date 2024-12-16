# Solution Design Document: Open Source Observability Stack for Kubernetes Clusters

| Name | Date |   Version   | Remarks                                                   |
|:---------------------|----------|:-----------:|:--------------------------------------------------------------|
| Kulbhushan Bhalerao             | 16/12/2024 | v1  | Initial draft |


## 1. **Overview**

This document outlines the design and implementation plan for setting up a scalable, highly available, and cost-efficient observability stack for monitoring multiple Kubernetes clusters. The stack will provide separation of data based on use cases, support derived calculations, ensure actionable alerting, and enable integration with CI/CD pipelines for infrastructure as code (IaC).

---

## 2. **Requirements**

### 2.1 **Key Functional Requirements**

- Collect metrics, logs, and traces from multiple Kubernetes clusters.
- Separate data into categories:
  - **Technical Infrastructure Data**: Metrics/logs from Kubernetes nodes, control plane components, and cluster-level resources.
  - **Application Health Data**: Service-level metrics and application logs.
  - **Functional Data**: Business KPIs, user events, and application-specific use cases.
  - **Derived Data**: Data obtained via aggregations or calculations.
- Compact and aggregate historical metrics to reduce storage costs.
- Move data to cold storage daily.
- Maintain high availability and scalability.
- Enable monitoring and alerting with actionable insights.
- Automate the observability stack using IaC and support data import/export to higher environments.

### 2.2 **Non-Functional Requirements**

- **Cost-efficiency**: Use open-source tools where possible.
- **Performance**: Ensure the stack can handle high-throughput metrics and logs.
- **Security**: Role-based access control (RBAC) for monitoring data.
- **Ease of Maintenance**: Automated deployments and upgrades.
- **Extensibility**: Support for future enhancements such as AI-based anomaly detection.

---

## 3. **Proposed Architecture**

### 3.1 **Kubernetes Cluster Setup**

- **Central Observability Cluster**: Deploy a dedicated Kubernetes cluster to aggregate observability data from all clusters.
- **Local Observability Agents**: Each Kubernetes cluster will run Prometheus, Loki, and OpenTelemetry agents for local data collection and forwarding.
- **Namespace Organization**:
  - `observability`: For shared observability components (e.g., Mimir, Loki, Jaeger).
  - `monitoring`: For monitoring technical infrastructure (e.g., nodes, control plane).
  - `application-health`: For service and application monitoring.
  - `functional-data`: For use case-specific metrics and derived data.
- **Cluster Federation**: Use Prometheus Federation for cross-cluster data querying.

---

### 3.2 **Observability Stack**

#### 3.2.1 **Metrics**

- **Prometheus**:
  - Deploy Prometheus in each cluster to scrape metrics from Kubernetes components and applications.
  - Use Kubernetes ServiceMonitors to configure scraping targets.
  - Retention: Store metrics locally for 7 days.
- **Mimir**:
  - Replace Thanos with Mimir for long-term storage, query federation, and metric aggregation.
  - Benefits:
    - Better scalability and multi-tenancy for metrics across clusters.
    - Advanced query optimization and parallelization for PromQL queries.
    - Simplified architecture compared to managing Thanos components.
  - Retention: Define custom retention policies for raw and aggregated metrics to balance cost and performance.

#### 3.2.2 **Logs**

- **Loki**:
  - Deploy `promtail` agents in each cluster to collect and forward logs.
  - Centralize log storage in a multi-tenant Loki setup.
  - Integrate with S3-compatible object storage for long-term log retention.
  - **Hybrid Approach**:
    - Use Loki for efficient Kubernetes-native log aggregation and querying.
    - Forward selected logs to **Elasticsearch** for advanced filtering, tokenization, and complex alerting workflows.

#### 3.2.3 **Traces**

- **Jaeger**:
  - Deploy Jaeger for distributed tracing.
  - Use Elasticsearch or ClickHouse as the backend storage for Jaeger.
  - **Elasticsearch**:
    - Full-text search for trace attributes and logs.
    - Integrates well with existing ELK stack setups.
  - **ClickHouse**:
    - Optimized for analytical queries, reducing storage and compute costs.
    - Lower maintenance overhead compared to Elasticsearch for high-throughput environments.
  - Retention: Configure policies to store traces locally for 7 days and archive older traces to cold storage.

#### 3.2.4 **Cold Storage**

- Use **S3-compatible storage** (e.g., MinIO, AWS S3, or GCP Cloud Storage) to store:
  - Aggregated metrics older than 7 days.
  - Logs older than 1 day.
  - Traces older than 7 days.

#### 3.2.5 **Visualization**

- **Grafana**:
  - Centralized dashboards for all observability data.
  - Create folders for different data categories (technical, application, functional, derived).
  - Integrate with Prometheus, Loki, Jaeger, and Mimir for seamless visualization.

#### 3.2.6 **Alerting**

- **Alertmanager**:
  - Define rules in Prometheus for threshold-based alerts.
  - Create alert groups for different severities and escalation policies.
  - Integrate with notification channels (Slack, PagerDuty, email).
- **Runbooks**: Attach resolution playbooks to critical alerts for faster triage.

---

### 3.3 **Data Separation and Management**

- Use **Labels and Annotations**:
  - Apply consistent labeling to categorize metrics, logs, and traces.
  - Example labels: `cluster`, `namespace`, `app`, `environment`.
- **Derived Data Pipelines**:
  - Implement custom aggregations and calculations using PromQL or Mimir Query.
  - Store derived metrics in separate namespaces or Grafana folders.

---

## 4. **Implementation Plan**

### 4.1 **Infrastructure as Code**

- Use **Terraform** for infrastructure provisioning:
  - Create Kubernetes clusters with managed or self-hosted solutions (e.g., GKE, EKS, AKS, Kubeadm).
  - Provision storage backends (e.g., S3 buckets, Elasticsearch).
- Use **Helm** for deploying observability components:
  - Prometheus Operator, Loki Operator, and Jaeger Helm charts.
- Integrate CI/CD pipelines for automated deployments and upgrades.

### 4.2 **Monitoring Agents**

- Deploy Prometheus `node-exporter` and `kube-state-metrics` for technical infrastructure monitoring.
- Deploy `promtail` as a DaemonSet for log collection.
- Configure OpenTelemetry Collector as a Deployment for trace ingestion.

### 4.3 **Data Retention and Compaction**

- Set Mimir retention policies to balance storage usage and query performance.
- Use Mimir's compaction capabilities for efficient storage of historical data.
- Schedule daily jobs using Kubernetes CronJobs to move older data to cold storage.

### 4.4 **Alerting and Scheduling**

- Define Prometheus alert rules for:
  - Resource thresholds (e.g., CPU, memory, disk usage).
  - Application errors (e.g., HTTP 5xx rates, latency).
  - Custom business KPIs.
- Use **Argo Workflows** to automate operational tasks like scaling or cleanup jobs.

### 4.5 **Export and Import**

- **Grafana Dashboards**:
  - Export dashboards as JSON for version control.
  - Import dashboards into higher environments via CI/CD.
- **S3 Data Replication**:
  - Automate replication of S3 buckets for disaster recovery.
  - Use versioning for stored data to maintain consistency.

---

## 5. **Operational Considerations**

### 5.1 **High Availability**

- Deploy observability components with 3 replicas to ensure resilience.
- Use Kubernetes Ingress with external load balancers for accessing Grafana and APIs.
- Implement multi-region S3 storage for durability.

### 5.2 **Scalability**

- Scale Mimir horizontally to handle increased metrics volume.
- Use HPAs to scale components like Loki and Jaeger based on workload.
- Implement lifecycle policies for S3 storage to manage cost.

### 5.3 **Security**

- Use **RBAC** to restrict access to observability namespaces.
- Configure network policies to limit inter-component communication.
- Use TLS for all communication (e.g., Prometheus to Mimir, Grafana to Loki).
- Encrypt data at rest and in transit for all storage backends.

---

## 6. **Outcome and Benefits**

- **Centralized Observability**: Unified monitoring across multiple clusters.
- **Cost Efficiency**: Optimized storage with aggregation and cold storage policies.
- **Actionable Insights**: Clear, categorized dashboards and alerts.
- **Portability**: IaC ensures consistency across environments.
- **Scalability**: Seamless scaling for growing workloads.

---

## 7. **Tools Used and Justifications**

### Metrics

- **Prometheus**: Industry-standard for Kubernetes metrics collection, easy integration with Kubernetes.
- **Mimir**: Provides scalable, multi-tenant long-term metrics storage and query optimization, reducing complexity and enhancing performance compared to Thanos.

### Logs

- **Loki**: Lightweight, Kubernetes-native log aggregation with Prometheus-like querying.
- **Elasticsearch**: Tokenization and advanced log filtering for creating alerts and business insights. Used selectively for critical logs requiring detailed analysis.

### Traces

- **Jaeger**: Proven solution for distributed tracing, integrates well with OpenTelemetry. Supports Elasticsearch and ClickHouse as backends based on specific requirements (e.g., scalability, cost).

### Visualization and Alerting

- **Grafana**: Best-in-class visualization tool with wide integration support.
- **Alertmanager**: Handles alert routing and deduplication effectively.

### Cold Storage

- **S3-Compatible Storage**: Cost-effective, durable, and widely supported.

### Automation

- **Terraform**: Industry-standard IaC tool for infrastructure provisioning.
- **Helm**: Simplifies Kubernetes application deployments.
- **Argo Workflows**: Enables workflow automation in Kubernetes.

---

## 8. **Future Enhancements**

- Integrate AI/ML-based anomaly detection for proactive monitoring.
- Add user authentication via Single Sign-On (SSO).
- Explore Grafana Faro for frontend application observability.

---

## 9. **Appendix**

### 9.1 **Jaeger with Elasticsearch Backend**

Jaeger supports Elasticsearch as a backend for storing trace data. This integration provides:
- **Full-Text Search**: Enables advanced filtering and searching of trace attributes.
- **Seamless Integration**: Works well with existing ELK stacks if Elasticsearch is already used for logs.
- **Scalability**: Elasticsearch can scale horizontally to handle high trace ingestion rates.

**Implementation Notes:**
- Use Helm to deploy Jaeger with Elasticsearch configuration.
- Configure index templates in Elasticsearch to manage retention and optimize storage.

### 9.2 **Loki vs. Elasticsearch for Logs**

#### Loki:
- **Advantages**:
  - Lightweight and cost-efficient for Kubernetes-native log aggregation.
  - Uses labels for efficient querying without indexing every log line.
  - Easily integrates with Prometheus and Grafana.
- **Limitations**:
  - Limited capabilities for tokenization and advanced search.
  - Not ideal for use cases requiring full-text search or complex log analysis.

#### Elasticsearch:
- **Advantages**:
  - Supports full-text search, tokenization, and log transformation.
  - Enables advanced filtering and alerting based on log content.
- **Use Case**:
  - Forward critical logs from Loki to Elasticsearch for advanced analysis and alerting.

**Implementation Notes:**
- Configure Loki to forward logs selectively to Elasticsearch using log streams.
- Use Elasticsearch's alerting engine for complex rule-based alerts.

---

