# Cluster Observability

## Problem

Kubernetes clusters run multiple workloads and host by design distributed systems. Hence, having a good observability suite for these clusters is of the essence, especially when it comes to diagnostics and troubleshooting.

What we understand under the observability of a Kubernetes cluster can be split into two main parts:

Infrastructure observability:

- Cluster-level resource metrics,
- Operational status of kube-system

Workload observability

- Application health and readiness
- Application telemetry data (e.g. Metrics, Traces, Logs)

There are multiple ways to extract the needed information from a cluster and get an overview of the cluster's health from the outside.
This article aims to showcase some of these options and include their up and downsides. 

- Collector-based approaches
  - Azure Container Insights
  - Opentelemetry
- Individual instrumentation-based approaches
  - SDK instrumentation

## Solutions

### Collector-based approaches

#### Azure Monitor Container Insights for Azure Arc-enabled K8S clusters.

[Container insights](https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-enable-arc-enabled-clusters) is an Azure Monitor extension. It currently runs on the [Log Analytics Agent for Linux](https://docs.microsoft.com/en-us/azure/azure-monitor/agents/agent-linux?tabs=wrapper-script) as an agent that is collecting container logs and performance metrics (mem, cpu) from the cluster (controller, nodes and containers).
See [Appendix](/Design-Decisions/[-ADR-]-Cluster-Observability/[Option-1]-Appendix) for details (screenshots, queries etc.).

Using Container Insights **requires an Azure Arc-enabled cluster** (see [Supported Environents](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-onboard#supported-configurations). Azure Arc is part of the overall MCfM strategic bet to enable Observability at the Edge. For more information see [MCfM Strategic Bets](/Design-Decisions/[-ADR-]-Cluster-Observability/[Option-1]-MCfM-Strategic-Bets).

Advantages of Azure Monitor Container Insights.

- Supports cluster metrics and container logs out of the box with Azure Arc integration.
- Leverages Azure Monitor Container dashboards that allow monitoring at scale (Cluster Insights, Fleet Overview)
- Deployment and management via Azure (REST API, ARM/Bicep, Azure Portal)
- Supports temporary disconnects

Disadvantages:

- Does not support multiple pipelines (i.e., only exporting to Log Analytics and Azure Monitor Metrics store)
- Limited Container Log filtering capabilities
- No distributed tracing (requires separate SDK such as App Insights / OpenTelemetry).

#### OpenTelemetry Collector

[OpenTelemetry](https://opentelemetry.io/) provides an SDK to expose telemetry using the OpenTelemetry Protocol (OPLT) to external systems, one of such is the [OpenTelemetry collector](https://opentelemetry.io/docs/collector/).
The OpenTelemetry collectors can be used to create pipelines to receive, process, and export telemetry data of the of types: Metrics, Logs, and Traces.
It currently supports a list of receiver types (see [core](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver) and [extended](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver) ), as well as exporters (see [core](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter) and [extended](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter))

Advantages of OpenTelemetry with the use of the collector.

- Support for multiple protocols and receivers eg. fluentbit, prometheus, kafka, opencensus, etc...
- Support for multiple exports eg. File system, azure monitor exporter, grafana, zipkin etc...
- Support for multiple pipelines eg. Export to cloud as well as serve data locally.
- Single container (collector) connecting to the cloud and no need to share instrumentation keys and other secrets with other applications.
- Single source to handle buffering and statefulness

Disadvantages:

- Early in their roadmap
- No metrics export support for Azure at the moment
- No buffering to cloud support at the moment.

#### Comparison of the collector bases approaches

|                                                                  | Container Insights for Azure Arc-enabled cluster                                                                    | OpenTelemetryCollector                                                                                                                                                     |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Infrastructure metrics Resources: CPU / RAM**                 | <font color="green">**Yes**</font> – out the box for the cluster (not for the VM)                                   | <font color="green">**Yes**</font>- Has a K8s operator since Oct 2022 see docs [OpenTelemetry Operator for Kubernetes](https://opentelemetry.io/docs/k8s-operator/)</font> |
| **Application metrics** "e.g.Requests/sec, workload health"      | <font color="green">**Yes**</font> - application needs to expose Prometheus endpoint (Prometheus /OpenTelemetry SDK) | <font color="green">**Yes**</font> - application needs to expose Prometheus endpoint (Prometheus /OpenTelemetry SDK)                                                       |
| **Send logs to the cloud**                                       | <font color="green">**Yes**</font> – uses Log Analytics agent for Linux (to be deprecated in 2024).                 | <font color="green">**Yes**</font>                                                                                                                                         |
| **Distributed tracing** "Performance: latency, etc."             | Needs to instrument with App-Insights SDK (or OpenTelemetry SDK)                                                    | Need to instrument with OpenTelemetry SDK _OpenTelemetry SDK suports autoinstrumentation for : C#, Java, Python, PHP and Ruby_                                             |
| **Supported datastores**                                         | Log Analytics Workspace (cloud)                                                                                     | <font color="green">**Multiple options**</font>                                                                                                                            |
| **Send metrics to on-site monitoring platform**                  | <font color="orange">**No**</font>                                                                                  | <font color="green">**Yes**</font> possible                                                                                                                                |
| **View (UI)**                                                    | Azure Monitor / Grafana                                                                                             | Zipkin / Jaeger / Grafana. <font color="orange">Azure Monitor (no support for metrics)</font>                                                                              |
| **Configure notifications**                                      | Azure Monitor                                                                                                       |                                                                                                                                                                            |
| **Dependencies**                                                 | _Arc_                                                                                                               | -                                                                                                                                                                          |
| **Buffering when it goes offline** No loss of observability data | App Insights logs for 48hrs                                                                                         | _No out of the box support_                                                                                                                                                |
| **Effort**                                                       | <font color="green">**Low**</font>                                                                                  | <font color="red">**High**</font>                                                                                                                                          |
| **Maturity / Risk**                                              | <font color="green">**Mature**</font>                                                                               | Still early                                                                                                                                                                |

## Individual instrumentation-based approaches

Using SDKs directly to instrument the application can be useful in some cases when the agents have short comings. However, given some of the disadvantages, this option is not often a suitable choice.

List of SDKs:

- [Application Insights SDK](https://learn.microsoft.com/en-us/azure/azure-monitor/app/api-custom-events-metrics)
- [OpenTelemetry SDK](https://opentelemetry.io/docs/instrumentation/)

Advantage:

- Comes often with auto instrumentation for popular languages
- Often supports all telemetry data (Logs, Metrics, Traces)

Disadvantages:

- Each application needs to manage a secrete to connect to send data upstream.
- Multiple connections to upstream resource
- Individual management of offline capabilities.
