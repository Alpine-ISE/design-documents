# Architecture decision records & design documents

This repository contains a collection of the Alpine crews architecture design records and design documents.

The ADR's should include the problem, a short description of the analyzed or discussed solutions and pro and cons for the different solutions. This repository does not contain the actual decision that was taken; the reader should make their own conclusions.

## Table of contents

### Edge configuration management
- [GitOps: Representation of Environments](./architecture-decision-records/edge-config-management/GitOpsEnvironments.md)
- [GitOps: Promotion of changes](./architecture-decision-records/edge-config-management/GitOpsEnvironmentPipeline.md)
- [GitOps: Large configuration files](./architecture-decision-records/edge-config-management/LargeConfigurationFiles.md)
- [GitOps: Health checks](./architecture-decision-records/edge-config-management/HealthChecks.md)
- [K8s: Image registry configuration](./architecture-decision-records/edge-config-management/ConfigureImageRegistry.md)
- [K8s: Creating Flux kustomizations](./architecture-decision-records/edge-config-management/CreatingFluxKustomizations.md)
- [Git: How to connect to our GIT from edge/ConfigService](./architecture-decision-records/edge-config-management/ConnectionToGit.md)
- [Maintenance windows: General implementation of maintenance windows](./architecture-decision-records/edge-config-management/MaintenanceWindows.md)
- [Maintenance windows: Image for maintenance window job](./architecture-decision-records/edge-config-management/FluxImageForMaintenanceWindow.md)
- [Rollback: how to roll back clusters to a known good state](./architecture-decision-records/edge-config-management/Rollback.md)
