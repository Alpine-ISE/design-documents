
# Creating Flux kustomizations

## Problem

There are two ways to create Flux `kustomizations` and there needs to be a decision on which way is more suitable for the solution.

## Context

Currently, all kustomization are created [in terraform](https://learn.microsoft.com/en-us/azure/templates/microsoft.kubernetesconfiguration/2022-01-15-preview/extensiontypes?pivots=deployment-language-terraform) with [Azure Arc Flux extension](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/conceptual-gitops-flux2) when the device is provisioned.

This causes two problems:

- When creating a new cluster application sets (`clusters/<cluster_name>/sets/<set_name>`) via the Config Service, there no Flux `kustomization` deploys the new set into the cluster.
- Additionally, the extension uses the [Azure REST APIs](https://learn.microsoft.com/en-us/rest/api/azure/) [Flux Configurations endpoint](https://learn.microsoft.com/en-us/rest/api/kubernetesconfiguration/flux-configurations/create-or-update) to configure Flux.
That endpoint only supports a limited number of Flux `kustomization` parameters, as seen [here](https://learn.microsoft.com/en-us/rest/api/kubernetesconfiguration/flux-configurations/create-or-update?tabs=HTTP#kustomizationdefinition).

This means that a number of features within the `kustomization` resource are not configurable via the Arc Flux extension, such as [variable substitution](https://fluxcd.io/flux/components/kustomize/kustomization/#variable-substitution) and [health assessment](https://fluxcd.io/flux/components/kustomize/kustomization/#health-assessment), the latter being a requirement of using the [notification provider](https://fluxcd.io/flux/guides/notifications/).

In the rest of this document `kustomization` will refer to [Flux `kustomizations`](https://fluxcd.io/flux/components/kustomize/kustomization/) and sets will refer to application sets.

## Decision Drivers

Following list drives the decision but it is not an exhaustive list of reasons.

- How easy it is to create application sets?
- Are Flux `kustomization` health checks supported?
- Are other Flux `kustomization` features supported?
- How can you view the status of the `kustomizations`?

### Results of Flux kustomization testing

#### Are manual changes to Flux Kustomizations created via the Azure Arc extension overridden?

**Test protocol**

1. Create a simulated edge device via IaC. The terraform creates four Flux `kustomizations` via the Azure Arc extension
2. Edit one of those Flux Kustomizations, `gitops-repo-apps-windowed`, change `.spec.path` from `./clusters/edge-device-0/sets/set-01` to `./clusters/edge-device-0/sets/set-01/test`

**Result**

After 48h+ the change is still present. However, the change is not reflected in the Kustomizations blade on the Azure portal:

![Change now shown in portal](./images/CreatingClusterApplicationSets/changed-kustomization.png)

#### Can a Flux Kustomization install more Flux Kustomizations?

**Test protocol**

1. Create a simulated edge device via IaC. The terraform creates four Flux kustomizations via the Azure Arc extension.
2. Change one of the Flux kustomizations, `gitops-repo-apps-live` so that `.spec.path` points to clusters/<cluster_name>/flux

**Result**

The new Flux Kustomizations from `/clusters/device-name/flux` are installed onto the cluster. Those Kustomizations are also picked up by the Flux controller, and new resources are installed into the cluster based on those new Flux `kustomizations`.

#### Are Flux Kustomizations that are manually installed onto the cluster visible on the Azure portal?

**Test protocol**

Follow steps from the [previous test](#can-a-flux-kustomization-install-more-flux-kustomizations).

**Result**

The new Flux `kustomizations` are present in the Azure portal, on the gitops view, in the *'Configuration objects'* blade:

![New kustomizations are present](./images/CreatingClusterApplicationSets/new-kustomizations.png)

Even the `kustomizations` from the `flux/` folder are shown. The new `kustomizations` seem to work in a similar manner as Azure Arc installed ones in this view.

The new `kustomizations` are however **not** shown on the *'Kustomizations'* blade.

#### Can you stop Flux kustomization changes to be overwritten by the state in the `flux/` folder?

**Test protocol**

1. Create all kustomization resources using the bootstrap customization
2. Manualy suspend maintenance window kustomization using `kubectl` or `flux cli`.
3. Check if the added suspend value will be removed by the bootstrap kustomization.

**Result**

After waiting for up to 1h the `suspend` key has not been removed from the kustomization.

Flux seems to use a "merge" behavior when updating the kustomization resources.
Unless the parameter `suspend` is set in the manifests created by ConfigService, it will not be overwritten by flux. This behavioure is true, even if it is changed/added manually on the cluster.
Although we are not setting the `suspend` in the generated manifests, to preemptively prevent overwrites, we can label all maintenance window kustomization resources with `kustomize.toolkit.fluxcd.io/reconcile=disabled`. This label will prevent the resources to be reconciliation by flux.
During the maintenance window, the label can be overwritten with `kustomize.toolkit.fluxcd.io/reconcile=enabled`, so that potential changes to the kustomization manifest can be reconciled.

## Options

### Follow up process after creating a set

After a new set is created we can proceed with a follow up process managed outside of the Config Service that will created the new `kustomization`.

Potential solutions include:

- Manually triggered pipeline that will create the new kustomization based on user input
- Automatically triggered pipeline based on changes in the GitOps repository
  - Would require a way to associate a cluster folder with a Arc Cluster (naming the cluster the same as Arc Connect in a known, additional association data stored in a file, ect)

Pros:

- Able to reconfigure `kustomizations` via Azure and view the status of the Flux resources created by the Flux Extension

Cons:

- Creation and management of sets requires an additional step outside of config-service
- Unable to create `kustomization` with all supported features

### Kustomizations are created via Config Service

Split Flux configuration into two parts:

a) Creation of a `GitRepository` source and a '`bootstrap`' `kustomization` with Arc Flux extension
b) The '`bootstrap`' `kustomization` points to a `cluster/<cluster_name>/flux` folder, which contains the rest of the Flux kustomizations needed for the cluster, defined in yaml. These Flux kustomizations are already being created by the Config Service.

Pros:

- Able to reconfigure the '`bootstrap`' Flux kustomization via Azure. As this is the entrypoint for Flux, this allows fully reconfiguring the cluster.
- Able to view at least the state of the bootstrap Flux kustomization in Azure.
- Config Service already supports automatically generating the `kustomization` within the `cluster/<cluster_name>/flux` folder
- Can possibly still view the status of `kustomization` as part of the clusters resources in Azure **(unverified)**
- Able to reconfigure and view state of the `kustomization` via the GitOps repository

Cons:

- Config Service `flux/` folder generation does not support adding additional `kustomization` features, such as `spec.postBuild` and `spec.healthChecks`
  - This could potentially be added
- 'Maintenance Window' requires an additional label (`kustomize.toolkit.fluxcd.io/reconcile: disabled`) to be added it to prevent overwriting via the existing Flux `kustomization`. More information on flux reconciliation can be found [here](https://fluxcd.io/flux/components/kustomize/kustomization/#reconciliation).

Changes required to adopt this feature:
> NOTE: to support creating 'Maintenance Window' `kustomizations` within the `flux/` folder they would need to be moved under `cluster/<cluster_name>/sets` from `cluster/<cluster_name>`
> NOTE: an additional flag needs to be added by the Maintenance Window cron job
> NOTE: The Config Service default namespace needs to be changed from `flux-workspace` to `flux-system`
