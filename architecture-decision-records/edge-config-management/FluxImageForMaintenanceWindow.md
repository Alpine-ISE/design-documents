# Flux Image For Maintenance Window

## Problem

During Maintenance Window, we need to suspend and resume Flux Kustomizations of configured modules.  
To achieve this, we are interacting with k8s controllers from within a CronJob.  
These CronJob pods need `kubectl` or `flux` CLI to run the commands.  

## Context

There are different public images available from credible sources and the two images we have tried in our POCs are:

1. [Bitnami](https://hub.docker.com/r/bitnami/kubectl/) => `docker pull bitnami/kubectl:latest`
1. [Flux](https://github.com/fluxcd/flux2/pkgs/container/flux-cli) => `docker pull ghcr.io/fluxcd/flux-cli:v0.36.0`

Both images provide the same capability.  

- [Bitnami](https://hub.docker.com/r/bitnami/kubectl/) image only operates using `kubectl` CLI.  
- [Flux](https://github.com/fluxcd/flux2/pkgs/container/flux-cli) comes with `flux` CLI and `kubectl` CLI.

Both images require `flux` operator to be installed on the cluster. This enables the commands for suspending and resuming a kustomization to be applied via Flux System Controller.
However, neither image requires the `flux` CLI to be installed on the cluster.

## Decision Drivers

Following list drives the decision but it is not an exhaustive list of reasons.  

- How much a developer is accustomed to running these commands?
- How maintainable these commands are?
- Where are the container images hosted and is there any limitations on that?
- Can these images support both commands?
- Is there any particular security benefits of using these images?

A sample command to suspend and resume a Flux Kustomization using `kubectl` CLI via [Bitnami](https://hub.docker.com/r/bitnami/kubectl/) image

```sh
// Suspend a Kustomization
kubectl patch kustomizations.kustomize.toolkit.fluxcd.io {KustomizationName} -n {Namespace} --patch '{"spec":{"suspend":true}}' --type merge;

// Resume a Kustomization
kubectl patch kustomizations.kustomize.toolkit.fluxcd.io {KustomizationName} -n {Namespace} --patch '{"spec":{"suspend":false}}' --type merge;
```

A sample command to suspend and resume a Flux Kustomization using `flux` CLI via [Flux](https://github.com/fluxcd/flux2/pkgs/container/flux-cli)

```sh
// Suspend a Kustomization
flux suspend ks {KustomizationName};

// Resume a Kustomization
flux resume ks {KustomizationName};
```

**How much a k8s developer is accustomed to running these commands?**

- `kubectl` command requires the operator to know exactly what Flux System Controller API needs. Also, `kubectl` command requires writing a correct JSON and a `--merge` flag in the end.
- `flux` command requires only the name of the Kustomization and all underlying requirements to apply it via  Flux System Controller API is abstracted away. This means it may be easier to understand if you're familiar with the `flux` CLI. However, `flux` CLI is not used in the production environment, as all configuration of the `flux` operator is managed by the `az k8s-configuration flux`.

**How maintainable these commands are?**

Both commands are single line and should be easy to update.

`kubectl` command requires to know exactly what Flux System Controller API needs like the flags `--type merge` and `--patch` as well as the payload `'{"spec":{"suspend":true}}'`

```sh
kubectl patch kustomizations.kustomize.toolkit.fluxcd.io {KustomizationName} -n {Namespace} --patch '{"spec":{"suspend":true}}' --type merge;
```

`flux` command can be as simple as looking up `flux --help` in the terminal and can be maintained much easily. Actually `flux` command just requires the name of the kustomization.

```sh
flux suspend ks {KustomizationName};
```

**Where are the container images hosted and is there any limitations on that?**

Both images are hosted on public repositories. Limitations of these public repositories are summarised below:

Docker Hub

- Anonymous and Free Docker Hub users are limited to 100 and 200 container image pull requests per six hours
- To increase your pull rate limits you can upgrade your account to a Docker Pro or Team subscription

Github Container Registry

- User-to-server requests are limited to 5,000 requests per hour and per authenticated user.

**Can these images support both commands?**

These are findings from the POC that can add additional context to the decision. [Flux](https://github.com/fluxcd/flux2/pkgs/container/flux-cli) images comes with `kubectl` CLI already installed. So, even if the k8s developer doesn't know `flux` commands but still prefers `kubectl` commands, they can use it without changing the image.

However, [Bitnami](https://hub.docker.com/r/bitnami/kubectl/) image doesn't support running `flux` commands. Running the following CronJob will fail with the error

```sh
/bin/sh: 1: flux: not found
```

## Conclusion

Using `flux` CLI vs `kubectl` for maintenance window operations can be compared as:

|                          | flux CLI                                 | kubectl CLI                                                                                                                                      |
|--------------------------|:-----------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------|
| Suspend a Kustomization  | `flux suspend ks {KustomizationName};`   | `kubectl patch kustomizations.kustomize.toolkit.fluxcd.io {KustomizationName} -n {Namespace} --patch '{"spec":{"suspend":true}}' --type merge;`  |
| Resume a Kustomization   | `flux resume ks {KustomizationName};`    | `kubectl patch kustomizations.kustomize.toolkit.fluxcd.io {KustomizationName} -n {Namespace} --patch '{"spec":{"suspend":false}}' --type merge;` |
| Required Parameters      | 1 - {KustomizationName}                  | 5 - {API Name} {KustomizationName}, {Namespace}, {path}, {type}                                                                                  |
| Ease of Maintenance      | Easy                                     | Moderate                                                                                                                                         |
| Error Proneness          | Low - (Single parameter)                 | Medium (5 parameters, patch being a correct JSON input)                                                                                          |
| Current usage            | Only in dev, stag environments           | All environments                                                                                                                                 |

Below is the comparison of the options:

|                                       | Flux Image     | Bitnami Image  |
|---------------------------------------|:---------------|:---------------|
| Ease of Maintenance                   | Easy           | Easy           |
| Public Container Registry Limitations | 5,000 per hour | 33 per hour    |
| Supporting `flux` Commands            | Yes            | No             |
| Supporting `kubectl` Commands         | Yes            | Yes            |

To summarise, using `flux` CLI has better ease of maintenance and less error prone compared to `kubectl`. However, it still depends on the developer preference on which command to run.  
Also from above comparisons, [Flux](https://github.com/fluxcd/flux2/pkgs/container/flux-cli) images supports both `flux` and `kubectl` commands.  
This means even a developer can prefer using `kubectl` interchangeably with `flux` using Flux Image.
However, using Bitnami Image will constrain the developer to only use `kubectl` commands.

In addition, Flux image is hosted on a more generous platform that only requires a `GITHUB_TOKEN` to mitigate rate limiting.
However, [Bitnami](https://hub.docker.com/r/bitnami/kubectl/) image not only supports `kubectl` commands and also hosted on a platform that may require extra licensing to mitigate rate limiting.
