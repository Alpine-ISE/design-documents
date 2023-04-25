
# Configure image registry

Author: Eliise Seling

## Problem

We want to be able pull images from environment specific image repositories.

## Context

Each of our environments has a separate GitOps repository and Azure container registry.

For the purpose of this ADR, there are two important steps that occur when application bundle versions are promoted to subsequent environments:

- Moving over the application bundle version folder to the subsequent GitOps repository
- Importing all the images within the application bundle to the subsequent image registry

As a result the image tag and name remain constant between all environments, but the each environment requires a different image registry.

## Decision Drivers

Following list drives the decision but it is not an exhaustive list of reasons.  

- Do you need to specify the image registry multiple times?
- What is the process for changing the image registry for an environment after it's set?
- How can you tell which registry is being used by a cluster?
- Can you configure the image registry on the cluster level, environment level or both?
  - Ideally both, but on environment level you do not have to configure each cluster individually
- Is there support for pulling images from multiple registries?
- How difficult is it to implement?
- How easy it is to debug? How easy it is to change the ACR manually?

## Options

### Flux variable substitution

The [Flux `kustomization` resource](https://fluxcd.io/flux/components/kustomize/kustomization/) supports a feature called [variable substitution](https://fluxcd.io/flux/components/kustomize/kustomization/#variable-substitution). By adding `spec.postBuild.substitute/From` to your Flux `kustomization` you can provide a map of key/value pairs holding the variables to be substituted in the final YAML manifest, after kustomize build.

By placing the `configMap` into an arbitrary folder in the root of the GitOps repository (in this case `.default/`), we allow the configuration the be applied to all clusters in the environment.

The `spec.postBuild` field cannot be added onto an existing `kustomization`. It **must** be added either when the kustomization is created or by modifying the `kustomization` within the cluster.

Flux `kustomization` resources are created and managed via the Config Service within the `cluster/<cluster_name>/flux` folder and Config Service doesn't support extending the `kustomizations` created.

With these limitations, a potential way to use the Flux variable substitution is by modifying the Config Service so that arbitrary patches can be added to the `cluster/<cluster_name>/flux` folder, similar to how patches can already be added in the `cluster/<cluster_name>/sets/<set_name>/<app_name>/<app_version>/`.

```yaml
# cluster/<cluster_name>/flux/apps-windowed.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: apps-windowed
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: gitops-repo
    namespace: flux-system
  path: ./clusters/dev-edge/sets/set-01
  prune: true


# cluster/<cluster_name>/flux/kustomization.yaml
resources:
  - apps-windowed.yaml
patches:
  - target:
      kind: Kustomization
      name: apps-windowed
    path: patch.json


# cluster/<cluster_name>/flux/patch.yaml
[
    {
        "op": "add",
        "path": "/spec/postBuild",
        "value": {
            "substituteFrom": [
                {
                    "kind": "ConfigMap",
                    "name": "cluster-vars"
                }
            ]
        }
    }
]

# .defaults/cluster-vars.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-vars
data:
  ACR_DOMAIN: <acr-name>.azurecr.io


# OUTPUT: after kustomize build
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: apps-windowed
  namespace: flux-system
spec:
  interval: 1m
  path: ./clusters/dev-edge/sets/set-01
  postBuild:
    substituteFrom:
    - kind: ConfigMap
      name: cluster-vars
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-repo
    namespace: flux-system

# OUTPUT: resulting deployment manifest on cluster
apiVersion: apps/v1
kind: Deployment
metadata:
  name: module-name
spec:
  replicas: 1
  selector:
    matchLabels:
      app: module
  template:
    metadata:
      labels:
        app: module
    spec:
      containers:
        - name: module-container-name
          image: <acr-name>.azurecr.io/<container-name>:<container-tag>
```

Pros:

- Image registry is specified once
- Able to configure the image registry on both cluster and environment level (manually for both)
- Support for pulling images from multiple registries
- Enables support for generic variable substitution as well
- Provide support for generic patching of Flux `kustomizations`
- All configuration is defined in the gitops repository

Cons:

- Must modify Config Service
  - Patches are already supported for the `cluster/<cluster_name>/sets/<set_name>/<app_name>/<app_version>/` folder
- Can Config Service support not overwriting?
- Is it maintainable? May be a lot of files
- Harder to debug issues because templating happens on the cluster level, via the Flux Operator, meaning you can't see the templated `deployment` manifests in the gitops repo

### Kustomize replacements

Using [kustomize `replacements`](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/replacements/) feature to replace the image url.
By placing the `configMap` into an arbitrary folder in the root of the GitOps repository (in this case `.default/`), we allow the configuration the be applied to all clusters in the environment.

[Support the ability to replace the image registry domain · Issue #4414 · kubernetes-sigs/kustomize](https://github.com/kubernetes-sigs/kustomize/issues/4414)

```yaml
# apps/<app_name/<app_version>/kustomize.yaml
resources:
...
  - ../../../.defaults/image-registry-config.yaml

replacements:
  - source:
      kind: ConfigMap
      name: local-config
      fieldPath: data.registry
    targets:
      - select:
          kind: Deployment
        fieldpaths:
          - spec.template.spec.containers.*.image
        options:
          delimiter: "/"
          index: 0

# .defaults/image-registry-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-config
data:
  registry: <acr-name>.azurecr.io
```

Pros:

- Able to configure the image registry on both cluster and environment level (manually for both)
- Image registry is specified once
- Least amount of work to add
- Everything is in gitops

Cons:

- No support for pulling images from multiple registries

### K3s registries.yaml

Using the K3s [Private Registry Configuration](https://docs.k3s.io/installation/private-registry) you can specify
mirrors in `/etc/rancher/k3s/registries.yaml` that instruct `containerd` to fetch images from any registries defined there instead of in the application deployment manifest.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: module-name
spec:
  template:
    spec:
      containers:
        - name: <container-name>
          image: default.com/<container-name>:<container-tag>

# registries.yaml
mirrors:
  default.com:
    endpoint:
      - "<acr-name>.azurecr.io"
```

Pros:

- Simple setup: created on device with the initial onboarding script (cloud-init)
- Support for pulling images from multiple registries
- Easiest to debug, because templating happens as the files are added to the repo

Cons:

- Accessing `registries.yaml` requires shell access to the clusters device
- Potentially don't know which registry is being used without accessing `registries.yaml` on the device
  - image registry could be exposed in K3s logs or other metrics that are exported to Azure Arc **(unverified)**
- Changing image registry requires modifying `registries.yaml` which means accessing each device individually
- Registry configuration set on an individual cluster level, not on an environment level
- Need to restarts k3s service to apply the change

### Modify the image registry url when promoting a new version

Do the templating ourselves when we promote to a new environment in the pipeline

Pros:

- All configuration is in the gitops repository, making it very explicit and easy to view 
- Able to configure the image registry on both cluster (with patches) and environment level

Cons:

- Requires custom implementation to be created in the pipeline when updating
- Changing the registry url requires you to change the url in each app deployment, which means creating a new application version with the new registry url due to the applications being immutable

## Conclusion

Answers for decision drivers:

| Decision drivers                                                                       | Flux variable substitution     | Kustomize replacements     | K3s registries.yaml        | Pipeline templating               |
| -------------------------------------------------------------------------------------- | :----------------------------- | :------------------------- | :------------------------- | :-------------------------------- |
| Do you need to specify the image registry multiple times?                              | No                             | No                         | Yes, in each cluster       | Yes, in each app deployment       |
| What is the process for changing the image registry for an environment after it's set? | Change ConfigMap in Gitops     | Change ConfigMap in Gitops | Change file on each device | Change apps deployment on cluster |
| How can you tell which registry is being used by a cluster?                            | Gitops repo                    | Gitops repo                | File on device             | Gitops repo                       |
| Can you configure the image registry on the cluster level, environment level or both?  | Both, manually                 | Both, manually             | Cluster only               | Both                              |
| Is there support for pulling images from multiple registries?                          | Yes                            | No                         | Yes                        | Yes                               |
| How difficult is it to implement? (estimated)                                          | Change Config Service + gitops | Change gitops              | Change cloud-init          | Change promotion pipeline         |

Scored decision criteria:

| Criteria                                                                                                                             | Flux variable substitution | **Kustomize replacements**               | K3s registries.yaml | **Pipeline templating**      |
| ------------------------------------------------------------------------------------------------------------------------------------ | -------------------------- | ---------------------------------------- | ------------------- | ---------------------------- |
| 1. Maintainable - very explicit and simple to understand                                                                             | 3/5                        | 4/5                                      | 0/5                 | **5/5**                      |
| 2. Easy to debug:<br> - Configuration is easily viewable from the GitOps Repo & Config Service.<br> - `kustomize build` can be used. | 3/5                        | **4/5**                                  | 0/5                 | **4/5**                      |
| 3. Image registry is specified in one place                                                                                          | **Yes**                    | **Yes**                                  | No                  | No                           |
| 4. Easy to change the ACR URL (for existing clusters)                                                                                | **Yes**                    | **Yes**                                  | No                  | No                           |
| 5. Easy to change URL for new deployments                                                                                            | No                         | **Kustomization changes within cluster** | No                  | **Requires new app version** |
| 6. Easy to implement                                                                                                                 | 1/5                        | **5/5**                                  | **5/5**             | 3/5                          |
| 7. No reliability on Config Service                                                                                                  | No                         | **Yes**                                  | **Yes**             | **Yes**                      |
| 8. Configuration visible at:                                                                                                         | Flux/Cluster               | **Kustomization build**                  | Device              | **Gitops**                   |
