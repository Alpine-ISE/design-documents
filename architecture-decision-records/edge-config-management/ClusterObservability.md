# Cluster observability

## Problem

Kubernetes clusters run multiple workloads and host by design distributed systems. Hence, having a good observability suite for these clusters is of the essence, when it comes to measuring and monitoring as well as troubleshooting.

What we understand under the observability of a Kubernetes cluster can be split into two main parts:

Infrastructure observability:

- Cluster-level resource metrics
- Operational status of kube-system

Workload observability

- Application health and readiness
- Application telemetry data (e.g. Metrics, Traces, Logs)

There are multiple ways to extract the needed information from a cluster and get an overview of the cluster's health from the outside.
This article aims to showcase some of these options and include their up and downsides.

- Collector-based approaches
  - Azure Container Insights
  - OpenTelemetry
- Individual instrumentation-based approaches
  - SDK instrumentation

## Solutions

### Collector-based approaches

#### Azure Monitor Container Insights for Azure Kubernetes clusters

[Container insights](https://docs.microsoft.com/azure/azure-monitor/containers/container-insights-enable-arc-enabled-clusters) is an Azure Monitor extension. It currently runs on the [Log Analytics Agent for Linux](https://docs.microsoft.com/azure/azure-monitor/agents/agent-linux?tabs=wrapper-script) as an agent that is collecting container logs and performance metrics (memory, CPU) from the cluster (controller, nodes, and containers).

Using Container Insights **requires either an AKS or Azure Arc-enabled cluster** (see [Supported configurations](https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-onboard#supported-configurations)).

Advantages of Azure Monitor Container Insights.

- Supports cluster metrics and container logs out of the box with Azure Arc integration.
- Leverages Azure Monitor Container dashboards that allow monitoring at scale (Cluster Insights, Fleet Overview)
- Deployment and management via Azure (REST API, ARM/Bicep, Azure Portal)
- Supports temporary disconnects

Disadvantages:

- Does not support multiple pipelines (i.e., only exporting to Log Analytics and Azure Monitor Metrics store)
- Limited Container Log filtering capabilities
- No distributed tracing (requires separate SDK such as App Insights / OpenTelemetry).
- Doesn't support offline mode

#### OpenTelemetry Collector

[OpenTelemetry](https://opentelemetry.io/) provides an SDK to expose telemetry using the OpenTelemetry Protocol (OTLP) to external systems, one of such is the [OpenTelemetry collector](https://opentelemetry.io/docs/collector/).
The OpenTelemetry collector defines pipelines to receive metrics, logs and traces from multiple producers, process it, and distribute across multiple observability backends.
It currently supports a list of receiver types (see [core](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver) and [extended](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver) ), as well as exporters (see [core](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter) and [extended](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter))

Advantages of OpenTelemetry with the use of the collector.

- Support for all three observability pillars: metrics, logs, traces
- Support for multiple protocols and receivers eg. fluentbit, Prometheus, Kafka, OpenCensus, etc...
- Support for multiple exports eg. File system, Azure monitor exporter, Grafana, Zipkin etc...
- Support for multiple pipelines eg. Export to cloud as well as serve data locally.
- Single container (collector) connecting to the cloud and no need to share instrumentation keys and other secrets with other applications.
- Single source to handle buffering and statefulness
- Data aggregation before promoting it to upstream, such as cloud (see [Observability on Edge at Scale](./README.md#observability-on-edge-at-scale))
- Support for offline mode (see [Deliver data from Edge to the Cloud](./README.md#deliver-data-from-edge-to-the-cloud))

Disadvantages:

- Early in their roadmap

#### Comparison of the collector bases approaches

|                                                                  | Container Insights for Azure Arc-enabled cluster                                                                     | OpenTelemetryCollector                                                                                                                                                                                |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Infrastructure metrics Resources: CPU / RAM**                  | <font color="green">**Yes**</font> – out the box for the cluster (not for the VM)                                    | <font color="red">**No**</font> - Requires additional scrapper like [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) to expose systems metrics for the collector to grab</font> |
| **Application metrics** "e.g.Requests/sec, workload health"      | <font color="green">**Yes**</font> - application needs to expose Prometheus endpoint (Prometheus /OpenTelemetry SDK) | <font color="green">**Yes**</font> - application needs to expose Prometheus endpoint (Prometheus /OpenTelemetry SDK)                                                                                  |
| **Send logs to the cloud**                                       | <font color="green">**Yes**</font> – uses Log Analytics agent for Linux (to be deprecated in 2024).                  | <font color="green">**Yes**</font>                                                                                                                                                                    |
| **Distributed tracing** "Performance: latency, etc."             | Needs to instrument with App-Insights SDK (or OpenTelemetry SDK) <br/>                                               | Need to instrument with OpenTelemetry SDK _OpenTelemetry SDK supports auto instrumentation for : C#, Java, Python, PHP and Ruby_                                                                      |
| **Supported datastores**                                         | Log Analytics Workspace (cloud)                                                                                      | <font color="green">**Multiple options**</font>                                                                                                                                                       |
| **Send metrics to on-site monitoring platform**                  | <font color="orange">**No**</font>                                                                                   | <font color="green">**Yes**</font>                                                                                                                                                                    |
| **View (UI)**                                                    | Azure Monitor / Grafana                                                                                              | Azure Monitor / Zipkin / Jaeger / Grafana</font>                                                                                                                                                      |
| **Configure notifications**                                      | Azure Monitor                                                                                                        |                                                                                                                                                                                                       |
| **Dependencies**                                                 | _Azure_                                                                                                              |                                                                                                                                                                                                       |
| **Buffering when it goes offline** No loss of observability data | App Insights logs for 48hrs                                                                                          | Support for [persistent queue](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/exporterhelper#persistent-queue) (still in _alpha_ state)                                 |
| **Effort**                                                       | <font color="green">**Low**</font>                                                                                   | <font color="red">**High**</font>                                                                                                                                                                     |
| **Maturity / Risk**                                              | <font color="green">**Mature**</font>                                                                                | Still early                                                                                                                                                                                           |

## Individual instrumentation-based approaches

Using SDKs directly to instrument the application can be useful in some cases when the agents have short comings. However, given some of the disadvantages, this option is not often a suitable choice.

List of SDKs:

- [Application Insights SDK](https://learn.microsoft.com/azure/azure-monitor/app/api-custom-events-metrics)
- [OpenTelemetry SDK](https://opentelemetry.io/docs/instrumentation/)

Advantage:

- Comes often with auto instrumentation for popular languages
- Often supports all telemetry data (Logs, Metrics, Traces)

Disadvantages:

- Each application needs to manage a secrete to connect to send data upstream.
- Multiple connections to upstream resource
- Individual management of offline capabilities.
