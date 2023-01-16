# ADR Cluster Observability

## Context

We operate multiple k8s clusters on the edge. Workloads on these clusters are managed through GitOps.
We require infrastructure, deployment, and workload observability.

Infrastructure observability refers to the

- cluster-level resource metrics,
- and "real-time" operational status of system services (e.g., _kube-system_).

Deployment observability includes

- when a GitOps configuration reconcile on the cluster,
- if the workload started, and otherwise, what are the issues

Workload observability includes

- application logs of the workloads running on the cluster
- operational status of the workload (i.e., searching logs in case of debugging)
- Application telemetry data (e.g. Metrics, Traces)

We need to consider both **_single cluster_** and **_fleet-level_** observability.

## Decision drivers

What were the drivers to make this decision?

- **Observability (infra, deployment, workload):** Can the solution cover all areas of observability? If not do, we need additional components?

- **Integration in Azure / Ease Deployment:** How well does the solution integrate into existing Azure solutions? How difficult is the setup?

- **Support of temporary disconnect:** Does the solution support occasional disconnects? What is the maximal duration of those disconnects so logs are preserved and later pushed?

- **Observability at Scale:** Consider scale across multiple fleets? How well does the deployment scale? How can we get insights across multiple clusters?

- **Versatile:** Compatible with a lot of open standards to be interoperable and have long-term maturity & support.

## Considered options

### 1. Option: Azure Monitor Container Insights for Azure Arc-enabled K8S clusters.

[Container insights](https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-enable-arc-enabled-clusters) is an Azure Monitor extension. It currently runs on the [Log Analytics Agent for Linux](https://docs.microsoft.com/en-us/azure/azure-monitor/agents/agent-linux?tabs=wrapper-script) as an agent that is collecting container logs and performance metrics (mem, cpu) from the cluster (controller, nodes and containers).
See [Appendix](/Design-Decisions/[-ADR-]-Cluster-Observability/[Option-1]-Appendix) for details (screenshots, queries etc.).

Using Container Insights **requires an Azure Arc-enabled cluster** (see [Supported Environents](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-onboard#supported-configurations). Azure Arc is part of the overall MCfM strategic bet to enable Observability at the Edge. For more information see [MCfM Strategic Bets](/Design-Decisions/[-ADR-]-Cluster-Observability/[Option-1]-MCfM-Strategic-Bets).

Advantages of Azure Monitor Container Insights.

- Supports cluster metrics and container logs out of the box with Azure Arc integration.
- Leverages Azure Monitor Container dashboards that allow monitoring at scale (Cluster Insights, Fleet Overview)
- Deployment and management via Azure (REST API, ARM/Bicep, Azure Portal)
- Supports temporary disconnects (tested for 5 mins and 2 hours)

Disadvantages:

- Does not support multiple pipelines (i.e., only exporting to Log Analytics and Azure Monitor Metrics store)
- Limited Container Log filtering capabilities
- No distributed tracing (requires separate SDK such as App Insights / OpenTelemetry).

### 2. Option: OpenTelemetry Collector

[OpenTelemetry](https://opentelemetry.io/) provides an SDK to expose telemetry using the OpenTelemetry Protocol (OPLT) to external systems one such is the [OpenTelemetry collector](https://opentelemetry.io/docs/collector/).
The OpenTelemetry collectors can be used to create pipelines to receive, process, and export telemetry data of the of types: Metrics, Logs, and Traces.
It currently supports a list of receiver types (see [core](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver) and [extended](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver) ), as well as exporters (see [core](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter) and [extended](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter))

Advantages of OpenTelemetry with the use of the collector.

- Support for multiple protocols and receivers eg. fluentbit, prometheus, kafka, opencensus, etc...
- Support for multiple exports eg. File system, azure monitor exporter, grafana, zipkin etc...
- Support for multiple pipelines eg. Exporting to the cloud as well as serving data locally.
- Single container (collector) connecting to the cloud and no need to share instrumentation keys and other secrets with other applications.
- Single source to handle buffering and statefulness

Disadvantages:

- Early in their roadmap
- No metrics export support for Azure at the moment
- No buffering to cloud support at the moment.

### 1:1 Comparison of Options 1 and 2

|                                                                  | Container Insights for Azure Arc-enabled cluster                                                                     | OpenTelemetryCollector                                                                                                                                                     |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Infrastructure metrics Resources: CPU / RAM**                  | <font color="green">**Yes**</font> – out the box for the cluster (not for the VM)                                    | <font color="green">**Yes**</font>- Has a K8s operator since Oct 2022 see docs [OpenTelemetry Operator for Kubernetes](https://opentelemetry.io/docs/k8s-operator/)</font> |
| **Application metrics** "e.g.Requests/sec, workload health"      | <font color="green">**Yes**</font> - application needs to expose Prometheus endpoint (Prometheus /OpenTelemetry SDK) | <font color="green">**Yes**</font> - application needs to expose Prometheus endpoint (Prometheus /OpenTelemetry SDK)                                                       |
| **Send logs to the cloud**                                       | <font color="green">**Yes**</font> – uses Log Analytics agent for Linux (to be deprecated in 2024).                  | <font color="green">**Yes**</font>                                                                                                                                         |
| **Distributed tracing** "Performance: latency, etc."             | Needs to instrument with App-Insights SDK (or OpenTelemetry SDK)                                                     | Need to instrument with OpenTelemetry SDK _OpenTelemetry SDK supports auto instrumentation for : C#, Java, Python, PHP, and Ruby_                                          |
| **Supported datastores**                                         | Log Analytics Workspace (cloud)                                                                                      | <font color="green">**Multiple options**</font>                                                                                                                            |
| **Send metrics to on-site monitoring platform**                  | <font color="orange">**No**</font>                                                                                   | <font color="green">**Yes**</font> possible                                                                                                                                |
| **View (UI)**                                                    | Azure Monitor / Grafana                                                                                              | Zipkin / Jaeger / Grafana. <font color="orange">Azure Monitor (no support for metrics)</font>                                                                              |
| **Configure notifications**                                      | Azure Monitor                                                                                                        |                                                                                                                                                                            |
| **Dependencies**                                                 | _Arc_                                                                                                                | -                                                                                                                                                                          |
| **Buffering when it goes offline** No loss of observability data | App Insights logs for 48hrs                                                                                          | _No out of the box support_                                                                                                                                                |
| **Effort**                                                       | <font color="green">**Low**</font>                                                                                   | <font color="red">**High**</font>                                                                                                                                          |
| **Maturity / Risk**                                              | <font color="green">**Mature**</font>                                                                                | Still early                                                                                                                                                                |
