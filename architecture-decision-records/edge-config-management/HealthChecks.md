# Health checks

## Problem

When deploying a new version of an application to a K3s cluster, we'd like to know if the application started successfully and is healthy. We can achieve this by configuring [Kubernetes probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

We'd like to add at least a `liveness probes` to all applications and to add `readiness probes` as appropriate.
The reason for that is that readiness probes are only used by the `kubelet` to know when a container is ready to accept traffic and since we're using a `MQTT broker` our traffic isn't being managed by the `kubelet`. Despite this, we can still use readiness probes to highlight problems with our application.

Additionally, there are also `startup probes` that can optionally be used to make the `liveness` and `readiness probes` wait until the app has started.

## Context

A probe requires that you define how you'd like to determine either the result of the probe. For example, by checking if a file has been created in your container. There are a variety of different events you can choose as the target of your probe.

## Decision Drivers

Following list drives the decision but it is not an exhaustive list of reasons.  

- Does this probe event give us the right type of information we need?
- How easy is this to implement?
- Are there any security concerns?
- How commonly is it being used by the community for health checks?

## Options

### Create a file on disk with Container Lifecycle Events

To create a file on disk without having to modify your code, you can [Attach Handlers to Container Lifecycle Events](https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/), specifically the `postStart` event. This would allow you to use this handler to create a 'ready' file on disk when the container is started.

> **NOTE:** Kubernetes sends the postStart event immediately after the Container is created. There is no guarantee, however, that the postStart handler is called before the Container's entrypoint is called. The postStart handler runs asynchronously relative to the Container's code, but Kubernetes' management of the container blocks until the postStart handler completes.

However, this would only tell you the same information that Kubernetes is already gathering about your application, such as that the container started, not if the application ready or alive.

### Reading application logs

To determine if the application has reached a specific state, you can read the log files of your application. The log files of your application are stored in the `/var/log/containers` directory of your host machine, not in the container of your application.

One option to access the logs would be to install kubectl in your pod, create a `serviceAccount` with `kubectl logs` access and use that account with your pod. This if done improperly, or even when done properly can still leave your cluster vulnerable to security risks, due to the container having kubectl access. More info on this discussion can be found [here](https://stackoverflow.com/questions/59846090/use-of-kubectl-log-in-readiness-probe).

Another, not recommended, option would be to run the pod as root and mount the log files as a volume, but that can expose you to pod escape security risks as detailed [here](https://aquasecurity.github.io/kube-hunter/kb/KHV047.html) and [here](https://blog.aquasec.com/kubernetes-security-pod-escape-log-mounts).

### Create a file on disk from application

You can have your application to create a file on disk when it has reached a known good state and then use a probe to verify this file exists.

```yaml
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

Just creating the file will not work as an appropriate `liveness probe` because the file is only created once, if the application fails after creating the file, the probe has no way to know.

You can solve this in a multitude of ways, for example:

- add a timestamp to the file and verify that the timestamp is not older than *X* seconds/minutes within your probe
- delete the file when your application reaches an unhealthy state. This requires you to always be able to tell when your application reaches a unhealthy state and it may not always be possible resulting in the probe incorrectly stating the application is alive.

### Create a health endpoint

Create a health endpoint in your application that you then query with from your probe.

```yaml
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 3
```

Examples of creating health endpoints:

- [Health monitoring | Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/monitor-app-health)

## Conclusion

Below is the comparison of the options:

|                                   | Container Lifecycle Events | Reading application logs | Reading a file | Querying endpoint |
| --------------------------------- | :------------------------- | :----------------------- | :------------- | :---------------- |
| Estimated implementing difficulty | No code changes            | Image, cluster changes   | Code changes   | Code changes      |
| Security risks                    | None                       | Potentially              | None           | None              |
| Is it a useful probe?             | No                         | Only startup probe       | Yes            | Yes               |
| How commonly is it used?          | Very Rare                  | Very Rare                | Uncommon       | Standard          |

