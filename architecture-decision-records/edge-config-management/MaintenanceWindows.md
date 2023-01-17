# Reconciliation/Maintenance windows when using GitOps with Flux

When using Flux to manage a K8s cluster every new change in your repository will be immediately applied to the cluster’s state. In some use cases, the newest changes to a GitOps repository should only apply to the cluster within a designated time window. For example, the cluster should reconcile to the newest changes of the GitOps repository only between Monday 8 am to Thursday 5 pm. Any change coming into the GitOps repository on Friday or the weekend will have to wait till Monday 8 am to be applied.

What are the scenarios this could be used for in real life?

- Sometimes the cluster is connected to external systems, which need to be in maintenance mode before updates can be applied.
- You want to be able to determine a designated time window when the next changes go into production so that in case of an issue you can react quickly.

Flux does not have a scheduling ability in place as of now. But using the flux suspension feature and K8s Cron jobs a similar solution can be achieved.

## Solution

The flux suspension feature allows users to halt and resume a `kustomization` resource from reconciling the newest changes.

```bash
flux suspend kustomization <name>  # halts reconciliation of changes
flux resume kustomization <name>   # resumes reconciliation of changes
```

Combining this with the scheduling ability of K8s `CronJobs`, flux `kustomization` resource can be managed by suspending reconciliations at the end of a reconciliation window and start applying changes again at the beginning of the new time window.
For this, we need an opening `CronJob` and a closing `CronJob`, as well as a service account, which allows the jobs to manipulate resources on the cluster.
See below a template of such jobs:

```yaml
# open-reconciliation-window-job.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: open-reconciliation-window
  namespace: jobs
spec:
  schedule: "* * * * *"
  suspend: true
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: <service-account-name>
          containers:
            - name: hello
              image: ghcr.io/fluxcd/flux-cli:v0.36.0
              imagePullPolicy: IfNotPresent
              command: ["/bin/sh", "-c"]
              args:
                - flux resume kustomization infra -n flux-system && flux resume kustomization apps -n flux-system;
          restartPolicy: Never
```

```yaml
# close-reconciliation-window-job.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: close-reconciliation-window
  namespace: jobs
spec:
  schedule: "* * * * *"
  suspend: true
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: <service-account-name>
          containers:
            - name: hello
              image: ghcr.io/fluxcd/flux-cli:v0.36.0
              imagePullPolicy: IfNotPresent
              command: ["/bin/sh", "-c"]
              args:
                - flux suspend kustomization infra -n flux-system && flux suspend kustomization apps -n flux-system;
          restartPolicy: Never
```

> **Note**: To use th template replace the `spec.schedule` with a cron string and set `spec.suspends` to `false`.

The services account needs the following rights:

```yaml
- apiGroups: ["kustomize.toolkit.fluxcd.io"]
  resources: ["kustomizations"]
  verbs: ["list", "patch", "watch", "update", "get"]
```

### Use GitOps to manage the reconciliation window resources

By placing the definition of the `CronJob` within the GitOps repository itself, it becomes part of the regular infrastructure. Similar to how one would customize infrastructure components for different clusters, `Cronjobs`'s schedule can be customized using the patching feature of `kustomize`.
For this, a `patch.yaml` which contains cron string needs to be added on the cluster level, as well as a reference to the patch in the `kustomization.yaml`.

Eg. The window opens up every 10 minutes for 5 minutes.

```yaml
# open-reconciliation-window-patch.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: open-reconciliation-window
  namespace: jobs
spec:
  schedule: "0,10,20,30,40,50 * * * *"

# close-reconciliation-window-patch.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: close-reconciliation-window
  namespace: jobs
spec:
  schedule: "5,15,25,35,45,55 * * * *"
  suspend: false
```

```yaml
# kustomization.yaml
resources:
  - ./../../../../infra/reconciliation-window

patchesStrategicMerge:
  - ./open-reconciliation-window-patch.yaml
  - ./close-reconciliation-window-patch.yaml
```

The full solution can be found in this sample repository: [ Sample 1 - GitOps repository that enables reconciliation windows](https://github.com/MahrRah/flux-reconciliation-windows-sample/tree/main/Sample1)

### Real-time and Maintenance window changes

This solution can be taken even one step further for scenarios where multiple configuration types are required. 
A consequence of using the Flux suspension feature to manage the reconciliation windows is, that the granularity of control ends at what resources get managed by one `kustomization`.
So as a result, in a case where a cluster needs real-time changes, as well as changes that get applied during a reconciliation window, the resources now need to be managed by two `kustomization` resources.
Hence refactoring the GitOps repository in such that each resource is either managed by one or the other `kustomization` resources, in adition to the above approach for one of the `kustomization` resources would enable having different configuration types.
There are multiple ways how this can be achieved. One of them is to split each application into two sub-version, to separate the resources which can be changed in real-time vs during a maintenance window.
A sample repo for this can be found here: [Sample 2 -  GitOps repository to support both real-time and reconciliation window changes](https://github.com/MahrRah/flux-reconciliation-windows-sample/tree/main/Sample2)

Advantages of this approach:

- All open-source or K8s and Flux native components
- Managed the same way every other application is managed on the cluster
- Simple to use, as the used technologies are well documented.
- With some adjustments to the GitOps repo it can handle different configuration types eg. real-time and maintenance windows

Consequences:

- Does not enable fine-grain configuration scheduling without a large overhead. The granularity of control stops at what resources are all managed by one kustomization.
  Config Service and the config service API needs to be adjusted to handle these new resources and folder structures in the GitOps repository.