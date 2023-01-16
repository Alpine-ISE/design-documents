# Managing and configuring Kubernetes workloads via GitOps

Kubernetes is increasingly being used in "heavy edge" sitautions in place of lightweight IoT - for example, in factories and manufacturing plants where a server is expected to function with intermittent Internet connectivity.

[GitOps using Flux](https://www.weave.works/technologies/gitops/) is ideally suited for deploying workloads to Kubernetes in such scenarios.  This playbook considers some of the common configuration challenges which can be encountered:

<br>

GitOps setup:

1. [Authentication options for Flux to connect to the GitOps repo](ConnectionToGit.md).

2. [How to define the Flux kustomizations on the actual Kubernetes cluster](CreatingFluxKustomizations.md) - in a maintainable manner.


<br>

Environments:

1. [How to structure the GitOps repo(s) to serve multiple environments (eg. "dev", "staging", "prod")](GitOpsEnvironments.md) - whether to use a single repo with multiple branches, or multiple repos, and the pros/cons of such approaches.

2. [How to promote/release code between environments via a pipeline](GitOpsEnvironmentPipeline.md).

3. [How to have the Kubernetes clusters in different environments ("dev", "staging", "prod") pull from different container registries](ConfigureImageRegistry.md) - in a scalable and maintainable manner.


<br>

Configuration Management:

6. [Managing Secrets in Kubernetes](SecretsManagement.md) - in an operationally efficient manner (e.g. developer efficiency, and easy secret rotation).

7. [How to schedule configuration changes to take effect at a future date and time range](MaintenanceWindows.md) - since some industrial plants have predefined maintenance windows, outside of which changes should not be made.

8. [Supporting large configuration files](LargeConfigurationFiles.md), where "large" is defined as greater than 1MB.



<br>

Observability, audit and recovery:

9. [How to know whether changes made at the GitOps repo level have been successfully deployed to the server](NotificationProvider.md).

10. [Options for realtime observability of the cluster workloads](ClusterObservability.md)

11. [Rolling back to prior cluster and configuration states](Rollback.md).



<br>

## Terminology used:

* "Application" or "Application Bundle" refers to a collection of "edge modules" (i.e. Docker containers) which will be deployed together to perform some functionality.  i.e. each module will be a separate pod in the Kubernetes cluster.

* We shall refer to the git repository from where Flux pulls configuration files as the "GitOps repo".  It follows a predefined structure; a [sample GitOps repo](https://github.com/buzzfrog/contoso3) is available.


```
├── apps
│   ├── dataprocessor
│   └── computervision
│       ├── v1.1
│       └── v1.2
├── clusters
│   ├── london
│   ├── sanfrancisco
│   └── seattle
└── infra
    └── base
```

  - The GitOps repository contains 3 base folders:
    - `apps`: Contains the base setup of each application. Each application is in turn versioned and can contain one or more deployments, services, ConfigMaps etc.
    - `clusters`: Contains a directory for each Kubernetes cluster which is connected to this GitOps repository (e.g. in the example above, 3 Kubernetes clusters connect to this GitOps repo).
    - `infra`: Contains common services which the applications need to run on the cluster eg. an MQTT Broker.
