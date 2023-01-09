# Configuration types and configuration scheduling

## Context

We want the ability to apply configuration changes in two ways:

- Immediate (∂t=10min) application to the cluster
- Scheduled to occur during the next maintenance window

In short, the solution should enable applying configuration changes during a pre-determined time window and prohibit potentially inconsistent states.

> Note: The goal is not to schedule fine-grained or specific configuration changes (eg. one single value) in their solution. Rather they want to be able to apply certain sets of configurations only in designated time windows.

## Decision drivers

- Manageable (windows can be set) either (a) manually via Git or (b) via Config Service REST API
- Simplicity (ease of use) for the end user
- Technical elegance & simplicity
- Ease of debugging
- Low maintenance
- Generalizable
- Mitigate the risk of inconsistent configuration states or conflicts between live & scheduled changes (see Appendix below)
- Little to no changes to ConfigService

## Considered options

### Option 1: Use an external system to manage the maintenance windows

Solve this problem by leveraging the approval feature of ConfigService to control the time when changes get merged into the main branch of the GitOps repository, from where the changes are then picked up by Flux and applied to the cluster.
Create a tool that automatically approves scheduled changes when a maintenance window begins.
This external system will also have to persist in the schedule for the maintenance windows.

### Option 2: Use GitOps, Flux, and K8s native functionality to manage the maintenance window

Solve this problem by leveraging Flux, K8s, and Kustomize features to enable certain configuration changes to be applied only during maintenance windows.

In the simple scenario of two types of configuration like in the problem statement the GitOps setup of a cluster would look the following:

```bash
.
├── apps
│  └── app1
├── clusters
│  ├── prod-edge
│  ├── staging-edge
│  └── dev-edge
│      ├── infra
│      │    └── kustomization.yaml
│      ├── infra-live
│      │    ├── jobs
│      │    │    ├── window-start-patch.yaml
│      │    │    ├── window-stop-patch.yaml
│      │    │    └── kustomization.yaml
│      │    └── kustomization.yaml
│      ├── sets-live
│      │    ├── asset-01
│      │    │   └── app1
│      │    │         └── kustomization.yaml
│      │    └── kustomization.yaml                        # This will  reference versions of type "live" which live under `sets-window`
│      └── sets-window
│          └── asset-01
│              ├── app1
│              │  ├── 1.0-live
│              │  │    ├── configuration-live.yaml
│              │  │    └── namespace.yaml
│              │  └── 1.0-window
│              │      └── kustomization.yaml
│              └── kustomization.yaml                     # This will only reference  versions of type "window"
└── infra
    ├── base
    └── jobs
         ├── kustomize-suspend.yaml
         └── kustomize-resum.yaml
```

By separating the configuration types into two versions folders, we can reference them from different roots `kustomization.yaml` (which the flux kustomization resources point to), as well as able to edit both configuration types with ConfigService.

```bash
flux create kustomization apps-<type>  \
    --path="./clusters/dev-edge/sets-<type>/asset-01/" \
    --source=$REPOSITORY \
    --prune=true \
    --interval=1m
```

By having independent kustomization resources for different types of configuration, we can use the suspension feature of Flux to stop/start the synchronization of the GitOps repository to the cluster of each type independently.
This means, that while configurations of type 'live' can be changed continuously during the day, configurations that should only be changed during maintenance windows can be suspended until such a window arrives and disabled right after its end.

In our example above this means, the flux kustomization resource of the configurations in `./clusters/dev-edge/sets-window/asset-01/` will be suspended after the initial setup, until the maintenance window arrives. During the maintenance, window changes will be applied as frequently as defined in the parameter `interval` (here `interval=1m`)

The suspension can be done manually or using a scheduled process.
By using K8s CronJobs to resume and suspend the kustomization resource `apps-window`, we are able to manage the definition and kustomization of these jobs within the GitOps cluster and leverage ConfigService to change the job configuration for each cluster individually via patching.
This could result in a CronJobs default definition under `infra/jobs`. Even though the CronJobs are managing the maintenance windows, the configuration of the CronJobs themselves happens via live configurations. This ensures the window period and time can be configured at any given time.

```yaml
# kustomize-resume.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: resume-job
spec:
  schedule: "* * * * *"
  suspend: true
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: bitnami/kubectl:latest
              imagePullPolicy: IfNotPresent
              command:
                - /bin/sh
                - -c
                - kubectl patch kustomizations.kustomize.toolkit.fluxcd.io  mw-poc -n flux-system --patch '{"spec":{"suspend":true}}' --type merge;
          restartPolicy: OnFailure
```

The customized patch configuration of the jobs in the respective cluster folder of the cluster `./clusters/dev-edge/infra/jobs`.

```yaml
#window-start-patch.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: resume-job
spec:
  suspend: false
  schedule: "0 8 * * 1".  # Starts the maintenance window every Monday at 8:00am
```

```yaml
#window-stop-patch.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: resume-job
spec:
  suspend: false
  schedule: "0 8 * * 1".  # Stops the maintenance window every Monday at 8:30am
```

This would solve the definition of maintenance windows for our use case and enable us to use ConfigService and not another external system to manage the definition of these maintenance windows.

Additionally, by suspending the kustomization resource (hence stopping the synchronization of the GitOps repo to the cluster by flux) we can still push any planned changes to the GitOps repository's main branch. This means the customer using ConfigService would be able to know, what the desired state is for the configuration types after the window has passed and can take that into account when changing the configurations. The explicit separation of configuration types will also handle potential overwrites of live changes by maintenance window changes.

As illustrated on two types of windows, this solution can be extended to accommodate more window types. However, this will increase the number of kustomization resources.

## Proposal

**Option 2**, because:

- The maintenance of the key components is handled by third parties (eg. K8s, Flux, Kustomize).
- Simple to use, as the used technologies are well documented.
- Solution can be generalized and customized to match the individual needs of more than one window.
- Solution can be used in parallel with approvers to also enable scheduling of fine grain changes.

## Consequences

- Option 2 does not enable fine-grain configuration scheduling without a large overhead. However, the existing approver process in ConfigService can be used to handle the configuration scheduling.
- Config Service needs to be only adjusted deploy new live version.

---

<br>

## Appendix - Mitigating the risk of inconsistent configuration states or conflicts between live & scheduled changes

Considerations:

Given that not all changes are applied to the cluster at the same time, the solution needs to make sure no inconsistent configuration state can be reached.
Eg:

- Overwriting of live configuration by outdated maintenance window changes
- Incompatible configurations which are defined in the live vs maintenance window
- Targeting the same configuration in the live and maintenance windows and causing a "merge conflict"
