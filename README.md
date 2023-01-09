# Architecture decision records & design documents

This repository contains a collection of the Alpine crews architecture design records and design documents.

The ADR's should include the problem, a short description of the analyzed or discussed solutions and pro and cons for the different solutions. This repository does not contain the actual decision that was taken; the reader should make their own conclusions.

## Table of contents

### Edge configuration management
- [GitOps: Representation of Environments](./arc/GitOpsEnvironments.md)
- [GitOps: Promotion of changes](./GitOpsEnvironmentPipeline.md)
- [GitOps: Large configuration files](./LargeConfigurationFiles.md)
- [GitOps: Health checks](./HealthChecks.md)
- [K8s: Image registry configuration](./ConfigureImageRegistry.md)
- [K8s: Creating Flux kustomizations](./CreatingFluxKustomizations.md)
- [Git: How to connect to our GIT from edge/ConfigService](./ConnectionToGit.md)
- [Maintenance windows: General implementation of maintenance windows](./MaintenanceWindows.md)
- [Maintenance windows: Image for maintenance window job](./FluxImageForMaintenanceWindow.md)
- [Rollback: how to roll back clusters to a known good state](./Rollback.md)